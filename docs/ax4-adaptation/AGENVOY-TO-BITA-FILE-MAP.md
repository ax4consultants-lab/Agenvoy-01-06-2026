# Agenvoy to Ax4/Bita File Adaptation Map

Repository under review: `ax4consultants-lab/Agenvoy-01-06-2026`  
Target live authority: `ax4consultants-lab/ax4-bita` at `/tools/ax4-bita`  
Current live service: `bita-telegram.service`  
Status: reference-only architecture map; no runtime migration or Go implementation is proposed for immediate deployment.

## 1. Executive Summary

Agenvoy is a Go-based personal agent daemon with Telegram, Discord, TUI, HTTP API, scheduler, tool registry, generated tool adapters, local memory, and RAG concepts. For Ax4/Bita, the safest value is not a full migration. The useful near-term path is to preserve the current Python Telegram bot as the primary live runtime and adapt only narrow, auditable patterns from Agenvoy into Bita.

Primary adaptation recommendation:

- **Keep Bita Python live and primary.** Do not replace `bita-telegram.service`.
- **Use Agenvoy as a reference architecture** for scheduler patterns, tool registry shape, audit logging, session boundaries, human confirmation, and future Go daemon interfaces.
- **Implement the first bridge as a filesystem queue sidecar**, not as direct Telegram control, direct external writes, or autonomous tool execution.
- **Prioritize the first revenue workflow:** `survey inputs -> structured report draft package -> human review`.
- **Defer risky components** such as self-generated tool deployment, external posting, email sending, live scanning, and cloud-model handling of client data until security review and operator approval gates exist.

## 2. Repository Structure Overview

| Path | Purpose in Agenvoy | Ax4/Bita classification | Primary use |
| --- | --- | --- | --- |
| `cmd/app/` | CLI/TUI/daemon entrypoint, lifecycle, update command, agent registry bootstrap | **KEEP AS REFERENCE** / **ADAPT INTO FUTURE GO DAEMON** | Future daemon shape only |
| `configs/` | Static model/provider JSON, prompts, webmode HTML, allow/deny lists | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Prompt and guardrail references |
| `extensions/` | Built-in API specs and skills for generated tools, scheduler skills, upload/install flows | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Tool schema ideas; do not auto-install |
| `internal/agents/` | Provider abstraction, dispatcher, execution loop, memory, subagents, push hooks | **KEEP AS REFERENCE** / **ADAPT INTO FUTURE GO DAEMON** | Future orchestration design |
| `internal/runtime/telegram/` | Telegram daemon runtime, auth, attachments, pending confirmations, push tools | **KEEP AS REFERENCE** / **ADAPT INTO BITA PYTHON NOW** for patterns only | Telegram interface patterns |
| `internal/runtime/discord/` | Discord runtime and push/list tools | **DISCARD** for current Bita / **KEEP AS REFERENCE** | Non-Bita channel reference |
| `internal/runtime/tui/` | Bubble Tea terminal operator interface | **KEEP AS REFERENCE** / **ADAPT INTO FUTURE GO DAEMON** | Operator console ideas |
| `internal/runtime/routes/` | Gin HTTP API for chat completions, send, tools, session logs/pages | **USE FOR GO SIDECAR** / **SECURITY REVIEW REQUIRED** | Future local API bridge |
| `internal/runtime/scheduler.go`, `task.go`, `cron.go` | One-shot and cron persistence/execution model | **ADAPT INTO BITA PYTHON NOW** for JSON queue design / **ADAPT INTO FUTURE GO DAEMON** | Proactive jobs |
| `internal/runtime/kuradb/`, `internal/runtime/torii/` | Local RAG/database runtime integrations | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Future memory/RAG |
| `internal/tools/` | Built-in tools: file, shell, scheduler, search, fetch, git, user data, calculator, agents | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Tool registry and sandbox patterns |
| `internal/toolAdapter/` | API/script/MCP dynamic tool adapters | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Future tool adapter design |
| `internal/filesystem/` | Config paths, runtime paths, logs, skills, records | **ADAPT INTO BITA PYTHON NOW** for directory/log conventions | Local state layout |
| `internal/session/` | Session IDs, history, summaries, config, Telegram/Discord session mapping | **KEEP AS REFERENCE** / **ADAPT INTO BITA PYTHON NOW** for session naming/audit ideas | Session continuity |
| `internal/utils/` | Auth and event utility helpers | **KEEP AS REFERENCE** | Small utility reference |
| `static/`, `index.html`, `configs/webmode.html` | Browser/static UI assets | **DISCARD** for now / **KEEP AS REFERENCE** | Not needed for Bita now |
| `doc/`, `wiki/`, `README.md` | Product docs and architecture explanations | **KEEP AS REFERENCE** | Human architecture source |
| `test/`, `internal/tools/calculator/*_test.go` | Test references | **KEEP AS REFERENCE** | Future validation style |
| `go.mod`, `go.sum`, `makefile`, `LICENSE`, `CNAME` | Project metadata and dependencies | **KEEP AS REFERENCE** | Dependency/security review |

