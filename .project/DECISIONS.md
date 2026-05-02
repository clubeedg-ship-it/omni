# DECISIONS

## D-001: Use Hermes native WhatsApp gateway
Hermes `gateway/platforms/whatsapp.py` (46KB) + `whatsapp_identity.py` (6KB) handles QR pairing, voice memos, cross-platform continuity. Custom Puppeteer spec killed.

## D-002: Use Hermes native Email gateway
Hermes `gateway/platforms/email.py` (27KB) handles IMAP ingestion with HTML stripping. Custom IMAP listener spec killed.

## D-003: Honcho deriver → local llama-server
Configure `DERIVER_MODEL_CONFIG__TRANSPORT` to `http://llama-server:8080/v1`. No cloud APIs.

## D-004: Separate Postgres instances
Paperclip manages its own embedded PG. Honcho gets pgvector on :5432. No sharing.

## D-005: GPU time-sharing
llama-server holds ~20-22GB persistently. Whisper bursts into remaining VRAM. If OOM, reduce context to 131072.
