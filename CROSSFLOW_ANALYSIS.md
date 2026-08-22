# Cross-Repo Feature Transfer Analysis

**Repos compared:**
- **openclaude** (`/Users/jkneen/Documents/GitHub/openclaude`) — modern Claude Code fork, TS + Ink/React, multi-provider (200+ LLMs), v0.29.x
- **Dario Code** (`/Users/jkneen/Documents/GitHub/flows/open-claude-code/open_claude_code`) — standalone reimplementation of Claude Code 2.1.x, plain JS (`.mjs`), v1.1.2, ~97% parity with Claude Code 2.1.50

**Date:** 2026-08-22
**Worktree:** `/Users/jkneen/Documents/GitHub/openclaude-crossflow-analysis`
**Updated:** 2026-08-22 — A1 (steering overlay) corrected after a full read of `src/components/permissions/AskUserQuestionPermissionRequest/` revealed openclaude already ships a richer version of the same overlay. The earlier claim that openclaude "has no tabbed multi-question overlay" was wrong (the fact-checker's directory scan missed this folder).

---

## 1. Direction A — Dario → openclaude (primary)

Dario is an *older, simpler* codebase. A close read of its headline features shows most are already covered by openclaude — including its steering-question overlay, which openclaude supersedes in the permission layer. The one genuinely novel feature is the source badge (A2).

### A1. Steering Questions overlay — DO NOT PORT (already superseded)
- **Source:** `src/tui/claude/components/steering-questions.mjs` (457 lines)
- **What:** Multi-tab interactive question overlay — tabbed questions with single/multi-select, custom text input, "chat about this" escape hatch, submit-review tab. Answers keyed by tab ID.
- **Finding:** openclaude **already has a superset** of this overlay in the permission layer — `src/components/permissions/AskUserQuestionPermissionRequest/`:
  - `QuestionNavigationBar.tsx` — the same ☐/✔ tab bar + Submit tab + `←`/`→` arrows (with width-aware truncation openclaude-style).
  - `QuestionView.tsx` — options with `→` focus, multiSelect checkboxes, a "Type something." custom text input, an automatic "Other" (`__other__`) option, a "Respond to Claude" escape hatch (Dario's "Chat about this"), plus previews and image paste Dario lacks.
  - `SubmitQuestionsView.tsx` — answer review + submit/cancel, warning on unanswered questions.
  - Driven by `use-multiple-choice-state.ts`; tab switching via `tabs:previous`/`tabs:next` keybindings. It opens whenever the model calls `AskUserQuestionTool`.
- **What's actually missing:** only the **user-initiated trigger**. Dario exposes a hidden `/steer` command (`steer-overlay`, `src/tui/claude/main.mjs:837`) that opens the overlay with hardcoded demo questions when run manually, plus a Promise-based `showSteeringQuestions(data)` hook (`main.mjs:3503`) callable from the agent loop. openclaude's dialog opens only when the model calls the tool — there is no `/steer` command a user can run to launch their own structured requirement-gathering session.
- **Recommendation:** do NOT port the component — the existing one is strictly richer. If wanted, add a small `/steer` slash command that reuses `AskUserQuestionPermissionRequest` rather than a new overlay. The only micro-feature Dario has that the existing dialog lacks is number-key quick-select (1–9).

### A2. Dual config reading + Source Badge — MEDIUM VALUE, CONCEPT
- **Source:** `src/tui/claude/components/source-badge.mjs` (60 lines), `src/config/` (dual read of `~/.dario` + `~/.claude`)
- **What:** Dario reads config from **both** `.dario` (primary, writable) and `.claude` (shared/read-only) and renders a small badge (`[OC]`, `[CC]`, `[PRJ]`, `[OC+CC]`) next to each setting in the `/config` UI so the user knows *where* a value comes from.
- **Why useful:** openclaude already reads legacy `~/.claude` settings (settings.ts has a `getSettings_DEPRECATED` path) but shows no provenance. When a value silently comes from an old `.claude` file, users get confused. The SourceBadge pattern is a cheap, high-clarity win for openclaude's `/config` and `/plugin` UIs.
- **Port effort:** Small (60 lines + resolveSource helper). The dual-read already exists in openclaude's settings layer; only the badge UI + wiring is new.
- **Requirement:** openclaude's settings read currently may not track *which file* a value came from — needs a small source-tracking refactor in `src/utils/settings/settings.ts`.

### A3. NPM plugin install path — LOW-MEDIUM VALUE, PARTIAL
- **Source:** `src/plugins/{installer,registry,loader,discovery,manifest}.mjs`
- **What:** Dario can install a plugin straight from the npm registry: `npm install` a package, verify it has a valid `manifest.json` (schema-validated: name/version/description/commands/tools/config), and hot-load it.
- **Why:** openclaude's plugin system is *marketplace-based* (git repos, `@marketplace` IDs, blocklists, trust warnings — much more sophisticated) but does **not** have a plain "install a package from npm" path. A Dario-style npm-install path is a complementary option.
- **Port effort:** Medium. openclaude's `src/plugins/` + `src/types/plugin.ts` are TS and marketplace-centric; would need a new `installFromNpm` mode mapped onto existing plugin types. Note the different manifest schemas (Dario `manifest.json` vs openclaude's plugin format) — adapt, don't copy.
- **Caveat:** This is arguably a *downgrade* vs. openclaude's marketplace system. Recommend porting only if the team wants direct npm installs as a convenience.

### A4. TUI variant loader — LOW VALUE, CONCEPT ONLY
- **Source:** `src/tui/loader.mjs` (claude / minimal / custom TUI variants, switched via env `DARIO_TUI`)
- **What:** Runtime-switchable TUI implementations.
- **Why:** openclaude has exactly one Ink UI. A "minimal" TUI (plain prompt, no panels) is a real use case (low-memory, SSH, screen-reader, CI). openclaude has `outputStyles` but those affect model output, not the terminal chrome.
- **Port effort:** High — openclaude's TUI is deeply integrated (App.tsx, screens, ink), not a swappable module. **Recommendation: don't port the loader**; instead note the *concept* of a `--minimal-ui` flag that renders a simpler `App` variant if ever needed.

### A5. Background task dependency tracking — LOW VALUE
- **Source:** `src/tasks/background.mjs` — spawn/kill/list background tasks, output buffering (10k lines / 5MB caps)
- **Why:** openclaude already has a far richer task system (`src/tasks/` with `LocalAgentTask`, `InProcessTeammateTask`, `MonitorMcpTask`, `RemoteAgentTask`, etc.) plus background Bash. Dario's manager is more primitive; nothing to take.

### A6. Everything else Dario has that openclaude already covers
| Dario feature | openclaude equivalent |
|---|---|
| `/review` command (coloured git diff) | `src/commands/review/` |
| `/pr-comments` | `src/commands/pr_comments/` |
| `/providers`, provider manager | `src/commands/model/`, `ProviderManager.tsx`, `src/integrations/` (200+ providers) |
| Plugin lifecycle (`/plugin`) | `src/commands/plugin/` + marketplace (richer) |
| Memory auto-extraction | `src/memdir/autoExtractFacts.ts` (vector index + team memory, richer) |
| Notifications | `src/services/notifier.ts`, `src/hooks/notifs/` |
| PTY / embedded terminal | `TerminalCaptureTool`, `TungstenTool` |
| WebSearch/WebFetch | `src/tools/WebSearchTool/`, `WebFetchTool/` |
| `/keybindings` | `src/keybindings/` (full parser/schema/resolver, richer) |
| OAuth | `src/services/oauth` |
| Auto-compact AI summarization | `src/services/compact/` (autoCompact.ts, reactiveCompact.ts) |
| Prompt footer / status line | openclaude's statusline components |

---

## 2. Direction B — openclaude → Dario (secondary)

Dario targets ~97% parity with Claude Code 2.1.50 but is built on an older feature set. openclaude, as a much newer fork, has major systems Dario lacks. These are the highest-value ports **to** Dario.

### B1. Multi-provider support — HIGHEST VALUE, LARGE EFFORT
- **Source:** `src/integrations/` (registry, descriptors, vendors, generated catalog), `src/commands/model/ModelPicker.tsx`, `ProviderManager.tsx`
- **What:** 200+ model providers — OpenAI, Gemini, DeepSeek, Ollama, Codex, Atomic Chat, LM Studio, etc. Dario currently supports only Anthropic + a basic OpenAI-compat shim (`src/providers/client-factory.mjs` converts Anthropic request shape → OpenAI chat completions).
- **Port effort:** Large. Dario's client-factory is a good *foundation* (it already has `convertMessagesToOpenAI`, `convertToolsToOpenAI`, streaming, local model alias resolution) but needs: provider registry/config discovery, per-provider auth, model catalog, cost tracking, and per-provider quirks (tool calling formats, thinking, max tokens). This is the single biggest feature gap.
- **Recommendation:** Port openclaude's *provider descriptor model* (id, baseURL, auth mode, model listing, isLocal) — a compact subset — into `src/providers/registry.mjs`, rather than the whole generated catalog.

### B2. Coordinator / swarm mode — HIGH VALUE, MEDIUM-LARGE EFFORT
- **Source:** `src/coordinator/coordinatorMode.ts`, `workerAgent.ts`; `TeamCreateTool`, `SendMessageTool`, `ListPeersTool` equivalents
- **What:** Multiple agent processes with a coordinator, peer messaging, task fan-out.
- **Why:** Dario has subagents (`src/agents/`) but only single-parent spawning; no coordinated multi-agent runs or inter-agent messaging.
- **Port effort:** Medium-large. Dario's `src/agents/` + `src/tasks/background.mjs` provide the process-management substrate; needs the coordinator state machine + messaging protocol on top.

### B3. Modern memory (vector + team) — MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/memdir/` — `memdir.ts`, `vectorIndex.ts`, `memoryScan.ts`, `teamMemPaths.ts`, `memorySecurity.ts`
- **What:** Auto fact extraction, embedding-based retrieval, per-project + team-shared memory directories with security checks.
- **Why:** Dario's `src/memory/auto-memory.mjs` is a simpler extract-and-store watcher; no vector retrieval or team sharing.
- **Port effort:** Medium. Dario already has the extraction concept; adding a vector index + team paths is additive.

### B4. Keybindings system — MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/keybindings/` — parser, schema, resolver, validation, `useKeybinding`, loadUserBindings
- **What:** Declarative keybinding schema with user overrides and validation errors.
- **Why:** Dario's `src/keyboard/` is hand-rolled (`keyboard/config.mjs`, `vim-mode.mjs`, `history-search.mjs`). openclaude's schema-driven approach is more maintainable and supports user-defined bindings.
- **Port effort:** Medium; Dario's Ink input layer needs to be refactored to dispatch via a resolver.

### B5. MCP resource + auth tooling — MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/tools/McpAuthTool/`, `src/tools/ListMcpResourcesTool/`
- **What:** OAuth flows for MCP servers and listing server resources.
- **Why:** Dario has basic MCP (`src/integration/mcp.mjs`) but no auth or resource listing.
- **Port effort:** Medium.

### B6. Query engine hardening — MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/query/` — `toolFailureLoopGuard.ts`, `tokenBudget.ts`, `transitions.ts`; `src/services/compact/` — `autoCompact.ts`, `reactiveCompact.ts` (auto-compact cooldown / state); `src/services/goal/` — `controller.ts`, `evaluator.ts`, `instructions.ts` (goal continuation)
- **What:** Compact cooldowns to stop prompt-cache busting, tool-failure loop detection, goal-continuation, token budgeting.
- **Why:** Dario has AI-compact but no loop guard or cooldown logic. `toolFailureLoopGuard` especially prevents the classic "model retries the same failing tool forever" pathology.
- **Port effort:** Medium; these are mostly self-contained algorithms.

### B7. i18n — LOW-MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/i18n/` (locales, `localize()`, `commandDescriptions.ts`)
- **What:** Full internationalization layer.
- **Why:** Dario is English-only hardcoded strings.
- **Port effort:** Medium (touch every command string). Worth it only if Dario targets non-English users.

### B8. Proactive / scheduled agents — LOW-MEDIUM VALUE, MEDIUM EFFORT
- **Source:** `src/proactive/index.ts`, `ScheduleCronTool`, `SleepTool`
- **What:** Scheduled/periodic agent runs.
- **Why:** Dario has `/loop` (run prompt on interval) — a lightweight cousin — but no cron or self-rescheduling tool.
- **Port effort:** Medium. Consider porting just the cron scheduling substrate.

### B9. Modern tool suite — CHECKLIST (smaller ports)
openclaude tools that Dario lacks entirely (each a standalone port):
- `EnterPlanModeTool` / `ExitPlanModeTool` / `VerifyPlanExecutionTool` (plan-mode state machine)
- `TerminalCaptureTool` (interactive CLI capture) — Dario has a bare `src/pty/session.mjs`
- `WebBrowserTool` (headless browser) 
- `WorkflowTool` (persistent workflows)
- `TaskCreate/TaskUpdate/...` tools (structured task tracking) — Dario has `/todos` but no tool surface
- `SnipTool` (context pruning) — highly valuable, small
- `ToolSearchTool` (lazy tool discovery) — medium
- `ScheduleCronTool`/`SleepTool` (see B8)

---

## 3. Prioritized Recommendations

### Take Dario → openclaude (do these)
1. ~~**Steering Questions overlay**~~ — **retracted**: openclaude already ships a richer tabbed multi-question overlay in `AskUserQuestionPermissionRequest/`. Only missing piece is a user-initiated `/steer` command (optional, small) that reuses the existing dialog.
2. **Source Badge + config provenance tracking** → small settings-layer refactor + badge component; high clarity payoff in `/config`.

### Take openclaude → Dario (do these)
1. **Provider descriptor model** (subset of `src/integrations/`) — unblocks every non-Anthropic user; Dario's client-factory already half-supports it.
2. **Coordinator/swarm mode** — biggest functional delta for agent workflows.
3. **Tool-failure loop guard + compact cooldown** — self-contained, prevents real pathologies.

### Don't port
- Dario's plugin npm-install path (openclaude marketplaces are better) — unless direct-npm is a product goal.
- Dario's TUI loader (openclaude TUI too integrated to make swappable cheaply).
- Dario's background task manager (openclaude's task system is strictly richer).
- Dario's steering-questions component (openclaude's `AskUserQuestionPermissionRequest` overlay is a strict superset).

