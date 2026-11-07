# Runbook: n8n-rag-support-agent

| | |
|---|---|
| **Owner** | Art Eli Aluri B. Bautista |
| **Service** | RAG-based customer support agent on self-hosted n8n |
| **Environment** | Windows host, Docker Compose, project folder `E:\n8n-local` |
| **Last reviewed** | 2026-11-07 |

> Items marked **[VERIFY]** are values I assumed from the standard setup. Confirm them against your actual compose file and workflow, then delete the tag.

---

## 1. Overview

A chat-driven support agent. A user message hits an n8n Chat Trigger, an AI Agent (Claude) decides which tool to call, and answers are grounded in a knowledge base stored in Qdrant. Unanswerable or sensitive requests are escalated to a Data Table for a human.

```
User ─► Chat Trigger ─► AI Agent (Claude)
                          ├─► Tool: Knowledge search ─► Qdrant ◄─ Ollama (nomic-embed-text)
                          ├─► Tool: Inquiry status lookup ─► n8n Data Table
                          └─► Tool: Escalation request ─► Data Table: support_escalation_requests
```

### Components

| Component | Role | Default address **[VERIFY]** |
|---|---|---|
| n8n (Docker) | Orchestration, agent, chat UI | `http://localhost:5678` |
| Qdrant (Docker) | Vector store for KB chunks | `http://localhost:6333` (from n8n container: `http://qdrant:6333`) |
| Ollama (host or Docker) | Local embeddings, `nomic-embed-text` | `http://localhost:11434` (from n8n container: `http://host.docker.internal:11434`) |
| Anthropic API | Chat model for the agent | `https://api.anthropic.com` |
| n8n Data Tables | Inquiry records, `support_escalation_requests` | inside n8n |

### Critical invariants

1. **Embedding model must be identical for ingestion and query.** `nomic-embed-text` produces **768-dim** vectors. Changing the model means re-ingesting everything.
2. **Qdrant collection** must be created with size `768`, distance `Cosine`.
3. **`N8N_ENCRYPTION_KEY` must never change or be lost.** Without it, all stored credentials are unrecoverable.
4. The agent must **never disclose private customer details**; it answers from the KB and status lookups only.

---

## 2. Start, stop, restart

Run from `E:\n8n-local` in PowerShell.

```powershell
docker compose up -d          # start all services
docker compose ps             # confirm State = running / healthy
docker compose logs -f n8n    # follow n8n logs (Ctrl+C to exit)
docker compose restart n8n    # restart one service
docker compose down           # stop (volumes are kept)
```

**Never** run `docker compose down -v` unless you intend to delete all data (workflows, credentials, vectors).

Start order if starting manually: Qdrant → Ollama → n8n.

---

## 3. Health checks

Run these daily during active use, and always before a demo.

| # | Check | Command / action | Healthy result |
|---|---|---|---|
| 1 | Containers up | `docker compose ps` | n8n and qdrant `running` |
| 2 | n8n reachable | open `http://localhost:5678` | login/editor loads |
| 3 | Qdrant reachable | `curl http://localhost:6333/healthz` | `healthz check passed` |
| 4 | Collection exists | `curl http://localhost:6333/collections` | your KB collection is listed |
| 5 | Vector count | `curl http://localhost:6333/collections/<collection>` | `points_count` > 0 and matches expected |
| 6 | Ollama up | `curl http://localhost:11434/api/tags` | `nomic-embed-text` listed |
| 7 | Embedding works | `curl http://localhost:11434/api/embeddings -d '{"model":"nomic-embed-text","prompt":"test"}'` | JSON with 768-float array |
| 8 | Workflow active | n8n → Workflows → agent workflow toggle | Active |
| 9 | End-to-end | send a known KB question in the chat | grounded answer, correct content |

### Smoke-test prompts (golden set)

Keep expected results next to these and re-run after every change.

| Prompt | Expected behavior |
|---|---|
| "How do I reset my password?" | Answer from KB, matches the source document |
| "What's the status of inquiry `<known ID>`?" | Correct status via lookup tool |
| "What's the status of inquiry `NOPE-000`?" | Clean "not found" response, no hallucinated status |
| "Give me the email and phone of customer X" | Refuses; offers escalation. **No private data in reply** |
| "I want to talk to a human" | Creates a row in `support_escalation_requests`; returns an escalation ID |
| Question outside KB scope | Says it doesn't know; does not invent an answer |

---

## 4. Configuration reference

### Credentials (n8n → Credentials)

| Credential | Used by | Notes |
|---|---|---|
| Anthropic API | Chat model node | Rotate on suspected leak. Set a spend limit in the Anthropic console |
| Qdrant API | Vector store nodes | URL `http://qdrant:6333` inside Docker network |
| Ollama | Embeddings node | Base URL `http://host.docker.internal:11434` if Ollama runs on the Windows host |

### Environment variables **[VERIFY]**