## 3. File-by-File / Directory-by-Directory Mapping

### Root Files

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `README.md` | **KEEP AS REFERENCE** | Product capability map. Useful to compare daemon, multi-channel, generated tools, scheduler, and RAG concepts against Bita's Python runtime. Do not treat feature claims as migration requirements. |
| `LICENSE` | **KEEP AS REFERENCE** | Review license compatibility before copying code or prompt text into Ax4/Bita. |
| `go.mod`, `go.sum` | **SECURITY REVIEW REQUIRED** | Dependency inventory for future Go sidecar/daemon. Not needed for Python Bita now. Review `gin`, Telegram/Discord libraries, scheduler, browser/fetch, ToriiDB/KuraDB dependencies before adoption. |
| `makefile` | **KEEP AS REFERENCE** | Build/developer commands only. Do not integrate into Bita deployment. |
| `index.html`, `CNAME` | **DISCARD** | Static site/web landing material; not relevant to Bita runtime. |

### `cmd/app/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `cmd/app/main.go` | **ADAPT INTO FUTURE GO DAEMON** | Reference for CLI modes, daemon hooks, push hook registration, and background summary cron. Not for Phase 1. Bita should keep Python Telegram as primary. |
| `cmd/app/newDeamon.go`, `cmd/app/cmdDeamon.go`, `cmd/app/daemonSlog.go` | **ADAPT INTO FUTURE GO DAEMON** | Reference for daemon process lifecycle and logging. For Ax4 sidecar, prefer systemd-managed service and filesystem queues instead of self-spawning. |
| `cmd/app/newTUI.go` | **KEEP AS REFERENCE** | TUI bootstrap ideas for future operator console; not required for first revenue workflow. |
| `cmd/app/buildAgentRegistry.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Provider/agent registry bootstrap pattern. Do not import multi-provider routing until data handling rules are defined. |

### `configs/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `configs/configs.go` | **KEEP AS REFERENCE** | Static embedded config pattern. Bita Python can mirror the idea with explicit config modules/files. |
| `configs/jsons/white_list.json` | **SECURITY REVIEW REQUIRED** | Command allowlist reference. Too broad for Ax4 production (`docker`, package managers, shell). Use only to design a stricter Bita allowlist. |
| `configs/jsons/denied_map.json` | **ADAPT INTO BITA PYTHON NOW** | Useful denylist categories for secrets and sensitive paths. Bita should maintain an equivalent denylist before any sandboxed execution. |
| `configs/jsons/exclude_list.json` | **ADAPT INTO BITA PYTHON NOW** | Useful default exclusions for indexing, scanning, and report package assembly. |
| `configs/jsons/providors/*.json` | **SECURITY REVIEW REQUIRED** | Multi-model provider configs. Keep as reference; do not route client data to cloud models by default. |
| `configs/prompts/telegram_format.md`, `telegram_system_prompt.md` | **ADAPT INTO BITA PYTHON NOW** | Useful formatting and channel-specific prompt patterns. Adapt conceptually to Bita's existing Telegram UX, not by wholesale replacement. |
| `configs/prompts/summary_context.md`, `summary_prompt.md`, `default_session_prompt.md` | **KEEP AS REFERENCE** | Helpful for report/session summaries and draft package generation. Review for Ax4 tone and privacy before use. |
| `configs/prompts/single_confirm.md`, `always_allow.md` | **SECURITY REVIEW REQUIRED** | Confirmation-mode patterns. Ax4 should default to explicit approval, not always-allow. |
| `configs/prompts/skill_execution.md` | **SECURITY REVIEW REQUIRED** | Strong instruction pattern for skill execution. Do not enable automatic skill execution in Bita until sandboxing and logging are proven. |
| `configs/prompts/agent_selector.md`, `chatcompletions_system_prompt.md`, `system_prompt.md`, `webmode_system_prompt.md` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Future routing and API-mode prompt references; avoid direct use with client data. |
| `configs/prompts/discord_*` | **DISCARD** for now | Discord is outside current Bita authority. |
| `configs/webmode.html` | **DISCARD** for now | Browser/webmode not part of first phases. |

