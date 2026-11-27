---
state: current
as-of: 38b9779
---

# A harness-agnostic plugin layer: preliminary findings

Research only: nothing in any repository was changed.
It maps what Claude Code, Gemini CLI (with Antigravity CLI beside it) and OpenCode offer a plugin, and sketches an API written once over all three.
The harnesses' documentation was read on 2026-11-27; `as-of` names the commit this repository stood at, since the report rests on no code here.

## Answer

- **Size: large overall, medium for a useful first cut.**
  The declarative half (instructions, skills, commands, agents, MCP, command hooks) is a file generator that others have already built.
  The programmable half (in-process hooks, UI, state) has no common model across the three and is where the work is.
- **The second harness is a moving target.**
  Google stopped serving Gemini CLI to consumer tiers on 2026-06-18 and points users to Antigravity CLI (`agy`), where extensions become "plugins".
  Gemini CLI still works with paid API keys and enterprise licences.
  Which one the adapter targets needs deciding before the design; both are mapped below.
- **OpenCode is mid-migration too.**
  Its docs describe a V2 plugin API (`Plugin.define({ id, setup(ctx) })` with server, TUI, GUI and RPC parts).
  The V1 form (a hooks object with `"tool.execute.before"`) is documented only as legacy.
- **Best base:** Agent Skills (`SKILL.md`) plus `AGENTS.md` plus MCP for the declarative part, and a small TypeScript core of the project's own for the rest.
  An existing MIT project, agent-plugin-sdk, already compiles one TypeScript definition to eight harnesses, but has no UI primitive and no documented degradation model.

## Capability matrix

Legend: (R) read in the docs or in Claude Code's bundled plugin-authoring reference; (I) inferred.

| Capability | Claude Code | Gemini CLI (Antigravity CLI in brackets) | OpenCode |
|---|---|---|---|
| Instructions / context | No plugin-level context file: a plugin-root `CLAUDE.md` is not loaded; use a skill, or mod events `prompt.section` / `prompt.context` (R) | `GEMINI.md` or `contextFileName` in the extension (R) [rules: always-on, glob- or model-triggered; `GEMINI.md` and `AGENTS.md` still read (R)] | `AGENTS.md` (I, page not read); plugin `instructions.entry.put` and session `context` hook pushing to `system` (R) |
| Skills | `skills/<name>/SKILL.md` (R) | `skills/<name>/SKILL.md` (R) [same; workspace `.agents/skills/` (R)] | `.opencode/skills`, also reads `.claude/skills` and `.agents/skills`; `ctx.skill` registry (R) |
| Slash commands | `commands/*.md`, or mod `$.command.register` and `command.run` (R) | `commands/*.toml`, nested as `/a:b` (R) [commands converted to skills on import; `/<skill-name>` (R)] | `.opencode/commands/*.md`, `commands` config key; TUI keymap commands with `slash` (R) |
| Subagents | `agents/*.md` (R) | `agents/*.md`, preview (R) [custom agents with `subagent` / `mainAgent` flags (R)] | `ctx.agent` registry (R); agent files (I, page not read) |
| Tool call, before | `PreToolUse` command hook: deny, ask, rewrite input; mod `tool.call` / `tool.check` / `tool.run` (R) | `BeforeTool`: deny, rewrite `tool_input` (R) [`PreToolUse` (R); blocking detail not stated] | `ctx.tool.hook` `execute.before`: replace input; denial is not documented there, it goes through `ctx.permission.hook` `evaluate` (R) |
| Tool call, after | `PostToolUse`, cannot block (R); mod `tool.call` around `next` can rewrite the result (R) | `AfterTool`: replace or hide result, add context, chain a tool call (R) [`PostToolUse` (R)] | `execute.after`: set `event.result` (R) |
| Prompt submit | `UserPromptSubmit` can block; mod `prompt.submit` (R) | `BeforeAgent`: deny, add context (R) [none listed (R)] | session hook `prompt`: rewrite text, add files or skills; no rejection API (R) |
| Session start / stop | `SessionStart` (no block), `Stop` (block), `SessionEnd`; mod `session.start`, `turn.complete`, `session.end` (R) | `SessionStart`, `SessionEnd` advisory; `AfterAgent` can force a retry (R) [`SessionStart`, `Stop` (R)] | `ctx.event.subscribe` on the server event stream; full event list not on the page (R) |
| Custom tools | MCP server, or mod `$.tool.register` (R) | MCP only (R) [MCP only (R)] | `ctx.tool.transform(editor => editor.add(...))`, in-process (R) |
| MCP servers | `.mcp.json` or `mcpServers` in `plugin.json` (R) | `mcpServers` in `gemini-extension.json` (R) [`mcp_config.json`, `serverUrl` key (R)] | `ctx.mcp` registry; config (R) |
| UI surfaces | Mod `ui.render` components `Pane`, `AbovePrompt`, `BesidePrompt`, `StatusLine`, `CodeBlock`, `ContextMenu`; `$.ui.toast`, `$.ui.notify`, `$.ui.open`; surfaces terminal, desktop, mobile, vscode (R) | Themes only; `systemMessage` from a hook (R) [none for plugins; MCP Apps HTML views (R)] | TUI part: slots `session.panel`, `sidebar.content`, `prompt.footer.status`, `home.footer.status` and others; `ui.toast.show`, dialogs, routes; Solid and OpenTUI JSX (R) |
| Persistent state | `${CLAUDE_PLUGIN_DATA}` directory; mod `$.store` (durable) and `$.state` (session, versioned) (R) | None; extension `settings` to `.env` and keychain (R) [not stated] | `ctx.storage` get/set/remove/scan; TUI `storage.store` and `storage.memory` (R) |
| Permissions / approval | `PreToolUse` `permissionDecision`, `PermissionRequest`; mod `tool.check` (R) | `policies/*.toml`, cannot `allow`; `Notification` observes only (R) [not stated] | `ctx.permission.hook` `evaluate` changes `effect`; `ctx.permission.assert` (R) |
| Model / system prompt | Output styles, `PreModelSwitch`; mod `prompt.section`, `turn.step`, `$.model.complete` (R) | `BeforeModel` rewrites the request or supplies a synthetic response; `BeforeToolSelection` filters tools (R) [not stated] | session hooks `context`, `model.request`, `experimental.model.resolve`, `compaction` (R) |
| Packaging / install | `.claude-plugin/plugin.json`, `marketplace.json`, `claude plugin install`, `--plugin-dir` (R) | `gemini extensions install <git url or path>`, `link` (R) [`plugin.json`, `agy plugin install`, `agy plugin import gemini` or `claude` (R)] | `plugins` array in `opencode.json`, npm or git, `.opencode/plugins/`, `opencode plugin add` (R) |

