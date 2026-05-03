# AGENTS.md

## Reading order
1. This file (done)
2. `.project/STATE.md` — where we are
3. `.project/QUEUE.md` — what to do next
4. `.project/DECISIONS.md` — settled rules

## Invariants
1. **Local-first.** No cloud LLM APIs. All inference on the local RTX 4090 via llama-server.
2. **Single GPU.** 24GB VRAM. Document impact of every GPU service.
3. **Compose-first.** All services in `docker-compose.yml`. No ad-hoc `docker run`.
4. **Test before ship.** Write tests before the service they validate.
5. **Report before build.** Start each phase with a state capture.
6. **No secrets in repo.** `.env` never committed.

## Autonomy protocol
- Run all phases end-to-end without human check-ins.
- When you hit a decision point, make the call, log it in `.project/DECISIONS.md`, keep going.
- If something is truly blocked (hardware missing, SSH unreachable), stop and report. Everything else — you decide.
- Commit after each completed phase: `phase-N: <summary>`.
- Final deliverable: update `.project/STATE.md` with results, push, then summarize what you did and what needs human attention.

## Port map
| Port | Service |
|------|---------|
| 8080 | llama-server (GPU, inference + tool calling) |
| 8081 | embedding-server (CPU, nomic-embed-text-v1.5) |
| 9000 | faster-whisper |
| 8000 | Honcho API |
| 5432 | Honcho Postgres |
| 3100 | Paperclip |
| 6379 | Redis |

## Context management
- Do NOT re-read files you've already read in this session.
- `.project/STATE.md` is the only file you update during execution.
- `.project/DECISIONS.md` is append-only.
- Reports go in `.project/reports/` — one file per phase, named `phase-N-<subject>.md`.
