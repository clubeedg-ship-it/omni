# SUPERPROMPT — VM Bootstrap

Read `AGENTS.md` first for invariants. Read `.project/STATE.md` for targets. Read `.project/DECISIONS.md` for settled rules (D-001 through D-009). Then execute top-to-bottom. Do not re-read those files during execution.

Make every decision yourself. Log non-obvious ones in `.project/DECISIONS.md`. Commit after each phase. Update `.project/STATE.md` and `.project/QUEUE.md` as you go. Push at the end with a summary.

---

## Phase 0 — Recon

Audit the VM. Save to `.project/reports/phase-0-vm-audit.md`.

```bash
echo "=== GPU ===" && lspci | grep -i nvidia && nvidia-smi 2>/dev/null
echo "=== CPU ===" && nproc && lscpu | head -12
echo "=== RAM ===" && free -h
echo "=== DISK ===" && df -h / && lsblk
echo "=== OS ===" && cat /etc/os-release | head -6
echo "=== DOCKER ===" && docker --version 2>/dev/null && docker compose version 2>/dev/null
echo "=== NVIDIA CONTAINER ===" && nvidia-ctk --version 2>/dev/null
echo "=== GPU CONTAINER TEST ===" && docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi 2>/dev/null
echo "=== PORTS ===" && ss -tlnp | grep -E '(3100|5432|6379|8000|8080|8081|9000)'
echo "=== NVIDIA DEVICES ===" && ls /dev/nvidia* 2>/dev/null
```

Commit: `phase-0: vm audit`

---

## Phase 1 — Fix gaps

Install whatever Phase 0 identified as missing.

**NVIDIA driver** (if `nvidia-smi` failed):
```bash
apt update && apt install -y nvidia-driver-560
reboot
```

**nvidia-container-toolkit** (if `nvidia-ctk` failed):
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt update && apt install -y nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker && systemctl restart docker
```

**Verify**: `docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi` — MUST pass.

**Dirs**: `mkdir -p /opt/omni/models /opt/omni/data/honcho-pg /opt/omni/data/redis /opt/omni/data/whisper-cache /opt/omni/logs /opt/omni/config /opt/omni/tests`

**Models**:
1. Search HuggingFace for exact Qwen3.6-27B Q4_K_M GGUF filename. Download to `/opt/omni/models/`.
2. Download `nomic-embed-text-v1.5.f16.gguf` (~64MB) from `nomic-ai/nomic-embed-text-v1.5-GGUF` to `/opt/omni/models/`.

Commit: `phase-1: prerequisites installed`

---

## Phase 2 — Docker Compose

Create `/opt/omni/docker-compose.yml`. Build incrementally — verify each service before adding the next.

### 2a: llama-server + embedding-server + Redis

**CRITICAL: llama-server MUST have `--jinja --reasoning-format deepseek` for tool calling to work (D-008).**

```yaml
services:
  llama-server:
    image: ghcr.io/ggerganov/llama.cpp:server-cuda-latest
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: all, capabilities: [gpu]}]
    command: >-
      -m /models/<ACTUAL_FILENAME>.gguf
      -ngl 99 -c 262144 -np 1 -fa on
      --cache-type-k q4_0 --cache-type-v q4_0
      --jinja --reasoning-format deepseek
      --temp 0.6 --top-k 20 --top-p 0.95 --min-p 0
      --host 0.0.0.0 --port 8080
    volumes: ["/opt/omni/models:/models:ro"]
    ports: ["8080:8080"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      start_period: 300s
    restart: unless-stopped

  embedding-server:
    image: ghcr.io/ggerganov/llama.cpp:server-latest
    command: >-
      -m /models/nomic-embed-text-v1.5.f16.gguf
      --embeddings -c 8192 -ngl 0
      --rope-scaling yarn --rope-freq-scale 0.75
      --host 0.0.0.0 --port 8081
    volumes: ["/opt/omni/models:/models:ro"]
    ports: ["8081:8081"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 30s
      timeout: 5s
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes: ["/opt/omni/data/redis:/data"]
    ports: ["6379:6379"]
    restart: unless-stopped
```

After starting, run:
```bash
# Get the exact model ID llama-server reports
curl -s http://localhost:8080/v1/models | jq '.data[0].id'
# Use this ID for all Honcho MODEL env vars
```

Test inference:
```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Say OK"}],"max_tokens":5}'
```

Test embedding:
```bash
curl -s http://localhost:8081/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input":"search_query: test embedding","model":"nomic-embed-text-v1.5"}'
```

If OOM: reduce `-c 262144` to `-c 131072`. Log as decision. Record `nvidia-smi` VRAM.

### 2b: faster-whisper

```yaml
  whisper:
    image: fedirz/faster-whisper-server:latest-cuda
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: all, capabilities: [gpu]}]
    environment:
      WHISPER__MODEL: large-v3
      WHISPER__DEVICE: cuda
    volumes: ["/opt/omni/data/whisper-cache:/root/.cache"]
    ports: ["9000:8000"]
    restart: unless-stopped
