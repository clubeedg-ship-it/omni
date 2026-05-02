# STATE

project: omni (Zero-Human Company — Second Brain)
branch: main
phase: M0-SCAFFOLD
active: INFRA.P0 (genesis — this commit)

## Runtime (to be filled by agent during Phase 0)
```
vm_hostname: TBD
vm_ip: TBD
tailscale: TBD
gpu: RTX 4090 24GB (passthrough) — driver version TBD
os: TBD
docker: TBD
nvidia_container_toolkit: TBD
free_disk: TBD
free_ram: TBD
```

## Stack targets
- llama-server :8080 — Qwen3.6-27B-Q4_K_M.gguf, 262k ctx, q4_0 KV cache, ~20-22GB VRAM
- faster-whisper :9000 — large-v3, CUDA, bursty GPU sharing
- Honcho :8000 — memory library, deriver pointed at local llama-server (NOT cloud)
- Honcho-PG :5432 — pgvector/pgvector:pg16
- Paperclip :3100 — company orchestration, embedded Postgres (separate from Honcho)
- Redis :6379 — job queue

## Truths
- Hermes WhatsApp gateway replaces custom Puppeteer (D-001)
- Hermes Email gateway replaces custom IMAP listener (D-002)
- Honcho deriver must use local LLM, not cloud APIs (D-003)
- Paperclip keeps its own Postgres — do not share with Honcho (D-004)
- GPU time-sharing between llama-server and whisper; fallback: ctx 131072 (D-005)

## After bootstrap, the next layer
- Hermes Agent connects to llama-server :8080 as LLM backend
- Hermes uses Honcho :8000 for persistent memory
- Hermes creates tickets in Paperclip :3100 via MCP bridge tool
- Audio from PWA hits faster-whisper :9000 for transcription
- WhatsApp + Email flow through Hermes native gateway
