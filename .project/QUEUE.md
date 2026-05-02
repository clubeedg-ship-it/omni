# QUEUE

| id | title | status | deps |
|---|---|---|---|
| INFRA.P0 | Scaffold repo | done | — |
| INFRA.P1 | VM audit (hardware, GPU, Docker, ports) | todo | P0 |
| INFRA.P2 | Install missing deps (NVIDIA toolkit, Docker GPU) | todo | P1 |
| INFRA.P3 | Download Qwen3.6-27B model | todo | P2 |
| STACK.P1 | Compose: llama-server + Redis | todo | P3 |
| STACK.P2 | Compose: faster-whisper | todo | S1 |
| STACK.P3 | Compose: Honcho (API + PG + deriver → local LLM) | todo | S1 |
| STACK.P4 | Compose: Paperclip | todo | S1 |
| TEST.P1 | Validation suite (all services) | todo | S1-S4 |
| PLAT.P1 | systemd + boot persistence | todo | T1 |
| PLAT.P2 | Health scripts + log rotation | todo | P1 |