```

### 2c: Honcho

**Honcho local LLM config is fully resolved (D-003, D-009).** Every feature uses this pattern:
```
<FEATURE>_MODEL_CONFIG__TRANSPORT=openai
<FEATURE>_MODEL_CONFIG__MODEL=<model-id-from-/v1/models>
<FEATURE>_MODEL_CONFIG__OVERRIDES__BASE_URL=http://llama-server:8080/v1
```

For embeddings, point at the CPU embedding server instead:
```
EMBEDDING_MODEL_CONFIG__OVERRIDES__BASE_URL=http://embedding-server:8081/v1
EMBEDDING_MODEL_CONFIG__MODEL=nomic-embed-text-v1.5
```

```yaml
  honcho-db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: honcho
      POSTGRES_USER: honcho
      POSTGRES_PASSWORD: ${HONCHO_DB_PASSWORD}
    volumes: ["/opt/omni/data/honcho-pg:/var/lib/postgresql/data"]
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U honcho"]
      interval: 10s
    restart: unless-stopped

  honcho:
    build: ./honcho
    depends_on:
      honcho-db: {condition: service_healthy}
      llama-server: {condition: service_healthy}
      embedding-server: {condition: service_healthy}
    environment:
      DB_CONNECTION_URI: postgresql+psycopg://honcho:${HONCHO_DB_PASSWORD}@honcho-db:5432/honcho
      AUTH_USE_AUTH: "false"
      SENTRY_ENABLED: "false"
      LLM_OPENAI_API_KEY: "sk-dummy-local"
      # Deriver
      DERIVER_MODEL_CONFIG__TRANSPORT: openai
      DERIVER_MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      # Summary
      SUMMARY_MODEL_CONFIG__TRANSPORT: openai
      SUMMARY_MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      SUMMARY_MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      # Dream
      DREAM_DEDUCTION_MODEL_CONFIG__TRANSPORT: openai
      DREAM_DEDUCTION_MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DREAM_DEDUCTION_MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      DREAM_INDUCTION_MODEL_CONFIG__TRANSPORT: openai
      DREAM_INDUCTION_MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DREAM_INDUCTION_MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      # Dialectic (all 5 levels)
      DIALECTIC_LEVELS__minimal__MODEL_CONFIG__TRANSPORT: openai
      DIALECTIC_LEVELS__minimal__MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DIALECTIC_LEVELS__minimal__MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      DIALECTIC_LEVELS__low__MODEL_CONFIG__TRANSPORT: openai
      DIALECTIC_LEVELS__low__MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DIALECTIC_LEVELS__low__MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      DIALECTIC_LEVELS__medium__MODEL_CONFIG__TRANSPORT: openai
      DIALECTIC_LEVELS__medium__MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DIALECTIC_LEVELS__medium__MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      DIALECTIC_LEVELS__high__MODEL_CONFIG__TRANSPORT: openai
      DIALECTIC_LEVELS__high__MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DIALECTIC_LEVELS__high__MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      DIALECTIC_LEVELS__max__MODEL_CONFIG__TRANSPORT: openai
      DIALECTIC_LEVELS__max__MODEL_CONFIG__MODEL: "${LLAMA_MODEL_ID}"
      DIALECTIC_LEVELS__max__MODEL_CONFIG__OVERRIDES__BASE_URL: http://llama-server:8080/v1
      # Embeddings (CPU server on :8081)
      EMBEDDING_MODEL_CONFIG__TRANSPORT: openai
      EMBEDDING_MODEL_CONFIG__MODEL: nomic-embed-text-v1.5
      EMBEDDING_MODEL_CONFIG__OVERRIDES__BASE_URL: http://embedding-server:8081/v1
      VECTOR_STORE_TYPE: pgvector
    ports: ["8000:8000"]
    restart: unless-stopped
