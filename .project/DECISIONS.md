# DECISIONS

## D-001: Use Hermes native WhatsApp gateway
Hermes `gateway/platforms/whatsapp.py` (46KB) + `whatsapp_identity.py` (6KB) handles QR pairing, voice memos, cross-platform continuity. Custom Puppeteer spec killed.

## D-002: Use Hermes native Email gateway
Hermes `gateway/platforms/email.py` (27KB) handles IMAP ingestion with HTML stripping. Custom IMAP listener spec killed.

## D-003: Honcho deriver → local llama-server
Configure `DERIVER_MODEL_CONFIG__TRANSPORT=openai` + `DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL=http://llama-server:8080/v1`. No cloud APIs. Same pattern for all 5 Honcho features (deriver, dialectic levels, summary, dream, embeddings). Set `LLM_OPENAI_API_KEY=sk-dummy-local` (required but not validated by llama-server).

## D-004: Separate Postgres instances
Paperclip manages its own embedded PG. Honcho gets pgvector on :5432. No sharing.

## D-005: GPU time-sharing
llama-server holds ~20-22GB persistently. Whisper bursts into remaining VRAM. If OOM, reduce context to 131072.

## D-006: Hermes-Paperclip bridge is direct HTTP, not MCP
Paperclip exposes REST on :3100. Hermes skills system calls it directly. No protocol layer.

## D-007: Multi-agent inference via queue, not parallel slots
`-np 1` single slot. All agents share one inference queue natively.

## D-008: llama-server requires --jinja --reasoning-format deepseek for tool calling
Source: Qwen official llama.cpp docs. Without these flags, Qwen3 tool calls are not parsed. The corrected command is:
```
-m /models/Qwen3.6-27B-Q4_K_M.gguf -ngl 99 -c 262144 -np 1 -fa on --cache-type-k q4_0 --cache-type-v q4_0 --jinja --reasoning-format deepseek --temp 0.6 --top-k 20 --top-p 0.95 --min-p 0 --host 0.0.0.0 --port 8080
```
Also added Qwen-recommended sampling params (temp 0.6, top-k 20, top-p 0.95, min-p 0).

## D-009: Separate CPU-only embedding server on port 8081
llama-server with a chat GGUF does NOT serve /v1/embeddings. Solution: run a second llama-server instance on port 8081, CPU-only, with `nomic-embed-text-v1.5.f16.gguf` (~64MB, no GPU needed). Honcho points `EMBEDDING_MODEL_CONFIG__OVERRIDES__BASE_URL=http://embedding-server:8081/v1`. The embedding server command:
```
llama-server -m /models/nomic-embed-text-v1.5.f16.gguf --embeddings -c 8192 --rope-scaling yarn --rope-freq-scale 0.75 -ngl 0 --host 0.0.0.0 --port 8081
```