---

## 4. Key Files Reference

**Dario → openclaude**
| Dario file | What to take | openclaude target |
|---|---|---|
| `src/tui/claude/components/steering-questions.mjs` | `/steer` trigger only (overlay already exists) | `src/commands/` (optional) |
| `src/tui/claude/components/source-badge.mjs` | Source badge | `src/components/` (new) + `src/utils/settings/settings.ts` |
| `src/config/` | Dual `.dario`+`.claude` read pattern | settings layer (adapt) |
| `src/plugins/installer.mjs` | npm-install path (optional) | `src/plugins/` |

**openclaude → Dario**
| openclaude source | What to take | Dario target |
|---|---|---|
| `src/integrations/registry.ts`, `descriptors.ts` | Provider descriptor model | `src/providers/registry.mjs` |
| `src/coordinator/coordinatorMode.ts` | Coordinator state machine | `src/agents/` |
| `src/query/toolFailureLoopGuard.ts`, `src/services/compact/autoCompact.ts`, `src/services/goal/controller.ts` | Loop guard + compact cooldown + goal continuation | `src/core/` or `src/session/` |
| `src/memdir/` (subset) | Vector + team memory | `src/memory/` |
| `src/keybindings/` | Keybinding schema | `src/keyboard/` |
| `src/tools/McpAuthTool/`, `ListMcpResourcesTool/` | MCP auth/resources | `src/integration/mcp.mjs` |
| `src/i18n/` | Localization | global |

---

## 5. Effort Summary

| Port | Direction | Effort | Value |
|---|---|---|---|
| Steering questions overlay | D→O | — | None (already superseded) |
| `/steer` user trigger (optional) | D→O | S | Low-Med |
| Source badge + provenance | D→O | S-M | Medium |
| Provider descriptor model | O→D | L | Highest |
| Coordinator/swarm | O→D | M-L | High |
| Tool loop guard + compact cooldown | O→D | M | Medium-High |
| Vector/team memory | O→D | M | Medium |
| Keybindings schema | O→D | M | Medium |
| MCP auth/resources | O→D | M | Medium |
| i18n | O→D | M | Low-Med |
| Cron/scheduled agents | O→D | M | Low-Med |
| NPM plugin install | D→O | M | Low (optional) |
| TUI variant loader | D→O | L | Low (concept only) |