```

### 2d: Paperclip

Check if Docker image exists. If not, build from source:
```dockerfile
FROM node:20-slim
RUN npm install -g pnpm@9
WORKDIR /app
RUN git clone --depth 1 https://github.com/paperclipai/paperclip.git . && pnpm install
EXPOSE 3100
CMD ["pnpm", "dev"]
```

Create `/opt/omni/.env`:
```env
HONCHO_DB_PASSWORD=<openssl rand -hex 16>
LLAMA_MODEL_ID=<value from curl localhost:8080/v1/models>
COMPOSE_PROJECT_NAME=omni
```

Commit: `phase-2: docker-compose stack`

---

## Phase 3 — Tests

Create `/opt/omni/tests/validate.sh`:

```bash
#!/bin/bash
PASS=0; FAIL=0
check() { if eval "$2" >/dev/null 2>&1; then echo "PASS $1"; ((PASS++)); else echo "FAIL $1"; ((FAIL++)); fi }

check "GPU in container" "docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi"
check "llama-server health" "curl -sf http://localhost:8080/health"
check "llama-server inference" "curl -sf http://localhost:8080/v1/chat/completions -H 'Content-Type: application/json' -d '{\"messages\":[{\"role\":\"user\",\"content\":\"Say OK\"}],\"max_tokens\":5}'"
check "embedding-server health" "curl -sf http://localhost:8081/health"
check "embedding-server embed" "curl -sf http://localhost:8081/v1/embeddings -H 'Content-Type: application/json' -d '{\"input\":\"test\",\"model\":\"nomic-embed-text-v1.5\"}'"
check "whisper health" "curl -sf http://localhost:9000/health"
check "honcho api" "curl -sf http://localhost:8000/"
check "paperclip api" "curl -sf http://localhost:3100/"
check "redis ping" "docker compose -f /opt/omni/docker-compose.yml exec redis redis-cli ping | grep -q PONG"

echo "---"
echo "$PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

Save results to `.project/reports/phase-3-test-results.md`. Commit: `phase-3: test suite`

---

## Phase 4 — Persistence

```bash
cat > /etc/systemd/system/omni.service << 'EOF'
[Unit]
Description=Omni AI Stack
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/omni
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=600

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable omni.service
```

Create `/opt/omni/scripts/health.sh`:
```bash
#!/bin/bash
echo "=== Services ==="
docker compose -f /opt/omni/docker-compose.yml ps --format "table {{.Name}}\t{{.Status}}"
echo "=== GPU ==="
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv,noheader
echo "=== Ports ==="
for p in 8080 8081 9000 8000 3100 5432 6379; do
  (echo >/dev/tcp/localhost/$p) 2>/dev/null && echo "$p: open" || echo "$p: closed"
done
```

Commit: `phase-4: systemd + monitoring`

---

## Phase 5 — Final report

Update `.project/STATE.md` with real values.
Update `.project/QUEUE.md` statuses.

Write `.project/reports/phase-5-summary.md`:
- What's running (services, ports, VRAM)
- What works (test results)
- What needs human attention
- Decisions made during bootstrap
- Next steps for Agent lane (Hermes install, Paperclip HTTP skills, gateway config)

Commit and push: `phase-5: bootstrap complete`

---

## If you get stuck

- OOM: reduce context to 131072, log decision
- Honcho won't start: skip deriver, get API up, log known issue
- Paperclip no Docker image: build from source
- Model filename changed: use whatever Q4_K_M GGUF exists for Qwen3.6 27B
- Port conflict: change external port, log decision
- Tool calling broken: verify --jinja --reasoning-format deepseek flags are present
- Embeddings fail: verify embedding-server is running on 8081 with --embeddings flag
- True blocker: stop, push what you have, report
