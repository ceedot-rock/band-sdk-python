# BAND SDK — First-Pass Analysis

Cloned 2026-10-19 from `ceedot-rock/band-sdk-python` (fork of `band-ai/band-sdk-python`).
Version: **5.1.0** (2026-10-13). Python SDK, ~54MB, `src/band/` + `packages/band-mcp` + `band-bridge/`.

This is what's actually in the code — not the launch marketing.

---

## 1. Overall architecture

BAND is a **centralized coordination platform**, not a protocol. The moving parts:

- **The platform** (`app.band.ai`, SaaS; on-prem also supported) owns rooms, agents, message delivery, task boards, memory, and contacts. The SDK is a client of it.
- **Agents are platform-registered entities.** You create an agent in the Band dashboard, get back an `agent_id` + `api_key`, and paste them into `agent_config.yaml`. Every adapter process authenticates with those bearer credentials. There is no self-sovereign identity — the platform is the root of trust.
- **The SDK** (`src/band/`) connects any of 15+ agent frameworks to the platform through a uniform adapter interface. Layout:
  - `adapters/` — one per framework (claude_sdk, codex, copilot_sdk, copilot_acp, opencode, cursor_acp, omp_acp, langgraph, anthropic, gemini, crewai, pydantic_ai, agno, strands, letta, parlant, google_adk, a2a, slack, nooa)
  - `core/` — protocols, types, turn lifecycle, the `SimpleAdapter` base every adapter extends
  - `runtime/` — execution context, room management, the `band_*` platform tools agents actually call
  - `client/` — transport: Phoenix Channels WebSocket (`client/streaming/client.py`) + Fern-generated REST client (`client/rest.py`)
  - `integrations/` — deep framework bindings (e.g. `integrations/codex/` has a `CodexStdioClient` speaking JSON-RPC over stdio)
  - `converters/` — per-framework history format converters
- **Transport:** Phoenix Channels WebSocket for real-time events (`wss://app.band.ai/api/v1/socket/websocket`), REST fallback. Message delivery has an explicit per-recipient lifecycle: `DELIVERED → PROCESSING → PROCESSED | FAILED`. `FAILED` is not terminal — the platform's `/next` endpoint re-serves failed messages, so a restarted agent can pick them up. This is a genuine at-least-once delivery guarantee, LLM-independent.
- **Entry point:** `src/band/agent.py` — the `Agent` class composes runtime + preprocessor + adapter. `Agent.create(...)` / `agent.run(...)` is the standard boot path.

In one sentence: BAND is Slack-for-agents with a real delivery layer, plus a Docker-compose kit that puts four different coding CLIs in one shared room.

---

## 2. The handoff mechanism (planner → coder → reviewer)

There is **no structured handoff protocol in the code**. This is the most important finding.

- `src/band/core/handoff.py` is about *context commitment* between adapter setup and execution (an internal SDK concern) — not agent-to-agent handoffs. The name is a red herring.
- Real handoffs are **prompt-engineered**. `src/band/prompts/roles.py` defines `generate_planner_prompt()` and `generate_reviewer_prompt()`, both embedding a shared `CONVERSATION_DISCIPLINE` block. The handoff protocol is literally:
  1. Finish your work.
  2. `@mention` the next agent ("Planning complete. Handing off to @agent/role for [next phase]").
  3. Go silent. Don't reply unless @mentioned again.
- The discipline rules read like a function-call convention: *"@mentioning an agent is like calling a function — it triggers them to respond."* Mention detection is textual (`@planner`, `@username/agent-name`), with carve-outs so `@dataclass` and email addresses don't trigger.
- **Content moves via shared files, not chat.** The `examples/coding_agents/prompts/planner.md` states it as a rule: *"Chat is for coordination, files are for content."* The docker-compose kit mounts four shared Docker volumes across all agent containers:
  - `/workspace/repo` — the git working tree (each container clones on startup; `repo.index: true` generates `context/structure.md`, `context/patterns.md`, `context/dependencies.md`, auto-injected into system prompts)
  - `/workspace/notes` — `plan.md` (planner owns), `review.md` (reviewer owns), plus free-form collaboration files
  - `/workspace/state` — persistent state across restarts
  - `/workspace/context` — generated codebase context
- **Task board** as a second coordination surface: `band_list_tasks`, `band_create_task`, `band_get_task`, `band_update_task`, `band_get_task_history`, `band_get_board`, `band_set_board` (all in `src/band/runtime/tools/types.py`).
- **Platform memory** as a third: `band_store_memory`, `band_list_memories`, `band_get_memory`, `band_supersede_memory`, `band_archive_memory` (typed: `MemoryType`, `MemorySegment`, scopes in `core/memory_types.py`).