### `extensions/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `extensions/extensions.go` | **KEEP AS REFERENCE** | Embedded extension loader pattern only. |
| `extensions/apis/*.json` | **USE FOR GO SIDECAR** / **SECURITY REVIEW REQUIRED** | Public API schema examples can inform a future read-only report enrichment sidecar. Disable live calls until approval. |
| `extensions/skills/scheduler-skill-creator/SKILL.md` | **KEEP AS REFERENCE** / **ADAPT INTO FUTURE GO DAEMON** | Useful scheduler-skill packaging pattern. For Bita now, use fixed reviewed job definitions, not generated scheduler skills. |
| `extensions/skills/script-tool-add/SKILL.md`, `api-tool-add/SKILL.md`, `extension-install/SKILL.md`, `extension-upload/SKILL.md` | **SECURITY REVIEW REQUIRED** | High-risk self-extension and publishing flows. Do not migrate yet. |
| `extensions/skills/tool-reviewer/SKILL.md`, `code-reviewer/SKILL.md` | **KEEP AS REFERENCE** | Review checklist patterns can be adapted for human-reviewed draft packages. |
| `extensions/skills/plan/SKILL.md`, `readme-generate/SKILL.md`, `commit-generate/SKILL.md`, `version-generate/SKILL.md` | **KEEP AS REFERENCE** | Internal productivity references only; not runtime requirements. |
| `extensions/skills/search-suitable-public-api/SKILL.md` | **SECURITY REVIEW REQUIRED** | Could support future report enrichment, but live API discovery should not run autonomously. |
| `extensions/skills/skill-creator/*` | **SECURITY REVIEW REQUIRED** | Skill creation is useful later, but not safe for early Bita revenue workflow. |

