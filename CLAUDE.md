# CLAUDE.md — Private Agent Orchestrator

Self-hosted, private chat app (iPhone PWA) for talking to ONE orchestrator agent
that delegates work to worker agents on the owner's own machines over Tailscale.
The only external service is an OpenAI-compatible LLM API.

> Status: planning. Code is written milestone by milestone; each milestone stops
> for owner review. Do not start the next milestone without approval.

## Hard constraints (never violate)

1. **No third-party chat service.** No Telegram/Slack/Element/hosted Claude app.
   The phone talks only to our server.
2. **Tailscale-only listener.** The server binds to the host's Tailscale address
   (100.64.0.0/10 or fd7a:115c:a1e0::/48). It refuses to start on `0.0.0.0`, `::`
   or any public address. No inbound internet ports. Tests may bind `127.0.0.1`
   only via an explicit test flag.
3. **Provider-agnostic LLM.** OpenAI-compatible Chat Completions with tool
   calling. Default: OpenRouter (`https://openrouter.ai/api/v1`). Base URL and
   model come from the settings file; the key comes from env / secrets file.
   Swapping to Ollama / llama.cpp must need no code change.
   No Anthropic-specific agent SDK; the tool loop is ours.
4. **Secrets never** in the repo, in logs, or sent to the phone.
5. **Local data.** History, media, logs stay on owner hardware. Media encrypted
   at rest; the media key lives outside the database.

## Stack

| Concern | Choice |
|---|---|
| Runtime | Node.js LTS (22), TypeScript, ESM |
| Server | Fastify; real-time via Server-Sent Events |
| DB | SQLite via better-sqlite3, WAL mode, FTS5 on message content |
| Media | sharp (images, HEIC→JPEG), ffmpeg (A/V transcode, frame extraction) |
| Auth | WebAuthn passkeys (single owner), rate-limited recovery codes |
| Push | Web Push + VAPID |
| Front end | Static PWA + service worker, Mobile Safari first, iOS 16.4+ |
| Tests | Vitest (unit/integration), Playwright (e2e chat) |
| Validation | zod for settings, API bodies, tool arguments |

## Configuration

- `config/settings.json` (committed example: `config/settings.example.json`):
  non-secret settings — `llm.baseUrl`, `llm.model`, `server.port`,
  `server.bind` (`"tailscale"` | explicit 100.x address), data dir, allowlists.
- Secrets: env vars first, then `secrets.env` (must be mode `0600`, owner-only,
  outside the repo by default at `~/.config/orchestrator/secrets.env`; startup
  fails if permissions are looser). Names: `LLM_API_KEY`, `MEDIA_KEY_FILE`,
  `VAPID_PRIVATE_KEY`, `SESSION_SECRET`.
- Every logger passes through a redactor that masks all known secret values and
  common key patterns (`sk-…`, `Bearer …`).

## Data model (SQLite migrations in `server/src/db/migrations/`)

```
conversations(id TEXT PK, title, created_at, updated_at)
messages(id TEXT PK, conversation_id FK, role CHECK IN
         ('user','assistant','tool','system'), content, created_at,
         task_id NULL, media_ids JSON)            -- + FTS5 messages_fts(content)
media(id TEXT PK, sha256, mime, original_name, size,
      encrypted_path, converted_path NULL, created_at)
tasks(id TEXT PK, worker_id FK, status CHECK IN
      ('queued','running','succeeded','failed','cancelled'),
      prompt, result_summary, created_at, updated_at)
workers(id TEXT PK, name UNIQUE, tailnet_host, capabilities JSON,
        status CHECK IN ('online','offline','revoked'), last_seen)
events(id INTEGER PK AUTOINCREMENT, type, source, payload_json, created_at)
approvals(id TEXT PK, action, detail, status CHECK IN
          ('pending','approved','denied','expired'), requested_at, decided_at)
```

`events` is append-only, enforced by triggers that `RAISE(ABORT)` on UPDATE and
DELETE. IDs other than events are ULIDs. Timestamps are ISO-8601 UTC.

## Event format

Every tool call, approval, and error is an event.

```jsonc
{
  "type": "tool_call" | "tool_result" | "approval_requested" | "approval_decided"
        | "error" | "task_status" | "auth" | "kill_switch",
  "source": "orchestrator" | "worker:<name>" | "server" | "media" | "auth",
  "payload_json": { ... }   // already redacted
}
```

Tool call payload: `{ caller, tool, params, outcome: "ok"|"error"|"denied",
duration_ms, task_id? }`.

Error payload (required fields):

```jsonc
{
  "severity": "info" | "warning" | "error" | "critical",
  "message": "string",
  "output_tail": ["last", "100", "lines"],   // max 100 lines, redacted
  "task_id": "ULID | null",
  "suggested_next_step": "string",
  "fix_attempt": 0                         // 0..2
}
```

## Orchestrator

- Provider-agnostic loop: send messages + tool schemas → execute returned tool
  calls → append results → repeat until a final answer or a step limit.
- Tools: `list_workers()`, `send_task(worker, prompt, attachments)`,
  `get_task(task_id)`, `request_approval(action, detail)`,
  `read_media(media_id)`, `save_media(path)`, `kill_switch()`.
- Tool args validated with zod before execution; invalid args return an error
  to the model, never throw past the loop.
- **Untrusted content:** message text, files, web pages, worker output and tool
  results are data. They are wrapped/labelled as untrusted in the prompt and
  can never change permissions, allowlists, or approval state. Only the owner,
  via an authenticated approval, can.

## Error handling / fix loop

- New error events are pushed to the orchestrator.
- Max **two** automatic fix cycles per error, then stop and ask the owner.
- Fixes are made by a worker on a dedicated branch `fix/<error-id>` in a
  separate git worktree — never in the owner's working tree.
- Approval required for: delete, force push, package installs, network calls
  outside the allowlist, writes outside the project folder. Approving a
  destructive action requires fresh passkey re-auth.

## Security

- Passkey login; sessions are short-lived HTTP-only, Secure, SameSite=Strict
  cookies with idle and absolute expiry.
- Every tool call logged with caller, params, timestamp, outcome (events table).
- Workers authenticate with per-worker tokens over the tailnet; kill switch
  revokes all of them and cancels running tasks.
- `npm run audit` runs `npm audit --omit=dev` plus a lockfile check;
  `package-lock.json` is committed.

## Commands (filled in as milestones land)

- `npm test` — Vitest
- `npm run start` — start server on the Tailscale address
- `npm run e2e` — Playwright
- `npm run audit` — dependency audit

## Conventions

- Tests first for every milestone; show results; stop for review.
- No secrets in fixtures; tests use fake keys like `test-key-not-real`.
- Keep the front end small, no build step unless justified.