So the "four agents, one workspace" claim decomposes to: one chatroom + @mention discipline + shared Docker volumes + a task board. It works because the prompts are carefully written, not because anything is enforced. A planner that ignores the discipline and dumps a plan in chat breaks nothing structurally.

---

## 3. The SDK adapters (Claude Code, Codex CLI, Copilot, OpenCode)

Every adapter extends `SimpleAdapter` (`src/band/core/simple_adapter.py`) with a uniform constructor shape:
`XAdapter(config: XAdapterConfig, *, history_converter=..., additional_tools=..., **features)`.
Configs are frozen Pydantic models (`core/adapterconfig.py`) loadable from YAML. Feature flags (`emit=`, `capabilities=`, `include_tools=`) negotiate capabilities against what the platform advertises (`runtime/capabilities.py`).

How each of the four named integrations actually plugs in:

| Agent | Adapter file | Mechanism |
|---|---|---|
| **Claude Code** | `adapters/claude_sdk.py` (~2100 lines) | Wraps the `claude_agent_sdk` Python package (`ClaudeSDKClient`). Band platform tools are exposed to Claude as **MCP servers, per room** — tools are stored per-room-id so the MCP server resolves them in the right room context. |
| **Codex CLI** | `adapters/codex.py` (~4400 lines) | Talks to the **Codex app-server over JSON-RPC via stdio** (`integrations/codex/CodexStdioClient`). Manages Codex sessions, approval methods (`CODEX_APPROVAL_METHODS`), token usage, and dynamic tool calls. The biggest adapter by far. |
| **GitHub Copilot** | `adapters/copilot_sdk.py` + `adapters/copilot_acp.py` | Two paths: the Copilot SDK, and **ACP (Agent Client Protocol)** — JSON-RPC over stdio, the same protocol Zed/Cursor/JetBrains/Neovim speak (`docs/acp.md` covers session lifecycle and permission pairing). |
| **OpenCode** | `adapters/opencode/` (`adapter.py` + `approvals.py`) | Maps **each Band room to a session on a running `opencode serve`** instance (HTTP client in `integrations/opencode/`). Per `docs/adapters/opencode.md`, Band tools are never gated behind approval here — an explicit invariant, unlike other tool calls. |

Also notable: `adapters/acp.py` + `ACPServer`/`BandACPServerAdapter` let BAND act as an **ACP server** too (editors connect to it), and `adapters/a2a.py` / `a2a_gateway.py` bridge to Google's **A2A protocol** in both directions.

**Turn lifecycle** (`core/turn.py`, `core/turn_lifecycle.py`): every turn is judged by `band-sdk-core`'s rule — a turn *completes* when it replied, declined via `band_no_reply`, did real work, or was settled by the adapter; anything else is reported once and marked **FAILED**. Adapters never judge their own turns. There's an approval-interrupt path (`ApprovalInterruptMixin`, `ApprovalMode` in `core/types.py`) for human-in-the-loop gating — but it's per-turn interactive approval, not a standing mandate.

---

## 4. The chatroom / coordination layer (technical)

- **Rooms** are the unit of coordination. `runtime/rooms.py` manages `RoomExecutions`: admission control (`admit`/`hold`/`withdraw`/`converge`) decides which agent process is currently serving a room. This is how they avoid two processes double-processing the same room.
- **The tool surface** every agent gets (the actual coordination API, `runtime/tools/types.py`):
  - Messaging: `band_send_message`, `band_send_event`, `band_no_reply`
  - Membership: `band_add_participant`, `band_remove_participant`, `band_get_participants`, `band_lookup_peers`, `band_create_chatroom`
  - Contacts: `band_list/add/remove_contacts`, contact request flow
  - Memory: store/list/get/supersede/archive
  - Files: `band_list_room_files`, `band_read_room_file`, `band_send_room_file` (note: room file storage is an **on-prem-only** deployment flag, off on SaaS)
  - Tasks/board: list/create/get/update tasks, get/set board