Docs per harness:

- Claude Code: <https://code.claude.com/docs/en/plugins-reference> and <https://code.claude.com/docs/en/hooks>, plus the plugin-authoring reference and mod type declarations bundled with Claude Code.
- Gemini CLI: <https://geminicli.com/docs/extensions/reference/> and <https://geminicli.com/docs/hooks/reference/>.
- Antigravity: <https://antigravity.google/docs/plugins?tab=cli>, <https://antigravity.google/docs/cli/gcli-migration> and <https://developers.googleblog.com/en/an-important-update-transitioning-gemini-cli-to-antigravity-cli/>.
- OpenCode: <https://opencode.ai/docs/plugins/>, <https://opencode.ai/docs/build/plugins/server>, <https://opencode.ai/docs/build/plugins/cli>, <https://opencode.ai/docs/skills> and <https://opencode.ai/docs/commands>.

## Shape of the agnostic API

Core abstractions, in two layers:

1. **Declarative assets** (compile-time, emitted as files): `instruction`, `skill`, `command`, `agent`, `mcpServer`, `setting`.
   These map cleanly.
   The only real translation is commands (Markdown to TOML for Gemini, to a skill for Antigravity) and where always-on context goes (Claude Code has no plugin context file).
2. **Runtime behaviours** (need a host): `intercept(event, handler)`, `tool(def)`, `surface(kind, render)`, `store`, `notify`.

Where the three differ in meaning, not name:

- **Execution model.**
  Gemini and classic Claude Code hooks are a process per call with JSON on stdin and stdout.
  Claude Code mods and OpenCode plugins are in-process and long-lived.
  A portable handler must therefore be a pure function of serialisable input, with no closure state, or it cannot run on Gemini.
  The adapter for process-per-call harnesses has to bundle the handler into a CLI entry point and pay start-up on every call.
- **Deny.**
  Claude Code and Gemini deny a tool call from the before-tool hook.
  OpenCode's `execute.before` is documented as inspect or replace; the documented veto is the permission `evaluate` hook, which does not run for configured denies.
  So "guard" is one concept on two harnesses and a different hook on the third.
  Whether throwing in `execute.before` blocks in V2 is unverified.
- **Fail-open vs fail-closed.**
  A Claude Code mod hook that throws is skipped unless it has a `.catch`; a command hook that exits non-zero other than 2 lets the action proceed; Gemini is the same.
  The API needs an explicit `onFailure: block | allow`.
- **After-tool power.**
  Gemini can replace the result and chain another tool call.
  Claude Code's `PostToolUse` cannot block; only a mod can rewrite the result.
  OpenCode can set the result.
- **Prompt hooks.**
  Claude Code and Gemini can reject a prompt; OpenCode can rewrite but has no rejection API.
- **Middleware vs event.**
  Claude Code mods are a `next(e)` chain; the others are fire-and-answer.
  The portable form is best modelled as answer-only (`allow | deny | rewrite | addContext`), with `next` kept as a Claude Code extension.