### `internal/filesystem/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/filesystem/path.go` | **ADAPT INTO BITA PYTHON NOW** | Strong reference for explicit state paths: config, sessions, logs, tools, scheduler, auth files, downloads. For Bita, define an Ax4-owned equivalent under `/tools/ax4-bita` or an approved data dir. |
| `internal/filesystem/runtime.go` | **USE FOR GO SIDECAR** | Runtime PID/UID record pattern. For Phase 1 sidecar, prefer systemd and simple heartbeat/status files. |
| `internal/filesystem/record/log.go`, `usage.go` | **ADAPT INTO BITA PYTHON NOW** | Useful pattern for action/usage logs. Bita sidecar should use JSONL audit logs from the start. |
| `internal/filesystem/reader.go` | **KEEP AS REFERENCE** | File reading helper pattern; implement Python-native equivalents only if needed. |
| `internal/filesystem/skill/*.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Skill persistence and auto-commit patterns. Do not auto-commit generated tools in Bita production. |

### `internal/runtime/telegram/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/runtime/telegram/new.go`, `run.go`, `session.go` | **ADAPT INTO BITA PYTHON NOW** for patterns | Useful for authorized-chat handling, session derivation, inbound message normalization, attachment handling, and status updates. Do not replace the Python bot. |
| `internal/runtime/telegram/admin.go` | **SECURITY REVIEW REQUIRED** | Admin-code/authorization pattern. Adapt only after aligning with Ax4 operator identity policy. |
| `internal/runtime/telegram/pending.go` | **ADAPT INTO BITA PYTHON NOW** | Valuable human approval/pending-interaction pattern for sidecar queue decisions. |
| `internal/runtime/telegram/push.go` | **SECURITY REVIEW REQUIRED** | Outbound push from jobs is useful, but must require approval in early phases. |
| `internal/runtime/telegram/attachments.go`, `fileMarker.go`, `filegroup.go` | **KEEP AS REFERENCE** | Useful later for survey artifacts and uploads. Start with text/JSON survey inputs first. |
| `internal/runtime/telegram/chunk.go` | **ADAPT INTO BITA PYTHON NOW** | Useful Telegram message chunking/formatting concept. |
| `internal/runtime/telegram/tool/list.go`, `send.go`, `format.go`, `register.go` | **SECURITY REVIEW REQUIRED** | Tooling for listing/sending to Telegram. Do not let a sidecar post unsupervised. |

### `internal/runtime/discord/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/runtime/discord/**` | **DISCARD** for current Bita / **KEEP AS REFERENCE** | Discord is outside current live Bita scope. Some pending/push patterns may be reused later. |

### `internal/runtime/tui/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/runtime/tui/tui.go`, `init.go`, `view*.go`, `update.go`, `slog.go` | **ADAPT INTO FUTURE GO DAEMON** | Operator console reference. Not needed for Phase 1 sidecar. |
| `internal/runtime/tui/command*.go`, `handler*.go`, `sessionPublish.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Rich operator controls for model, keys, Telegram, Discord, scheduler, MCP, logs. Useful future UX map, but many commands alter live config and require guardrails. |
| `internal/session/tui/tuiHash.go` | **KEEP AS REFERENCE** | Session ID/hash reference only. |

### `internal/runtime/routes/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/runtime/routes/new.go` | **USE FOR GO SIDECAR** / **SECURITY REVIEW REQUIRED** | Local HTTP API route map is useful for a future sidecar. Phase 1 should avoid HTTP entirely and use filesystem queues. |
| `internal/runtime/routes/handler/chatCompletions/*` | **ADAPT INTO FUTURE GO DAEMON** | OpenAI-compatible local endpoint pattern. Not required while Bita Python remains authority. |
| `internal/runtime/routes/handler/send.go`, `sendResult.go`, `sendSSE.go`, `event.go` | **USE FOR GO SIDECAR** | Useful event/send/result model for later bridge. First bridge should be JSON files and JSONL logs. |
| `internal/runtime/routes/handler/tools.go` | **SECURITY REVIEW REQUIRED** | Tool API exposure is risky. Do not expose live tool execution until auth, sandboxing, and approvals are implemented. |
| `internal/runtime/routes/handler/log.go`, `status.go`, `page.go`, `key.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Status/log visibility is useful. Key handling requires strict review. |

### `internal/runtime/` Shared Runtime

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/runtime/runtime.go` | **USE FOR GO SIDECAR** | PID/UID lifecycle model for future daemon. |
| `internal/runtime/listener.go`, `pending.go` | **ADAPT INTO BITA PYTHON NOW** | Human approval listener/pending registry is directly useful for queue approvals. |
| `internal/runtime/pubsub/*.go` | **USE FOR GO SIDECAR** | Event fan-out pattern for future HTTP/TUI/log subscribers. Not Phase 1. |
| `internal/runtime/scheduler.go`, `task.go`, `cron.go` | **ADAPT INTO BITA PYTHON NOW** / **ADAPT INTO FUTURE GO DAEMON** | Core reference for one-shot tasks and cron persistence. In Python Bita, start with reviewed JSON job definitions and no autonomous external writes. |
| `internal/runtime/skill.go` | **SECURITY REVIEW REQUIRED** | Skill matching/execution bridge. Keep disabled until generated skill safety is proven. |
| `internal/runtime/monitor/monitor.go` | **KEEP AS REFERENCE** | Monitoring concept reference. |
| `internal/runtime/kuradb/**` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | RAG/search reference. Do not index client data into cloud embeddings by default. |
| `internal/runtime/torii/torii.go` | **KEEP AS REFERENCE** | Local store abstraction reference. |

### `internal/agents/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/agents/types/*.go` | **ADAPT INTO FUTURE GO DAEMON** | Useful normalized message/event/agent types. For Python now, define equivalent JSON schema for sidecar queue messages. |
| `internal/agents/host.go` | **KEEP AS REFERENCE** | Agent registry/host pattern. |
| `internal/agents/exec/execute.go`, `run.go`, `toolCall.go`, `selectAgent.go`, `trimMessages.go`, `systemPrompt.go` | **ADAPT INTO FUTURE GO DAEMON** / **SECURITY REVIEW REQUIRED** | Core orchestration and tool-call loop. Too broad for immediate Bita migration. |
| `internal/agents/exec/push.go`, `adminChannel.go` | **SECURITY REVIEW REQUIRED** | Push/admin hooks need strict approval and audit before use. |
| `internal/agents/exec/allow/**` | **ADAPT INTO BITA PYTHON NOW** for policy concept | Allowlist/permission toggles are useful. Implement smaller Python policy controls. |
| `internal/agents/exec/memory/save.go`, `search.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Memory/RAG concept. Client data privacy must be settled before use. |
| `internal/agents/exec/summary/generate.go` | **ADAPT INTO BITA PYTHON NOW** | Good reference for report/session summary generation. Use for draft packages only, with human review. |
| `internal/agents/exec/execWithSubagent.go`, `external.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Subagent/external-agent orchestration is future-only. |
| `internal/agents/external/*.go` | **SECURITY REVIEW REQUIRED** | External agent matching can leak data or trigger unreviewed actions. Do not migrate early. |
| `internal/agents/provider/**` | **SECURITY REVIEW REQUIRED** | Multi-provider LLM integrations, login/refresh flows, image/STT/YouTube tools. Do not adopt without data classification and key handling review. |

### `internal/tools/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/tools/register/register.go`, `tools.go`, `executor.go`, `types/executor.go` | **ADAPT INTO BITA PYTHON NOW** for schema concepts / **ADAPT INTO FUTURE GO DAEMON** | Tool registry pattern is highly useful. In Bita now, implement a static registry of reviewed report-generation tools only. |
| `internal/tools/file/*.go`, `file/denied/denied.go` | **SECURITY REVIEW REQUIRED** | File read/write/search/patch tools are powerful. Use denylist ideas, but do not expose arbitrary filesystem writes to Bita workflows. |
| `internal/tools/runCommand.go`, `runCommandShell.go` | **SECURITY REVIEW REQUIRED** | Sandboxed command execution reference. Do not migrate live shell execution into Bita until all commands are sandboxed, tested, logged, and reviewed. |
| `internal/tools/scheduler/*.go` | **ADAPT INTO BITA PYTHON NOW** / **ADAPT INTO FUTURE GO DAEMON** | Scheduler tool schemas are directly useful. Start with fixed report jobs and reviewed cron/task files. |
| `internal/tools/searcher/*.go` | **KEEP AS REFERENCE** | Tool discovery/listing pattern. Disable self-activation for Bita early phases. |
| `internal/tools/fetchPage/**`, `downloadFile.go`, `external/searchWeb/**`, `external/googleRSS/**`, `external/yahooFinance/**` | **SECURITY REVIEW REQUIRED** | Web fetching and external data can support reports later, but live scans/fetches require approval and domain allowlists. |
| `internal/tools/userData/*.go` | **SECURITY REVIEW REQUIRED** | User email/log/error tools touch sensitive data and outbound email. Do not migrate emailing without approval. |
| `internal/tools/git/*.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Useful for controlled skill versioning, but not relevant to report workflow. |
| `internal/tools/interactive/*.go` | **ADAPT INTO BITA PYTHON NOW** for approval UX | Ask-user/store-secret/install-dependency patterns are useful. Dependency install and secret storage require review. |
| `internal/tools/agent/**` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Subagent, planning, and external review patterns are future-only. |
| `internal/tools/errorMemory/**` | **KEEP AS REFERENCE** | Could inform internal troubleshooting memory, not revenue workflow. |
| `internal/tools/calculator/**` | **KEEP AS REFERENCE** | Simple safe tool example and test pattern. |
| `internal/tools/updatePage.go`, `tuiOnly.go` | **DISCARD** for now | TUI/page-specific functionality not needed. |

### `internal/toolAdapter/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/toolAdapter/api/*.go`, `example.json` | **USE FOR GO SIDECAR** / **SECURITY REVIEW REQUIRED** | API tool adapter schema can power future read-only enrichment tools. Needs auth, rate-limit, domain allowlist, and dry-run mode. |
| `internal/toolAdapter/script/*.go` | **SECURITY REVIEW REQUIRED** | Dynamic script execution is high risk. Do not migrate until sandbox policy exists. |
| `internal/toolAdapter/mcp/*.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | MCP client support is useful later. Do not connect client data to arbitrary MCP servers by default. |

### `internal/session/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/session/session.go`, `concurrent.go`, `clean.go`, `reset.go`, `checkAssign.go` | **ADAPT INTO BITA PYTHON NOW** for concepts | Session mapping, concurrency isolation, and reset/assignment ideas can improve Bita. |
| `internal/session/telegram/telegram.go` | **ADAPT INTO BITA PYTHON NOW** | Hash-based Telegram session mapping is directly useful as a reference. |
| `internal/session/discord/discord.go` | **DISCARD** for now | Not current Bita scope. |
| `internal/session/history/*.go`, `summary/*.go`, `log/*.go`, `toolError/*.go` | **ADAPT INTO BITA PYTHON NOW** | Useful for audit trail, report draft provenance, and user-visible history summaries. |
| `internal/session/config/**` | **KEEP AS REFERENCE** | Session-scoped model/bot/status configuration pattern. Avoid multi-model routing until privacy rules are set. |

### `internal/utils/`

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `internal/utils/auth.go` | **ADAPT INTO BITA PYTHON NOW** | Authorized chat/user file pattern can inform Bita access controls. |
| `internal/utils/event.go`, `chatbotEvent.go`, `utils.go` | **KEEP AS REFERENCE** | Utility/event formatting reference. |

### Assets, Docs, Tests

| Path | Classification | Ax4/Bita adaptation notes |
| --- | --- | --- |
| `static/**`, `doc/*.png`, `doc/*.jpg`, `doc/logo.svg`, `static/*.svg` | **DISCARD** for now | Branding/UI assets are not useful for Bita runtime. |
| `doc/README.zh.md`, `wiki/**` | **KEEP AS REFERENCE** | Useful architecture and configuration notes, especially daemon config, scheduler, pending approvals, and log layout. |
| `test/providers_integration_test.go` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Provider integration test reference; not for Bita now. |
| `internal/tools/calculator/calculate_test.go` | **KEEP AS REFERENCE** | Simple unit test style for future safe tools. |

## 4. Component Usefulness Matrix

| Component need | Agenvoy paths to study | Classification | Bita recommendation |
| --- | --- | --- | --- |
| Telegram interface | `internal/runtime/telegram/**`, `internal/session/telegram/telegram.go`, `configs/prompts/telegram_*` | **ADAPT INTO BITA PYTHON NOW** for patterns | Preserve current Python bot; adapt authorization, session IDs, pending approvals, formatting, and chunking. |
| Scheduler/proactive jobs | `internal/runtime/scheduler.go`, `task.go`, `cron.go`, `internal/tools/scheduler/**`, `extensions/skills/scheduler-skill-creator/SKILL.md` | **ADAPT INTO BITA PYTHON NOW** then **FUTURE GO DAEMON** | Start with reviewed JSON jobs and filesystem queue. Later move to daemonized scheduler. |
| Tool registry | `internal/tools/register/register.go`, `internal/tools/tools.go`, `internal/tools/types/executor.go`, `internal/toolAdapter/**` | **ADAPT INTO BITA PYTHON NOW** for static registry; **SECURITY REVIEW REQUIRED** for dynamic tools | Define reviewed Bita tool schemas for report package generation only. |
| Sandboxed tool execution | `internal/tools/runCommand.go`, `runCommandShell.go`, `configs/jsons/white_list.json`, `denied_map.json`, `internal/tools/file/denied/denied.go` | **SECURITY REVIEW REQUIRED** | Do not enable arbitrary shell or write tools. Use sandboxed, fixed commands only after review. |
| Memory/RAG | `internal/runtime/kuradb/**`, `internal/runtime/torii/torii.go`, `internal/agents/exec/memory/**`, `internal/session/history/**` | **KEEP AS REFERENCE** / **SECURITY REVIEW REQUIRED** | Start with local JSONL provenance and report context. Delay embeddings/vector search for client data. |
| Report automation | `internal/agents/exec/summary/generate.go`, `configs/prompts/summary_*`, `internal/filesystem/record/**`, `internal/session/summary/**` | **ADAPT INTO BITA PYTHON NOW** | First revenue workflow should generate structured draft packages for human review. |
| TUI/desktop operator interface | `internal/runtime/tui/**`, `cmd/app/newTUI.go` | **ADAPT INTO FUTURE GO DAEMON** | Later operator console. Not needed for first bridge. |
| HTTP/API interface | `internal/runtime/routes/**`, `internal/toolAdapter/api/**` | **USE FOR GO SIDECAR** / **SECURITY REVIEW REQUIRED** | Phase 1 should avoid HTTP. Phase 2/3 may add localhost-only API after queue bridge is stable. |

## 5. Staged Migration Path

### Phase 0: Reference Only

- Keep this Agenvoy repository as immutable architecture reference.
- Keep `ax4-bita` Python Telegram bot and `bita-telegram.service` live and primary.
- Extract only design patterns: session IDs, audit logs, pending approvals, scheduler persistence, tool schema descriptions, and report summary prompt structure.
- No Go runtime, no service replacement, no generated tool execution.

### Phase 1: Sidecar Bridge

- Add a Bita-owned filesystem queue beside the Python runtime.
- Python Telegram bot writes approved survey/report requests to an inbound queue.
- Sidecar reads queue items and produces draft artifacts only.
- Sidecar writes JSONL audit logs for every state transition.
- Sidecar performs **no live external writes**, no Telegram sends, no email sends, no scans, and no production deployment.
- Human operator reviews draft package before any client-facing delivery.

### Phase 2: Report Automation Daemon

- Promote sidecar into a report automation daemon only after Phase 1 audit logs prove reliability.
- Add fixed, reviewed report templates and deterministic package layout.
- Add scheduled report-draft generation for approved survey workflows.
- Continue routing final delivery through human review and the existing Bita Telegram interface.

### Phase 3: Shared Memory/Tools

- Introduce a shared local memory/tool catalog for Bita and sidecar.
- Use local-only storage by default.
- Add opt-in RAG/indexing for approved non-sensitive documents.
- Add a reviewed static tool registry before any dynamic tool creation.
- Consider a localhost-only API after filesystem queue stability.

### Phase 4: Optional Full Go Daemon

- Only after Bita workflows, approvals, logging, and report automation are stable should Ax4 consider a full Go daemon.
- The Go daemon would absorb scheduler, queue, tool registry, local API, and operator console responsibilities.
- Telegram authority should migrate only by explicit cutover plan with rollback to Python `bita-telegram.service`.

## 6. First Safe Sidecar Bridge Definition

### Filesystem Queue

Recommended queue layout, owned by Bita rather than Agenvoy:

```text
/tools/ax4-bita/var/sidecar/
  inbox/              # new work requests from Python Bita
  processing/         # claimed work items
  outbox/             # completed draft packages and status payloads
  failed/             # failed items with error metadata
  audit/audit.jsonl   # append-only state transitions
  reports/            # generated draft packages
```

Queue item schema:

```json
{
  "id": "uuid-or-timestamp-id",
  "created_at": "RFC3339 timestamp",
  "source": "bita-telegram",
  "requested_by": "telegram_user_or_operator_id",
  "workflow": "survey_report_draft",
  "approval_state": "operator_approved_for_draft_only",
  "input_ref": "path to local survey JSON or markdown",
  "constraints": {
    "external_writes": false,
    "email": false,
    "live_scans": false,
    "cloud_models": false
  }
}
```

### JSONL Audit Logs

Every transition should append one JSON object per line:

```json
{"ts":"RFC3339","item_id":"...","actor":"bita|sidecar|operator","event":"queued|claimed|drafted|failed|reviewed","path":"...","hash":"sha256 optional","notes":"short human-readable note"}
```

Minimum audit events:

- `queued`: Python Bita accepted an approved draft-only request.
- `claimed`: sidecar atomically moved item from `inbox` to `processing`.
- `drafted`: sidecar wrote report package to `reports/` and completion payload to `outbox/`.
- `failed`: sidecar captured a non-sensitive error and wrote failed item metadata.
- `reviewed`: operator marked draft reviewed, approved, rejected, or revised.

### No Live External Writes

Phase 1 sidecar must not:

- Send Telegram messages directly.
- Send email.
- Post to external services.
- Write to production/client systems.
- Install dependencies.
- Create or deploy runtime tools.
- Run network scans or scrape client assets.

## 7. What Must NOT Be Migrated Yet

- Full Go daemon replacement for current Python Bita.
- Telegram bot runtime cutover from `bita-telegram.service`.
- Discord runtime.
- Browser/webmode UI.
- Self-generated script tools and API tools.
- Extension upload/install registry workflows.
- Always-allow execution mode.
- Arbitrary shell command execution.
- Arbitrary filesystem write/patch tools.
- Email sending or user-email collection automation.
- External posting to Telegram, Discord, social platforms, or client systems without approval.
- MCP server connections to unreviewed tools.
- Multi-provider automatic cloud routing for client data.
- KuraDB/vector indexing of client data before privacy classification.
- Live web scans, vulnerability scans, or OSINT collection without explicit approval.
- Automatic dependency installation or package-manager execution.

## 8. Security and Compliance Guardrails

Mandatory Ax4/Bita guardrails for any adaptation:

- **No autonomous production tool deployment.** Generated tools must be sandboxed, tested, logged, reviewed, and manually approved before use.
- **No unsupervised external posting.** Telegram, email, Discord, social, ticketing, CRM, and client-system output requires human approval.
- **No emailing without approval.** Email tools stay disabled until approval, templates, recipient checks, and audit logging exist.
- **No live scans without approval.** Network, web, vulnerability, or data-discovery scans require explicit scope, authorization, and logging.
- **No client data to cloud models by default.** Default processing should be local or use sanitized summaries. Cloud model use must be opt-in per workflow/data class.
- **All generated tools sandboxed, tested, logged, and reviewed.** No direct execution of model-created code in production.
- **Least privilege for filesystem access.** Use explicit working directories and deny secrets such as `.ssh`, `.aws`, `.gcloud`, `.gnupg`, `.docker`, `.env`, key/cert files, shell histories, and credentials.
- **Append-only audit.** Queue events, tool decisions, approvals, and report package hashes should be recorded in JSONL.
- **Dry-run first.** New sidecar actions should produce draft files and status payloads, not external effects.
- **Human review remains the release gate.** Report drafts are not client deliverables until reviewed by an Ax4 operator.

## 9. First Revenue Workflow

Workflow name: `survey_report_draft`

```text
survey inputs -> structured report draft package -> human review
```

### Inputs

- Telegram-guided survey answers or uploaded survey JSON/Markdown.
- Optional local reference documents explicitly approved by the operator.
- Workflow constraints declaring no external writes, no email, no live scans, and no cloud models unless separately approved.

### Sidecar Processing

- Validate queue item schema.
- Normalize survey answers into a structured internal JSON document.
- Generate a draft report package under `reports/<item_id>/`:
  - `input.normalized.json`
  - `report.draft.md`
  - `findings.json`
  - `assumptions.md`
  - `review-checklist.md`
  - `audit-summary.json`
- Write an `outbox/<item_id>.json` completion payload for Bita/operator review.
- Append all state changes to `audit/audit.jsonl`.

### Human Review

- Operator reviews `report.draft.md`, assumptions, findings, and audit summary.
- Operator revises or approves the report.
- Only after approval may Bita help package a client-facing response.
- Delivery remains manual or approval-gated through current Bita Telegram workflow.

## 10. Practical Ax4/Bita Adoption Priorities

1. Define Bita-side queue and audit log schemas using Agenvoy's filesystem/session/log patterns as reference.
2. Add fixed, reviewed report-draft workflow definitions in Python Bita; do not add dynamic tools.
3. Add explicit operator approval states for draft generation and delivery.
4. Add safe local report package generation.
5. Add status summaries back to current Bita Telegram without allowing the sidecar to post directly.
6. After proving Phase 1, consider a Go sidecar that reads the same queue schema and produces the same artifacts.