- **Identity of participants** is `@username/agent-name` handles or role names (`@planner`). The platform resolves mentions and routes messages. `is_authorized_sender` (from `band_sdk_core`) gates who can trigger an agent.
- **Retention/idle:** `core/retention/` has a gated adapter-owned room-retention lifecycle (5.0.0) — rooms can be suspended when idle and resumed, with shared room-ownership prerequisites (5.1.0, #781).
- **MCP:** one framework-neutral engine (`integrations/mcp/engine.py`) builds every Band MCP tool registration for both the published `band-mcp` CLI package (`packages/band-mcp/`) and the embedded `LocalMCPServer`. MCP-package imports are AST-allowlisted (`tests/mcp/test_import_boundary.py`).

Net: a well-engineered group-chat runtime with delivery guarantees, admission control, and a typed tool surface. The "coordination" is real infrastructure, not vibes — but the *agent-to-agent protocol* on top of it is prompt text.

---

## 5. BAND vs. Agent-Rider + Warrant

### Where they overlap
- Both have the **room** as the coordination primitive (BAND rooms ≈ Rider's `#lab-team` room).
- Both do agent-to-agent messaging with delivery semantics.
- Both have memory and a notion of agent identity/handles.

### Where they diverge — and it's structural

| Dimension | BAND | Agent-Rider + Warrant |
|---|---|---|
| **Identity** | Platform-issued `agent_id` + `api_key` (bearer secrets). The platform is the root of trust; no BAND, no identity. | Self-sovereign keypairs; ES256-signed JWTs verifiable by anyone via public JWKS. Portable across platforms. |
| **Authority / delegation** | None as a primitive. An agent acts with the full authority of its API key. Gating is per-turn human approval (`ApprovalMode`) or prompt discipline. | **Warrant**: signed, chained, *attenuating* mandates. A delegate structurally cannot mint authority its parent never had. Machine-enforceable, not prompt-suggested. |
| **Handoff enforcement** | Prompt text: "go silent after handoff." An LLM that ignores it breaks nothing structurally. | Attenuation is in the format itself — the math, not the manners. |
| **Trust boundary** | Trust `app.band.ai` (or your on-prem deploy). Centralized. | Verify a signature. Decentralized. |
| **Spend / budgets** | No concept of budgets, caps, or metering anywhere in the SDK. | Warrant's `max_cents` chains are the whole point; the lab's x402 rail meters per call. |

### The honest assessment

BAND is **strictly better** at the thing it does: multi-framework agent coordination. Fifteen-plus adapters, a real turn lifecycle with failure judgements, at-least-once delivery, admission control, a task board, A2A bridging, ACP editor integration — this is a serious, production-shaped coordination layer, and the lab has nothing comparable for the "four coding agents in one room" use case.

Rider + Warrant is **strictly better** at the thing BAND doesn't do: trust. BAND has a gaping hole exactly where Warrant sits — nothing in BAND stops an agent from exceeding its intended authority except prompt discipline and optional human approval. A compromised or buggy BAND agent holds the full API key. There is no delegation chain, no attenuation, no signed proof of what was authorized.

**They're complementary layers, not competitors.** BAND answers "how do agents talk to each other." Rider + Warrant answers "how do agents prove who they are and what they're allowed to do." The interesting play: Rider identity + Warrant mandates *as the auth/delegation layer underneath* a BAND-style room. BAND's `agent_id`/`api_key` bearer model is precisely the "copied API key that confers unlimited authority" problem the NSF pitch describes — and BAND is now the concrete exhibit A.

### Concrete integration angles (if Corey wants them)
1. **Rider as a BAND adapter**: write a `RiderAdapter` (following the `SimpleAdapter` shape in `core/simple_adapter.py`) where the "framework" is a Rider-identified agent. BAND rooms gain portable identity for free.
2. **Warrant-gated tool calls**: BAND's approval-interrupt path (`ApprovalInterruptMixin`) is the seam — a Warrant verifier could auto-approve/deny tool calls against a mandate chain instead of asking a human every turn.
3. **The pitch exhibit**: BAND's bearer-key auth is the status quo the Warrant pitch argues against. It's now a named, shipped, funded example — useful in the NSF narrative and in sales.

---

## Files that matter most (if you go deeper)

- `src/band/prompts/roles.py` — the entire "handoff protocol" (prompt text)
- `examples/coding_agents/` — docker-compose, `agent_config.yaml.example`, `prompts/planner.md`, `prompts/reviewer.md`
- `src/band/core/simple_adapter.py` — the interface every adapter implements
- `src/band/core/turn.py` + `core/turn_lifecycle.py` — turn judging, approvals
- `src/band/runtime/tools/types.py` — the full `band_*` tool surface
- `src/band/runtime/rooms.py` — room admission/execution gating
- `src/band/client/streaming/client.py` — Phoenix Channels transport, delivery lifecycle
- `src/band/adapters/opencode/adapter.py` — cleanest adapter to study (HTTP, not stdio)
- `docs/acp.md`, `docs/websocket-events.md`, `docs/turn-outcome.md` — the three best docs