- **UI.**
  Claude Code draws a host-defined element tree per surface; OpenCode draws Solid JSX into named slots; Gemini has nothing.
  No shared component model exists.
  A portable UI would be a small retained tree (box, text, button, list) with two renderers, or only text-level surfaces.
- **Tool names and input shapes.**
  `Bash` vs `run_shell_command` vs OpenCode's names, with different argument keys.
  Every guard needs a normalised tool vocabulary; this table is permanent maintenance.

## Capability flags and degradation

Each adapter publishes a set such as `tool.before.deny`, `tool.before.rewrite`, `tool.after.rewrite`, `prompt.deny`, `tool.inprocess`, `ui.pane`, `ui.statusline`, `ui.toast`, `state.durable`, `state.session`, `runtime.longlived`.
Each feature declares `requires`, an ordered list of `fallbacks`, and `else: off | fail-build`, and the build prints a per-harness report of what was degraded.

- **PreToolUse guard.**
  Requires `tool.before.deny`.
  Claude Code: `PreToolUse` command hook or mod `tool.call`.
  Gemini: `BeforeTool` with exit 2.
  OpenCode: permission `evaluate` setting `effect` to deny; fallback, a policy rule in config.
  A security feature should declare `else: fail-build`, never silently off.
- **Pane.**
  Requires `ui.pane`.
  Claude Code: `Pane` via `$.ui.open`.
  OpenCode: `session.panel` slot with `ui.panel.open`.
  Gemini and Antigravity: fallback to a slash command that prints the same content as text, or an MCP tool; on Antigravity possibly an MCP Apps view.
  Notice at install.
- **Status line entry.**
  Requires `ui.statusline`.
  Claude Code: `StatusLine` render hook (or the user's `statusLine` command, which a plugin cannot set alone).
  OpenCode: `prompt.footer.status` slot.
  Gemini: off with a notice; fallback, a `systemMessage` at `SessionStart`.
- **Skill.**
  Requires nothing; all three read `SKILL.md`.
  Degradation is per field: `allowed-tools` is experimental, and Claude-specific frontmatter is dropped with a warning.

## Prior art

- **Agent Skills spec,** <https://agentskills.io/specification>: the one format all three read natively.
  Base for skills. (R)
- **AGENTS.md:** read by OpenCode and Antigravity; Claude Code needs an import from `CLAUDE.md`. (partly I)
- **MCP:** the only portable custom-tool route; also the tool fallback where in-process tools are missing.
- **agent-plugin-sdk,** <https://github.com/jal-co/agent-plugin-sdk> (MIT): define once in TypeScript, compile to Claude Code, Codex, Gemini CLI, Copilot, Cursor, Windsurf, Pi and OpenCode.
  Its primitives are instructions, skills, commands, subagents, hooks, MCP, tools and files.
  No UI, no stated degradation model.
  The closest candidate to build on or fork. (README read; code not)
- **wshobson/agents:** Claude Code Markdown as source, with generators for five harnesses.
  The Compound Engineering plugin CLI converts Claude Code plugins to other formats.
  agent-guard is one dispatch core with a thin hook adapter per harness, the right pattern for guards.
  (Search summaries only, not read.)
- **Antigravity's own `agy plugin import claude`** converts Claude Code plugins, which lowers the value of a hand-written adapter for the declarative half. (R)

## Complexity, risks, next round

- **Small:** declarative assets and MCP.
  **Medium:** portable command-style hooks with a normalised event and tool vocabulary.
  **Large:** in-process tools, UI and state, mainly because two of the three programmable APIs are early access and changing.
- **Riskiest unknowns:**
  - Whether the target is Gemini CLI or Antigravity CLI, and Antigravity's hook protocol: only four events are named, and their I/O and blocking are not documented on the pages reached.
  - OpenCode V1 vs V2: which versions users run, and whether `execute.before` can veto.
  - Claude Code mod API stability ("early access and moves between releases", per its own reference).
  - Whether a shared UI tree is worth it for two harnesses.
  - Tool-name and argument normalisation.
- **The next round:**
  - Build one guard and one pane on all three by hand, and record the differences.
  - Read agent-plugin-sdk's source and support matrix, to decide whether to fork it or build.
  - Pin the Antigravity hook contract from its schema.
  - Decide the handler constraint: pure and serialisable, or not.

## Not verified

- Claude Code's plugin-authoring reference was read only in part, and its type declarations were searched rather than read.
- OpenCode's pages for agents, rules, custom tools, permissions, and the GUI and RPC parts were not read.
- Claude Code's hooks page was read only through its first third.
- The Antigravity build page, <https://antigravity.google/docs/plugins/build>, returned 404.
- Gemini CLI's repository status was not checked.
- All web content came through a summarising fetch, so exact field names should be rechecked before they are relied on.
