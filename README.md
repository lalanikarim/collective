# collective — a fleet of project brains on the engram kernel

The flagship app on **[engram](https://github.com/lalanikarim/engram)** (pinned as an npm
dependency at the kernel tag — currently `v0.2.0` — composed via `createCoreInstance()`):
a durable, local-model **shared board** —
a standing team of named members (each with its own workspace, model, and domain memory)
coordinated by the root conversation, plus **remote members** living on other engram
instances.

Ecosystem framing (kernel vs. apps, solo/collective/society flows): engram's
[`docs/ecosystem-flows.md`](https://github.com/lalanikarim/engram/blob/main/docs/ecosystem-flows.md).

## Requirements

- Node ≥ 22.19 — native TypeScript type stripping, no compiler.
- A local OpenAI-compatible model endpoint (`OPENAI_BASE_URL` + `OPENAI_API_KEY`).
- Optional: Podman for sandboxed member conversations.

## Run locally

```bash
cp .env.example .env      # set OPENAI_BASE_URL, OPENAI_API_KEY, API_TOKEN
npm install               # installs engram@v0.2.0 from git (its prepare step compiles dist/)
npm start                 # node src/main.ts
```

The server binds to `HOST:PORT` (default `127.0.0.1:8791`). Storage: `data/agent.sqlite`,
member workspaces under `work/<name>/`, knowledge base in `kb/`. The Web UI is served by the
engram kernel at `http://HOST:PORT/` — since this app registers a `GET /board` route, its board
dashboard panel appears in the sidebar.

## Run tests

```bash
npm test   # 10 suites (live model where marked):
#   board.test.ts               deterministic: store, atomic claims, channels, knowledge,
#                               domain memory, routing allowlists, restart durability
#   board-live / -comms-live / -memory-live / -trust-live /
#   board-approval-live / -team-live   live-model member flows (A1–A4)
#   board-remote.test.ts        unit: token parsing, endpoint validation, exactly-once ids,
#                               remote registration/resolution, dispatch vs a stub peer
#   board-remote-live.test.ts   two live instances: coordinator routes over HTTP, peer
#                               transcript shows the untrusted-wrapped message
#   board-federation-live.test.ts  two live instances: peer claims + reports jobs over the
#                               M2.2 RPC, route-back with untrusted wrapping, A2 gates
```

## The board

The root conversation is the standing **coordinator**; named **members** are persistent
specialist conversations, each with its own workspace (`work/<name>`), optional model, domain
memory (`MEMORY.md` via `remember`/`recall`), and tool set. Members coordinate through board
docs — jobs (atomic single-winner `claim_job`), peer messages (`route_message`, delivered as
`<untrusted>` data), channels, and a knowledge base with provenance + trust tier. Peer routing
is allowlist-gated (`allowedTargets`); members can be disabled/enabled centrally and durably.
The human posts work to the coordinator, reads `GET /board` (or Telegram `/board`), and
approves — member side effects still route through the standard approval gates.

| Tool | Who | Purpose |
|---|---|---|
| `create_member` | coordinator/human | register a named member (own conversation + workspace) |
| `post_job` | coordinator/human | open a job on the board (atomic, indexed) |
| `claim_job` / `report_status` | members | atomic single-winner claim; done/failed + result |
| `route_message` | allowlisted members, coordinator, human | peer message, delivered as untrusted data, exactly-once |
| `board_status` | all members | members + open jobs |
| `remember` / `recall` | members | durable domain memory across sessions/compaction |
| `publish_knowledge` / `read_knowledge` | coordinator/human / all | knowledge posts with provenance + trust tier |
| `disable_member` / `enable_member` | coordinator/human | durable central freeze of a member |

## Remote members (M2.1) + board RPC (M2.2)

A member can live on *another engram instance*:

- **Transport (M2.1)** — the member's profile carries an `endpoint` (that instance's base URL)
  and a `tokenRef` (a name in this board owner's `BOARD_PEER_TOKENS` env; the bearer token
  itself is never stored in SQLite). `route_message` to a remote member POSTs to that
  instance's `/tasks` with a stable exactly-once request id, wrapped in `<untrusted>`
  delimiters (remote members default to `trustLevel: "untrusted"`). Registration goes through
  the coordinator's `register` tool or `POST /board/members`; the Web board panel marks such
  members with a `remote` badge. Design: engram's
  [`docs/plans/m2-remote-members.md`](https://github.com/lalanikarim/engram/blob/main/docs/plans/m2-remote-members.md).
- **Work loop (M2.2)** — the board owner exposes a peer-authenticated RPC surface
  (`GET/POST /board/jobs`, `POST /board/jobs/claim`, `POST /board/jobs/:id/status`,
  `POST /board/route`), and the remote instance installs the thin **board-client extension**
  (`BOARD_OWNER_URL` + `BOARD_MEMBER_ID`): tools `board_list_jobs` / `board_claim_job` /
  `board_report_status` so the member brain reads its pool, claims work, and reports results.
  Auth: the owner's `API_TOKEN` acts as `coordinator`; each `BOARD_PEER_TOKENS` value maps to
  that peer's member id. A2 invariants hold over HTTP — only coordinator/human post jobs,
  only assignees report on their own jobs, routing stays allowlist-gated.

## Outbound comms policy

Each engram can send communication out individually (its own Telegram bot), but for Collective
and Society members this is **disabled by default**: member instances run without
`TELEGRAM_BOT_TOKEN`, and only the coordinator/owner instance runs a bot unless a member's
outbound channel is selectively enabled. (Later phase: the collective/society Web UI can
embed each member's own Web UI.)

## Configuration

Core variables mirror engram's `.env.example`; this app adds:

- `BOARD_PEER_TOKENS` — comma-separated `name:token` pairs for peer instances this board owns
  routing to (owner side). The owner's own `API_TOKEN` acts as the `coordinator` identity on
  the RPC surface.
- `BOARD_OWNER_URL` + `BOARD_MEMBER_ID` — set on a *remote member* instance to install the
  board-client extension (the token comes from that instance's own `BOARD_PEER_TOKENS` entry).

Everything else (model catalog, approvals, sandbox, compaction) is configured exactly as in
engram — see its README.