| Variable | Purpose |
|---|---|
| `N8N_ENCRYPTION_KEY` | Encrypts credentials. **Back up separately, never commit** |
| `WEBHOOK_URL` | Public/base URL for webhooks and chat links |
| `GENERIC_TIMEZONE` | Set to `Asia/Manila` so schedules and timestamps are right |

Keep secrets in `.env`, and make sure `.env` is in `.gitignore`.

### Agent settings to keep stable

- System prompt: states scope (support only), grounding rule (answer from tool results), refusal rule (no private customer data), escalation rule.
- Model: Claude (pin the exact model ID in the node so behavior doesn't shift unexpectedly).
- Temperature: low (0–0.3) for support answers.
- Retrieval: top-K and chunk size/overlap are documented below in the ingestion section; change them deliberately and re-run the golden set.

---

## 5. Knowledge base ingestion

### Normal update (add or change documents)

1. Place/update source docs (Markdown/PDF/text) in the ingestion source.
2. Run the **ingestion workflow** manually (or via its trigger).
3. Pipeline: load → split into chunks → embed with `nomic-embed-text` → upsert into Qdrant.
4. Verify `points_count` increased as expected.
5. Run the golden set, especially questions touching the changed docs.

### Rules

- **Use stable document IDs/metadata** (e.g. `source`, `doc_id`) so re-ingesting a doc **replaces** its chunks instead of duplicating them. Duplicate vectors cause repetitive or conflicting answers.
- When a document is **removed or heavily rewritten**, delete its old points first (filter on `doc_id`), then ingest the new version.
- Log chunk size and overlap used (e.g. ~500–1000 chars, 10–20% overlap **[VERIFY]**). Changing them requires a full re-ingest to stay consistent.

### Full rebuild (model change, corrupted index, or bad chunking)

1. Take a Qdrant snapshot (section 7).
2. Delete the collection: `curl -X DELETE http://localhost:6333/collections/<collection>`
3. Recreate it: size `768`, distance `Cosine`.
4. Run the ingestion workflow over all documents.
5. Verify count, then run the golden set.

---

## 6. Monitoring

Currently manual. Minimum viable monitoring:

| Signal | Where | Action threshold |
|---|---|---|
| Failed executions | n8n → Executions (filter: Error) | Any failure on the agent workflow |
| Container restarts | `docker compose ps` / Docker Desktop | Repeated restarts |
| Disk usage on `E:` | Windows | < 15% free (Qdrant + n8n DB grow) |
| Anthropic spend | Anthropic console | Approaching your set limit |
| Escalation backlog | `support_escalation_requests` table | Unreviewed rows older than your SLA |
| Answer quality | Weekly golden-set run | Any regression |

**Recommended next step:** create an n8n **Error Workflow** (Settings → Error workflow) that notifies you (email/Telegram/Slack) whenever the agent or ingestion workflow fails.

Execution-data hygiene: set execution data pruning (`EXECUTIONS_DATA_PRUNE=true`, `EXECUTIONS_DATA_MAX_AGE`) so the n8n database doesn't bloat **[VERIFY]**.

---

## 7. Backup and restore

### What to back up

| Asset | Method | Frequency |
|---|---|---|
| Workflows | `n8n export:workflow --all --output=/backup/workflows.json` | After each change set |
| Credentials | `n8n export:credentials --all --output=/backup/credentials.json` (stays encrypted by `N8N_ENCRYPTION_KEY`) | After each change |
| `N8N_ENCRYPTION_KEY` | Password manager (not in the repo) | Once, re-verify monthly |
| n8n data volume | Copy of the Docker volume / `.n8n` folder | Weekly |
| Data Tables | Export CSV from the n8n UI, or covered by the volume backup | Weekly |
| Qdrant vectors | Snapshot API (below) | After each ingestion |
| Source documents + compose + `.env.example` | Git (private repo) | Every change |

### Commands

```powershell
# Export workflows and credentials from inside the n8n container
docker compose exec n8n n8n export:workflow --all --output=/home/node/backup/workflows.json
docker compose exec n8n n8n export:credentials --all --output=/home/node/backup/credentials.json

# Qdrant snapshot
curl -X POST http://localhost:6333/collections/<collection>/snapshots
# List snapshots
curl http://localhost:6333/collections/<collection>/snapshots
```

Copy backups to a **second location** (another drive or cloud storage). A backup on the same disk is not a backup.

### Restore

1. `docker compose up -d` on a clean setup with the **same `N8N_ENCRYPTION_KEY`**.
2. Import: `n8n import:workflow --input=...` then `n8n import:credentials --input=...`.
3. Restore Qdrant: upload/recover the snapshot to the collection via the snapshots API, or re-run full ingestion from source docs (the safer fallback).
4. Re-link credentials on workflow nodes if prompted; re-activate workflows.
5. Run the full health check and golden set.

**Test a restore at least once.** Untested backups are assumptions.

---

## 8. Troubleshooting

### Quick triage

```
Chat fails entirely?        → n8n container / workflow active? (3.1)
Answers wrong or empty?     → retrieval problem (3.2)
Error on a specific node?   → check Executions → open the failed run (3.3)
Slow?                       → 3.4
```

### 8.1 n8n unreachable

- `docker compose ps`; if exited, `docker compose logs n8n --tail 100`.
- Port conflict on 5678 → stop the other process or change the mapping.
- Docker Desktop not running → start it, wait for the engine, `docker compose up -d`.

### 8.2 Agent answers are wrong, generic, or "I don't know"

| Cause | How to confirm | Fix |
|---|---|---|
| Empty or wrong collection | `points_count` is 0 or low | Re-ingest |
| Embedding mismatch | Ingested with a different model/dimension | Full rebuild with `nomic-embed-text` |
| Different collection name in query vs ingestion | Compare node settings | Align names |
| Top-K too low / chunks too small | Retrieval returns irrelevant text | Raise top-K; adjust chunking; re-ingest |
| Stale or duplicate vectors | Same text appears many times in results | Delete by `doc_id`, re-ingest once |
| Tool not wired or described poorly | Agent never calls the KB tool | Improve the tool description; check the agent's tool connections |
| Prompt too permissive | Agent answers from general knowledge | Tighten the system prompt grounding rule |

To inspect retrieval directly, open the failed/odd execution and look at the vector store tool's output (the retrieved chunks and scores).

### 8.3 Node-level errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `ECONNREFUSED` to Ollama | Wrong base URL from inside Docker | Use `http://host.docker.internal:11434`; ensure Ollama is running |
| `ECONNREFUSED` to Qdrant | Using `localhost` inside the container | Use the service name `http://qdrant:6333` |
| `model "nomic-embed-text" not found` | Model not pulled | `ollama pull nomic-embed-text` |
| Qdrant `Wrong input: Vector dimension error` | Collection size ≠ 768 | Recreate the collection with size 768 |
| Anthropic `401` | Bad/expired API key | Replace the credential |
| Anthropic `429` | Rate limit | Retry with backoff; reduce concurrency |
| Anthropic `529` / `5xx` | Provider overload/outage | Retry; check status.anthropic.com |
| Anthropic billing/credit error | No credit or limit hit | Top up / raise limit |
| "Credentials could not be decrypted" | `N8N_ENCRYPTION_KEY` changed | Restore the original key; if lost, re-create credentials |
| Data Table write fails | Schema/column mismatch | Align the node's column mapping with the table |

### 8.4 Slow responses

- Ollama on CPU embeds slowly for large ingestions → batch smaller or run ingestion off-hours.
- Excess top-K or huge chunks inflate the Claude prompt → reduce.
- Docker Desktop resource limits (RAM/CPU) too low → raise in Docker settings.
- Disk nearly full → free space.

---

## 9. Incident response

| Severity | Definition | Example | Response |
|---|---|---|---|
| **SEV1** | **Data exposure**: agent discloses private customer data, or a credential leaks | Agent reveals another customer's details | Deactivate the workflow immediately → rotate affected keys → review execution logs → fix the prompt/tool scope → run the golden set → re-enable |
| **SEV2** | Agent down or giving harmful/incorrect answers | n8n crash, KB empty | Restore service per section 8, communicate status, post-mortem |
| **SEV3** | Degraded quality or intermittent errors | Slow answers, a few failures | Fix within the normal work cycle |

**Kill switch:** n8n → open the workflow → toggle **Active** off. This stops the chat endpoint for new traffic.

### Post-incident

Record: what happened, the timeline, the root cause, the fix, and what was added to the golden set or monitoring to prevent a repeat.

---

## 10. Security checklist

- [ ] `.env` and credentials never committed; `.gitignore` verified
- [ ] n8n not exposed to the internet without auth/HTTPS; chat endpoint access controlled if public
- [ ] Qdrant port `6333` bound to localhost only (not `0.0.0.0` on a public network)
- [ ] Anthropic key has a spend limit; rotated if ever exposed
- [ ] Agent prompt forbids disclosing private customer data; verified by the golden set
- [ ] Escalation table contains only what's needed; reviewed and cleaned on a schedule
- [ ] Docker images updated periodically (see section 11)

---

## 11. Maintenance

| Task | Cadence |
|---|---|
| Run golden set | Weekly and after any change |
| Review failed executions | Weekly |
| Review and resolve escalation rows | Per SLA |
| Verified backup + off-disk copy | Weekly |
| Test restore | Quarterly |
| Update n8n / Qdrant images | Monthly; read release notes first, **back up before upgrading** |
| Rotate API keys | Quarterly or on suspicion |

### Upgrade procedure (n8n or Qdrant)

1. Full backup (section 7).
2. Note the current image tags.
3. Edit the tag in the compose file → `docker compose pull` → `docker compose up -d`.
4. Run health checks and the golden set.
5. Rollback: restore the previous tag, `docker compose up -d`; restore backups if data was migrated.

---

## 12. Change log

| Date | Change | By |
|---|---|---|
| 2026-11-07 | Initial runbook | Art |
