---
state: current
as-of: 60914d3
---

# A harness-agnostic plugin layer: design overview and evidence

Research only: nothing was built or run.
It maps what Claude Code, Antigravity CLI (`agy`) and OpenCode offer a plugin, from their docs, published schemas and source, and lays out an API written once over all of them.
Everything was read on 2026-11-27; `as-of` names the commit this repository stood at, since the report rests on no code here.

## Answer

- **Build fresh, in TypeScript, and borrow three designs.**
  No existing project covers runtime hooks, capability fallbacks, UI and state across these harnesses.
  agent-plugin-sdk is a static file generator whose hook is a shell-command string; its `Harness` registry and structured build warnings are worth copying under its MIT licence.
- **There are four targets, not three.**
  OpenCode ships two release lines at once, 1.x and 2.x, with different plugin APIs, package names, config keys, tool ids and argument keys.
  A 1.x plugin does not run on 2.x, but one package can export both forms.
- **Antigravity CLI replaces Gemini CLI as the second harness.**
  Google stopped serving Gemini CLI to consumer tiers on 2026-06-18 and points users to Antigravity CLI.
- **A tool-call guard is portable to all four targets.**
  Every target can veto a call and rewrite its input before it runs.
  Asking the user is missing on OpenCode 1.x, and rewriting a tool's result is missing on Antigravity.
- **Failure behaviour differs on every target and must be explicit.**
  Claude Code and Antigravity fail open; on OpenCode a hook that throws vetoes the call, so a crashing guard fails closed.
- **The handler constraint is settled:** a handler is a function of a serialisable event and an injected host object, with no closure state and no Node built-ins.
  Antigravity runs a process per call, and Claude Code's in-process API has no Node and no npm imports.
- **On Claude Code, command hooks are the default and the TypeScript mod API is opt-in.**
  The mod API is the only route to panes, durable state and in-process tools there, and its own type declarations say it "may change between releases without notice".
- **UI is a two-renderer feature.**
  Claude Code's mod API and OpenCode's terminal UI plugins can both draw a pane and a status entry; Antigravity CLI offers a plugin nothing to draw.
- **Prototype first:** one guard, as a vertical slice through all four targets, with a fixture suite of each harness's recorded payloads.

## Design overview

### Core abstractions

Two layers, as before, now with their shapes fixed by the evidence:

1. **Declarative assets,** emitted as files at build time: `instruction`, `skill`, `command`, `agent`, `mcpServer`.
   All four targets read `SKILL.md`, and three of them take agents as Markdown with frontmatter.
   The translations are few: a command becomes a skill on Antigravity, always-on context becomes a plugin rule on Antigravity and a skill or prompt section on Claude Code, and the MCP file changes name and remote-URL key.
2. **Runtime behaviours,** each a handler or definition the adapter hosts:
   - `guard(match, handler)`: before a tool call; answers `allow`, `deny(reason)`, `ask(reason)` or `rewrite(patch)`.
   - `after(match, handler)`: after a tool call; observes, or answers `replaceResult(value)`.
   - `onPrompt(handler)`: answers `pass`, `reject(reason)`, `rewrite(text)` or `addContext(text)`.
   - `onSessionStart(handler)`: answers `addContext(text)`.
   - `onStop(handler)`: answers `stop` or `continue(reason)`, with a bound the layer enforces.
   - `tool(def)`: a custom tool with a JSON Schema input and an `execute` function.
   - `surface(kind, render)`: `pane`, `status` or `toast`, drawn from a small element tree.
   - `host.store`: durable JSON keyed by string; `host.exec`, `host.fetch` and `host.log` beside it.

Handlers answer; they do not wrap.
Claude Code's mod API is a `next(e)` middleware chain, and nothing else is, so the portable form is answer-only and `next` stays a Claude Code escape hatch.

### The handler constraint

A portable handler is `(event, host) => answer`, async, with these rules:

- **Serialisable in, serialisable out.**
  On Antigravity and on Claude Code command hooks the handler runs in a fresh process per call, fed JSON on stdin.
- **No closure state.**
  Anything kept between calls goes through `host.store`.
- **No Node built-ins and no runtime `import()`.**
  A Claude Code mod "runs in an environment of its own, with no DOM and no Node: everything outside it is reached through `$`", and a module that calls `import()` does not load.
  The build bundles each handler and its dependencies into one ES module per target.
- **Everything outside goes through `host`.**
  The adapter backs `host.exec` with `$.process.run` in a mod, with a child process under a command hook, and with the plugin context's shell on OpenCode.

A handler that breaks these rules is still allowed, marked `requires: runtime.longlived`, and then builds only for the in-process targets.

### The decision vocabulary, and where it bends

| Answer | Claude Code | Antigravity CLI | OpenCode 1.x | OpenCode 2.x |
|---|---|---|---|---|
| `deny` | `permissionDecision: "deny"`, or exit 2 | `decision: "deny"` with `reason` | throw in `tool.execute.before` | fail with `Tool.Error` in `execute.before` |
| `ask` | `permissionDecision: "ask"` | `decision: "ask"` or `"force_ask"` | none | permission `evaluate` sets `effect` to `ask` |
| `rewrite` | `updatedInput`, which replaces the whole input | `overwrite`, a shallow top-level merge | assign into `output.args` in place | assign `event.input` |
| `replaceResult` | `updatedToolOutput` | none | assign into the after hook's `output` | assign `event.result` |

`rewrite` is therefore defined as a patch of top-level keys.
The Claude Code adapter expands the patch into the full input, the Antigravity adapter passes it through, and the OpenCode adapters assign key by key, since reassigning `output.args` on 1.x changes nothing.

Every guard also declares `onFailure: block | allow`, because the targets disagree:

- Claude Code command hooks fail open unless the hook sets `"onFailure": "block"`, which needs version 2.1.295 or newer.
- A Claude Code mod hook that throws is skipped unless its registration has a `.catch`.
- Antigravity documents no exit-code channel, so the wrapper must catch every error itself and print a deny.
- OpenCode 1.x treats any throw as a veto, so `onFailure: allow` needs a wrapper that swallows errors.

### Capability flags and fallbacks

Each adapter publishes the flags it has.
Each feature declares `requires`, an ordered list of `fallbacks`, and `else: off | fail-build`.
The build prints, per target, what ran natively, what fell back and what was turned off, and fails where a feature said `fail-build`.

| Flag | Claude Code | Antigravity CLI | OpenCode 1.x | OpenCode 2.x |
|---|---|---|---|---|
| `tool.before.deny` | yes | yes | yes | yes |
| `tool.before.ask` | yes | yes | no | yes |
| `tool.before.rewrite` | yes | yes | yes | yes |
| `tool.after.observe` | yes | yes | yes | yes |
| `tool.after.rewrite` | yes | no | yes | yes |
| `prompt.reject` | yes | by terminating the model call | no | no |
| `prompt.rewrite` | mod only | no | yes | yes |
| `prompt.addContext` | yes | yes | yes | yes |
| `session.start.context` | yes | yes | no dedicated hook | yes |
| `loop.continue` | yes, capped at 8 in a row | yes, host-bounded | no | experimental |
| `tool.inprocess` | mod only | no | yes | yes |
| `tool.mcp` | yes | yes | yes | yes |
| `ui.pane` | mod only | no | not read | yes |
| `ui.status` | mod only, terminal | no | not read | yes |
| `ui.toast` | mod only | text to the user only | not read | yes |
| `state.durable` | mod `$.store`, or a data directory | a data directory | no | yes |
| `runtime.longlived` | mod only | no | yes | yes |

"Mod only" means the flag is present only when the plugin opts into Claude Code's early-access mod API.

Worked features:

- **Guard.**
  Requires `tool.before.deny`; present everywhere, so no fallback is needed.
  A guard that answers `ask` declares a fallback for OpenCode 1.x: `deny` with the reason, or `else: fail-build`.
  A security guard declares `onFailure: block` and `else: fail-build`, never silently off.
- **Result redaction.**
  Requires `tool.after.rewrite`.
  Antigravity has no fallback that keeps the meaning; the nearest is a guard that denies the call up front, so the feature declares that or `fail-build`.
- **Pane.**
  Requires `ui.pane`.
  Claude Code: a `Pane` render hook opened with `$.ui.open`.
  OpenCode 2.x: the `session.panel` surface.
  Fallback everywhere else: a skill, which is also a slash command, that prints the same tree as text.
- **Status entry.**
  Requires `ui.status`.
  Claude Code: the `StatusLine` render component, terminal only.
  OpenCode 2.x: the `prompt.footer.status` slot.
  Antigravity: off with a build notice, since its status line is the user's own setting.
- **Custom tool.**
  Requires `tool.inprocess`, with the fallback `tool.mcp`: the same `execute` function served by a generated stdio MCP server.
- **Durable state.**
  Requires `state.durable`.
  The data-directory form is a JSON file under `CLAUDE_PLUGIN_DATA` or `PLUGIN_DATA`, written atomically by the hook process.
  OpenCode 1.x has no plugin storage, so the fallback is a file under the project's own state directory, or off.

### The per-harness adapter

An adapter is one object: `id`, `capabilities`, `toolVocabulary`, `decode(raw) → event`, `encode(answer) → raw`, and `emit(plugin) → files`.
`decode` and `encode` are pure, so recorded payloads test them without a harness.

- **Claude Code.**
  Emits `hooks/hooks.json` with command hooks under `hooks`, each running one bundled entry script.
  With the mod opt-in it also emits a `modules` entry in the same file, and guards move in-process as `tool.call` hooks with a `.catch`.
- **Antigravity CLI.**
  Emits `hooks.json` at the plugin root as one named hook holding matcher groups for `PreToolUse` and `PostToolUse` and flat handler lists for the other four events.
  The entry script reads camelCase JSON, always exits 0, and prints the decision.
  Prompt hooks ride on `PreInvocation`, which carries `lastUserInput`.
- **OpenCode 1.x.**
  Emits a `server()` function returning the hooks object: `tool.execute.before`, `tool.execute.after`, `chat.message`, `tool`.
- **OpenCode 2.x.**
  Emits `Plugin.define({ id, setup })` using `ctx.tool.hook`, `ctx.permission.hook`, `ctx.session.hook`, `ctx.tool.transform` and `ctx.storage`, and a `./tui` export for surfaces.

The two OpenCode adapters ship in one package, `{ ...Plugin.define({ id, setup }), async server() { … } }`, which OpenCode's migration guide supports from 1.18.29.

### Packaging

One build, one output directory per target:

| Target | Manifest | Hooks | MCP | Other |
|---|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` | `hooks/hooks.json`, keys `hooks` and `modules` | `.mcp.json` | `skills/`, `commands/`, `agents/`; a `marketplace.json` to install from |
| Antigravity CLI | `plugin.json`, schema-checked, `name` required | `hooks.json` at the root | `mcp_config.json`, remote key `serverUrl` | `skills/`, `agents/`, `rules/`, `scripts/`; no `commands/` |
| OpenCode | an npm package, default export | the package's own code | through the plugin's `ctx.mcp`, or config | `plugins` key on 2.x, `plugin` on 1.x; `./tui` export |

The hook entry scripts need a JavaScript runtime on the user's machine, which neither Claude Code nor Antigravity promises to a hook process.
The build should therefore be able to emit either a script for `node` or a compiled single-file executable per platform.

## Capability matrix

Each cell ends with its source.
(R) was read this session in a primary source: official docs, a published schema, or source code at a tag.
(S) comes from a secondary source: a third-party copy of Google's bundled documentation, or third-party code.
(I) is inferred.
The source keys are listed under [Sources](#sources).

| Capability | Claude Code | Antigravity CLI | OpenCode 1.x (v1.19.6) | OpenCode 2.x (v2.4.2) |
|---|---|---|---|---|
| Before a tool call: deny | `PreToolUse` exit 2 or `permissionDecision: "deny"` (R: H); mod `tool.call` returning `{ deny }` (R: dts:5304) | `PreToolUse` stdout `decision: "deny"`, `reason` (R: AH) | a throw in `tool.execute.before` rejects the call before `item.execute` (R: O1 `opencode/src/session/tools.ts` L102-L131, `opencode/src/plugin/index.ts` L284-L297) | `execute.before` fails with `Tool.Error`: "a Tool.Error rejects the call before the tool runs" (R: O2 `plugin/src/effect/tool.ts` L57-L61, `core/src/tool.ts` L161-L237) |
| Before a tool call: ask the user | `permissionDecision: "ask"`; also `defer` in non-interactive runs (R: H); mod `tool.check` returning `{ decision }` (R: dts:5325-5342) | `decision: "ask"`, and `force_ask` to ignore cached grants (R: AH; cache behaviour S: AX) | none: `permission.ask` is declared (R: O1 `plugin/src/index.ts` L261) and no call site was found (I) | `ctx.permission.hook` `evaluate` sets `effect` to allow, ask or deny; it does not run when configured rules already deny (R: O2 `core/src/permission.ts` L208-L240) |
| Before a tool call: rewrite input | `updatedInput`: "Replaces the entire input object" (R: H); mod `next({...e, …})`, checked against the tool's schema (R: dts:18615-18618) | `overwrite` (R: AH), a "shallow, top-level merge" (S: AX) | mutate `output.args` in place; reassigning it has no effect (R, I: O1 `opencode/src/session/tools.ts` L102-L111) | assign `event.input`, and `event.tool` to redirect the call (R: O2 `core/src/tool.ts` L161-L237) |
| After a tool call | `PostToolUse` cannot block; `updatedToolOutput` replaces what the model sees, `additionalContext` adds to it (R: H); mod `tool.run` returning `{ result }` (R: dts:5369-5396) | `PostToolUse` receives `toolCall`, `stepIdx`, `error` and returns `{}`: observe only (R: AH) | `tool.execute.after` mutates `title`, `output`, `metadata` of the result (R: O1 `opencode/src/session/tools.ts` L112-L129) | `execute.after` rewrites `result` or `error`; it cannot fail (R: O2 `core/src/tool.ts` L199-L223) |
| Prompt submitted | `UserPromptSubmit` blocks and adds context but "can't replace the prompt" (R: H); mod `prompt.submit` rewrites the text or returns `{ drop }` (R: dts:5605-5615) | no prompt event; `PreInvocation` gets `lastUserInput`, and answers `injectSteps` or `terminationBehavior: "terminate"` (R: AH) | `chat.message` mutates the message and its parts before it is saved; no clean rejection (R: O1 `opencode/src/session/prompt.ts` L999-L1009) | `ctx.session.hook` `prompt` mutates the prompt; its failure type is `never` (R: O2 `plugin/src/effect/session.ts` L207-L223) |
| Session start | `SessionStart`: `additionalContext`, cannot block (R: H); mod `session.start`, awaited before the first prompt (R: ref:262-264) | `SessionStart`: `injectSteps` with a `systemMessage`, cannot block (R: AH) | the generic `event` hook only, fire-and-forget (R: O1 `opencode/src/plugin/index.ts` L255-L262) | `ctx.event.subscribe()`, and `ctx.session.hook` `context` before each model request (R: O2 `plugin/src/promise/plugin.ts` L27-L62) |
| Stop, and keeping the loop going | `Stop` with `decision: "block"`, capped at 8 consecutive continuations (R: H); mod `turn.complete` annotates only (R: dts:6357-6368) | `Stop` stdout `decision: "continue"`; `PostInvocation` `terminationBehavior: "force_continue"` (R: AH); a host limit exists, reason `MAX_FORCED_INVOCATIONS`, its number unknown (S: AX) | nothing found that forces a continuation (R: O1 `plugin/src/index.ts` L222-L335) | `experimental.idle`: input the hook admits runs instead of idling; "A plugin that always admits input must bound its own loop" (R: O2 `plugin/src/promise/README.md` L126-L148) |
| Session end | `SessionEnd`, 1.5 s shared budget (R: H) | none (R: AH) | `dispose`, and the `event` hook (R: O1 `plugin/src/index.ts` L222-L335) | `ctx.event.subscribe()` (R: O2 `plugin/src/promise/plugin.ts` L27-L62) |
| Hook failure | exit codes other than 0 and 2 let the action proceed; `"onFailure": "block"` from 2.1.295 (R: H); a mod hook that throws or overruns is skipped unless it has `.catch` (R: dts:12974-12986) | not documented (R: AH); no exit-code channel (S: GF) | a throw vetoes the call (R: O1, as above) | only `execute.before` may fail; whether a plain `Error` becomes a `Tool.Error` was not read (R, I) |
| Hook timeout | 600 s default for commands, 30 s on `UserPromptSubmit` (R: H); mod: 10 s of the hook's own time, 20 min on tool events (R: dts:7142-7186) | `timeout` in seconds, default 30; what happens on overrun is not documented (R: AH) | none found (I) | none found (I) |
| Handler types | `command`, `http`, `mcp_tool`, `prompt`, `agent` (R: H); in-process module (R: ref:14) | `command` only (R: AH; "no HTTP or prompt hooks yet" S: AX) | in-process function (R: O1) | in-process function, Promise or Effect (R: O2 `plugin/package.json` L11-L29) |
| Custom tools | MCP server; mod `$.tool.register` with a JSON Schema, served from a `tool.run` hook, named `mcp__<plugin>__<name>` (R: dts:19463-19487) | MCP only (R: AB) | `tool({ description, args, execute })` with zod args (R: O1 `plugin/src/tool.ts` L45-L54); files under `tool/` or `tools/` (R: O1 `opencode/src/tool/registry.ts` L183-L196) | `ctx.tool.transform(editor => editor.add({ name, description, input, execute }))`, JSON Schema or Effect Schema input (R: O2 `plugin/src/effect/tool.ts` L8-L23) |
| MCP servers | `.mcp.json`, or `mcpServers` in `plugin.json` (R: PR) | `mcp_config.json`, root key `mcpServers`; `command` or `serverUrl`; no SSE (R: AM, AB) | config (not read) | `ctx.mcp.transform`; config key not read (R: O2) |
| UI surfaces | mod `ui.render` components `Pane`, `StatusLine`, `AbovePrompt`, `BesidePrompt`, `CodeBlock`, `ContextMenu` and others (R: dts:13013); elements Box, Text, Button, Input, Select, Link, Code, Markdown, Progress (R: dts:5142-5205); `$.ui.toast`, `$.ui.notify` (R: ref:268) | none for a plugin in the CLI; hooks can show Markdown or an error to the user through `injectSteps` (R: AH; S: AX); the status line is a user setting (R: ASL; plugin exclusion I) | a `./tui` export exists (R: O1 `plugin/package.json`); not read | slots `sidebar.content`, `prompt.footer.status`, `home.footer.status`, `session.composer.top` and others; `session.panel`, `ui.toast`, `ui.dialog`; Solid and OpenTUI JSX (R: O2 `services/site/src/docs/content/build/plugins/cli.mdx` L219-L270, L520-L623) |
| Persistent state | `CLAUDE_PLUGIN_DATA`, which survives updates (R: H, PR); mod `$.store`, JSON, 4 MiB, across sessions; `$.state`, versioned, per session (R: dts:4451-4517) | `PLUGIN_DATA`, "a persistent, writable data directory outside the workspace", path not stated (R: AB) | none in the plugin input (R: O1 `plugin/src/index.ts` L56-L66) | `ctx.storage` `get`, `set`, `remove`, `scan`, scoped by plugin id (R: O2 `plugin/src/storage.ts` L1-L17); terminal UI `storage.store` and `storage.memory` (R: O2 `cli.mdx`) |
| Instructions | no plugin context file; a skill, or mod `prompt.section` and `prompt.context` (R: dts:5675-5699) | plugin `rules/`, `AGENTS.md` recommended (R: AB, AR) | `AGENTS.md`; hook `experimental.chat.system.transform` (R: O1 `plugin/src/index.ts`) | `AGENTS.md`, with a project `CLAUDE.md` read when there is none (R: O2 `core/src/config/plugin/instruction.ts` L41-L43, OR) |
| Skills | `skills/<name>/SKILL.md` (R: PR) | `skills/<name>/SKILL.md`; extra paths through `plugin.json` `skills` (R: AB, ASch) | skill directories (not read at this tag) | `skill/` and `skills/` under config directories; also scans `.claude` and `.agents` roots (R: O2 `core/src/config/plugin/skill.ts` L91-L92, `compatibility.ts` L84; default-on I) |
| Slash commands | `commands/*.md`; mod `$.command.register` with a `command.run` hook (R: PR, dts:2454-2493) | none; each skill is a slash command, and imported commands become skills (R: AMig, AB) | `command/` and `commands/` Markdown; hook `command.execute.before` (R: O1 `opencode/src/session/prompt.ts` L1460) | `command/` and `commands/` Markdown (R: O2 `core/src/config/plugin/command.ts` L119-L124) |
| Subagents | `agents/*.md` (R: PR) | `agents/<name>.md`; frontmatter `name`, `description`, `tools`, `model`, `mainAgent`, `subagent`, `mcpServers`, `skills` and others (R: ASub) | agent Markdown (not read at this tag) | `agent/`, `agents/`, `mode/`, `modes/` Markdown (R: O2 `core/src/config/plugin/agent.ts` L23-L24) |
| Packaging and install | `.claude-plugin/plugin.json`, only `name` required; `claude plugin install <name>@<marketplace>`; `--plugin-dir`; `claude plugin validate` (R: PR, PM) | `plugin.json` or `.antigravity-plugin/plugin.json`, schema with `name` required and no extra keys (R: ASch, AB); `agy plugin install` from a directory, a marketplace or a GitHub URL; `agy plugin validate --strict` (R: AP, AMk) | config key `plugin`; npm package or local directory (R: O1 `plugin/src/index.ts` L70-L72) | config key `plugins`; npm name, version, directory or `file://` URL; `.opencode/plugins/`; `opencode plugin add` (R: O2 `services/site/src/docs/content/build/plugins/index.mdx` L59-L90) |
| Stability | command hooks: documented and versioned; mod API: "EARLY ACCESS: this surface may change between releases without notice" (R: dts:4) | hooks changed in 2026: `lastUserInput` added in 2.28.0, a permission bypass by hooks fixed in 2.33.0 (R: AC) | the older line; GitHub's "latest" release (R: GitHub releases API) | the newer line by date; the terminal UI doc carries no experimental label, the GUI part does (R: O2) |

### Antigravity's hook contract

- **Six events:** `SessionStart`, `PreToolUse`, `PostToolUse`, `PreInvocation`, `PostInvocation`, `Stop`; no prompt, session-end, subagent, compaction or notification event. (R: AH)
- **Common stdin,** camelCase: `conversationId`, `workspacePaths`, `transcriptPath`, `artifactDirectoryPath`, `modelName`, `lastUserInput`. (R: AH)
  A secondary copy adds `hookEventName` and `cwd`. (S: AX)
- **`PreToolUse`:** stdin adds `toolCall` (`id`, `name`, `args`) and `stepIdx`; stdout `decision`, `reason`, `overwrite`, `permissionOverrides`. (R: AH)
  `permissionOverrides` is a list of resource strings such as `command(npm test)`, granted temporarily. (S: AX)
- **`PreInvocation` and `PostInvocation`:** stdin adds `invocationNum` and `initialNumSteps`; stdout `injectSteps` and `terminationBehavior`. (R: AH)
  Injected steps are `systemMessage`, `userMessage` or `errorMessage`; an `errorMessage` is shown to the user and not sent to the model. (R: AH)
- **`Stop`:** stdin `executionNum`, `terminationReason`, `error`, `fullyIdle`; stdout `decision: "continue"` with a `reason`. (R: AH)
  `continuationActive` on stdin says a Stop hook already kept this turn going. (S: AX)
- **`hooks.json`:** top-level keys are hook names, each holding event arrays and an optional `"enabled": false`. (R: AH)
  Tool events take groups of `{ matcher, hooks }`, the matcher a regular expression over the tool name; the other events take a flat handler list. (R: AB)
  A handler is `{ type: "command", command, timeout }`. (R: AH)
- **Where hooks load from:** a plugin's `hooks.json`, a workspace `.agents/hooks.json`, or the user's global `hooks.json`; a `hooks` key in `settings.json` is kept and never loaded. (R: AH, AS)
- **Environment:** `PLUGIN_ROOT` and `PLUGIN_DATA` are documented for MCP servers (R: AB) and reported for hooks (S: AX).

### OpenCode's two lines

- The repository is `anomalyco/opencode`; `sst/opencode` redirects to it. (R)
- GitHub's latest-release flag points at `v1.19.6`, published 2026-11-17; `v2.4.2` was published 2026-11-26 and is not a prerelease. (R: GitHub releases API)
- 1.x loads a function or `{ server }` from `@opencode-ai/plugin`; 2.x loads only a default export of `{ id, setup }` or `{ id, effect }` from `@opencode/plugin`. (R: O1 `opencode/src/plugin/index.ts` L88-L125; O2 `core/src/plugin/module.ts` L81-L94)
- "V1 plugin implementations do not run in V2." (R: O2 `services/site/src/docs/content/build/plugins/migrate-v1.mdx` L10-L12)
- The `v2/` folder inside 1.19.6 is an early subset with no tool, permission, session or storage hooks, so 2.x features cannot be reached from 1.x. (R: O1 `plugin/src/v2/effect/context.ts` L12-L22)
- No removal date for 1.x was found; the guide says to "Remove the V1 implementation after your support window ends."

### Claude Code's two mechanisms

- Command hooks and a mod ship in one plugin, in the same `hooks/hooks.json`: settings hooks under `hooks`, the module path under `modules`. (R: MR, ref:14)
- The command-hook event list is long: 30 or more events, of which `PreToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `PreCompact`, `PostToolBatch` and a dozen others can block. (R: H)
- A mod registers with `on(pattern, matcher?, hook)`, each hook `($, e, next)`; a mod naming an unknown event is refused whole, so `engines["claude-code"]` in `plugin.json` should pin the range. (R: ref:19-22, dts:9034-9039)
- A mod reaches the outside only through `$`: `$.process.run`, `$.http.fetch`, `$.fs`, `$.clock`, `$.store`, `$.state`, `$.ui`, `$.model.complete`. (R: dts:3004-4900)
- A mod's `classic.<Event>` hooks let it answer the command-hook events in-process. (R: ref:27)

## Tool vocabulary

A guard matches on a canonical operation, and each adapter maps it to its harness's tool name and argument keys.
OpenCode's two lines differ from each other, not only from the rest.

| Operation | Claude Code | Antigravity CLI | OpenCode 1.x | OpenCode 2.x |
|---|---|---|---|---|
| Shell | `Bash`: `command`, `timeout`, `run_in_background` | `run_command`: `CommandLine`, `Cwd` | `bash`: `command`, `timeout`, `workdir` | `shell`: `command`, `workdir`, `timeout`, `background` |
| Read a file | `Read`: `file_path`, `offset`, `limit` | `view_file`: `AbsolutePath`, `StartLine`, `EndLine` | `read`: `filePath`, `offset`, `limit` | `read`: `path`, `offset`, `limit` |
| Edit a file | `Edit`: `file_path`, `old_string`, `new_string`, `replace_all` | `replace_file_content`: `TargetFile`, `TargetContent`, `ReplacementContent`, `AllowMultiple`; `multi_replace_file_content`: `TargetFile`, `ReplacementChunks` | `edit`: `filePath`, `oldString`, `newString`, `replaceAll`; `apply_patch`: `patchText` | `edit`: `path`, `oldString`, `newString`, `replaceAll`; `patch`: `patchText` |
| Write a file | `Write`: `file_path`, `content` | `write_to_file`: `TargetFile`, `CodeContent`, `Overwrite` | `write`: `filePath`, `content` | `write`: `path`, `content` |
| Search contents | `Grep`: `pattern`, `path`, `glob` | `grep_search`: `SearchPath`, `Query`, `IsRegex`, `Includes` | `grep`: `pattern`, `path`, `include` | `grep`: `pattern`, `path`, `include`, `literal` |
| Find files | `Glob`: `pattern`, `path` | `find_by_name`: `SearchDirectory`, `Pattern`; `list_dir`: `DirectoryPath` | `glob`: `pattern`, `path` | `glob`: `pattern`, `path` |
| Fetch a URL | `WebFetch`: `url`, `prompt` | `read_url_content`: `Url` | `webfetch`: `url`, `format`, `timeout` | `webfetch`: `url`, `format`, `timeout` |
| Search the web | `WebSearch`: `query` | `search_web`: `query`, `domain` | `websearch`: `query` | `websearch`: `query` |
| Subagent | `Agent`: `prompt`, `description`, `subagent_type` | `invoke_subagent`: `Subagents[]` of `Prompt`, `Role`, `TypeName` | `task`: `prompt`, `description`, `subagent_type` | `subagent`: `agent`, `prompt`, `description` |
| MCP tool | `mcp__<server>__<tool>` | `mcp_<server>_<tool>` when loaded eagerly; otherwise `call_mcp_tool` with `ServerName`, `ToolName`, `Arguments` | by tool name | by tool name |

Sources, all (R) except where noted:

- Claude Code: the mod type declarations' tool-input table (dts:23559-24454); `Glob` and `Grep` from H, and the tools reference says both are absent by default on macOS, Linux and WSL, where search goes through `Bash`.
- Antigravity: AH for names and argument keys, AM for MCP naming; that a lazily loaded MCP call matches as `call_mcp_tool` is (I).
- OpenCode 1.x: `packages/opencode/src/tool/` at `v1.19.6`: `shell/prompt.ts` L16-L22, `read.ts` L28-L33, `edit.ts` L47-L56, `write.ts` L21-L22, `apply_patch.ts` L19, `grep.ts` L11-L15, `glob.ts` L11-L12, `task.ts` L44-L58.
  Models named `gpt-*` get `apply_patch` in place of `edit` and `write` (`registry.ts` L297-L300).
- OpenCode 2.x: `packages/core/src/tool/plugin/` at `v2.4.2`: `shell.ts` L29, `read.ts` L21-L30, `edit.ts` L21-L29, `write.ts` L17-L24, `patch.ts` L22-L25, `grep.ts` L15-L38, `glob.ts` L16-L29, `webfetch.ts` L13-L29, `subagent.ts` L17-L41.

Three consequences for guards:

- A file-write guard must cover every edit form: `Edit` and `Write`, three Antigravity tools, and OpenCode's patch tool, whose only argument is a patch text that has to be parsed for paths.
- A shell guard on Claude Code also sees file searches, where `Grep` and `Glob` are absent.
- An MCP guard on Antigravity must read `call_mcp_tool`'s arguments to learn the real tool.

## Build, fork or depend

**Build fresh.**

agent-plugin-sdk, <https://github.com/jal-co/agent-plugin-sdk>, read at commit `63cee1a`:

- **What it is:** a `definePlugin` object compiled to eight harnesses' files: Claude Code, Codex, Pi, OpenCode, Gemini CLI, Copilot, Cursor and Windsurf. (R: `packages/agent-plugin-sdk/src/harnesses/index.ts`)
- **Its hook** is `{ event, matcher?, command, timeout? }`, a shell-command string; only event names are translated, and payloads, tool names and decisions are not. (R: `src/types.ts`, `src/harnesses/hooks.ts`)
- **On OpenCode,** `hooks` is unsupported and nothing is emitted. (R: `src/harnesses/opencode.ts`)
- **On Claude Code,** it emits command hooks only, with no use of the mod API. (R: `src/harnesses/claude.ts`)
- **No Antigravity target;** its Gemini CLI target emits `gemini-extension.json` and a Claude-style `hooks/hooks.json`, which is not Antigravity's contract. (R: `src/harnesses/gemini.ts`; mismatch I)
- **Degradation** is one boolean per feature per harness; an unsupported feature is stripped with a warning, never an error, with no fallbacks. (R: `src/emit.ts`, `src/warnings.ts`)
- **No UI and no state primitive,** and the feature set is closed: a new primitive means editing the core. (R: `src/types.ts`)
- **Activity:** 16 stars, 40 commits, 3 of them after 2026-07-02, latest npm release `@jalco/ap-sdk` 0.4.1 on 2026-08-24. (R: GitHub and npm registry APIs)

Depending on it would add nothing this project needs at runtime, and forking it would mean replacing its `Hook` type, its `Feature` set and its degradation model, which is most of its core.

What to borrow, as designs:

- **From agent-plugin-sdk (MIT):** the `Harness` registry with `registerHarness`, the pure `emit(plugin, ctx) → files` translator, and structured build warnings.
- **From agenthooks,** <https://github.com/speakeasy-api/agenthooks> (Go, MIT): a normalised hook event set with decisions Allow, Deny, AskUser, updated input and replaced output across nine harnesses, Antigravity CLI and OpenCode among them. (R: its package documentation, in part)
- **From Apache Magpie's agent-guard,** <https://magpie.apache.org/docs/tools/agent-guard/readme> (Apache-2.0): one dispatch core with a thin adapter per harness; it denies on OpenCode by throwing in `tool.execute.before` and on Antigravity through `PreToolUse`, which agrees with the source read here. (R: its documentation)
- **From wshobson/agents,** <https://github.com/wshobson/agents> (MIT): per-harness capability booleans and tool-name maps in `tools/adapters/capabilities.py`, including an Antigravity adapter that emits no hooks. (R)

## Prior art

| Project | From and to | Runtime hooks | Degradation model | Read |
|---|---|---|---|---|
| [Agent Skills spec](https://agentskills.io/specification) | `SKILL.md`, read by all four targets | none | none | spec |
| [agent-plugin-sdk](https://github.com/jal-co/agent-plugin-sdk) | TypeScript definition to 8 harnesses | command strings, names only | a boolean and a warning | source |
| [wshobson/agents](https://github.com/wshobson/agents) | Claude Code Markdown to 7 harnesses, Antigravity among them | none | capability booleans, warnings | two files |
| [agenthooks](https://github.com/speakeasy-api/agenthooks) | Go hook handlers for 9 harnesses | yes, normalised decisions | not seen | package docs |
| [Apache Magpie agent-guard](https://magpie.apache.org/docs/tools/agent-guard/readme) | one Python core, an adapter per harness | deny only, shell commands | none | docs |
| [agent-harness-adapter](https://github.com/six5536/agent-harness-adapter) | Rust hook module for 8 or more harnesses | yes; "This crate is unstable" | not seen | summary |
| [weykon/agent-hooks](https://github.com/weykon/agent-hooks) | hook events bridged to JSON lines | observe only | none | summary |
| [rulesync](https://github.com/dyoshikawa/rulesync) | rules, MCP, commands, skills, hooks to many tools; has an `antigravity-plugin` target | files only | a support table | README |
| [compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin) | Claude Code plugin to other formats | not mentioned | none | README |
| [ruler](https://github.com/intellectronica/ruler), [ai-rulez](https://github.com/Goldziher/ai-rulez) | rules to many agents | not seen | not seen | metadata |
| [openskills](https://github.com/numman-ali/openskills), [vercel-labs/skills](https://github.com/vercel-labs/skills) | skill installers | none | none | metadata |
| `agy plugin import claude` | Claude Code plugins to Antigravity plugins | not documented | not documented | AP |

No project found converts hook handlers across Claude Code, Antigravity CLI and OpenCode in TypeScript, none has declared fallbacks, and none has a UI primitive.
No cross-harness standard for hooks was found; the Agent Skills spec covers `SKILL.md` alone.

## Complexity

| Piece | Size | Why |
|---|---|---|
| Declarative emitters, 3 output layouts | small | file generation; the layouts are close, and the manifests have published schemas or references |
| Event and decision codecs, 4 targets | medium | the contracts are now known for all four; the work is the fixtures and the edge cases |
| Tool vocabulary | medium, and permanent | 10 operations by 4 targets, with patch tools and lazy MCP calls as special cases |
| Guard, prompt and stop hooks end to end | medium | bundling, the per-call entry script, failure policy per target |
| Custom tools, in-process with an MCP fallback | medium | three in-process forms and one generated MCP server |
| Durable state | small to medium | two real stores and a file fallback |
| UI: pane, status, toast | large | two unrelated renderers, one of them early access, and a text fallback |
| Capability flags, fallbacks and the build report | small | a table and a resolver |

The useful first cut is everything but UI: medium overall.
The estimate fell for hooks, since every target's veto, rewrite and failure behaviour is now read from docs or source, and rose for OpenCode, which is two targets.

## Risks still open

- **Antigravity's undocumented edges:** what a non-zero exit, a timeout, or invalid or empty stdout does; how a hook's `decision` combines with the user's permission rules; the number behind the forced-continuation limit.
- **Antigravity's `decision` values:** the hooks page lists six, with `auto_approve` and `deny_unless_prior_grant`; a secondary copy of the bundled docs lists four, and third-party code says the host does not define `deny_unless_prior_grant`.
- **Antigravity hook environment:** whether hooks receive `PLUGIN_ROOT` and `PLUGIN_DATA`, the data directory's path, and whether workspace hooks need a trusted workspace.
- **`agy plugin import claude`:** what it does with Claude Code hooks is not documented.
- **OpenCode's default line:** which of 1.x and 2.x a new install gets, and how long 1.x is supported.
- **OpenCode 1.x `permission.ask`:** no call site was found, in a search that was not exhaustive.
- **OpenCode error text:** what the model sees after a veto on either line, and whether a plain `Error` vetoes on 2.x.
- **OpenCode 1.x terminal UI:** its `./tui` export was not read, so 1.x UI flags are unknown.
- **Claude Code's mod API:** early access; it may move under the adapter, and whether a mod can import bundled npm code beyond its listed file extensions is not stated.
- **A JavaScript runtime for hook scripts** on machines where neither harness provides one.
- **Per-call cost:** a process start on every tool call under command hooks, unmeasured.
- **Whether a shared UI tree earns its cost** for two renderers.

## Next step

Prototype one guard through all four targets, before any other feature:

1. Write the guard once: deny a shell command matching a pattern, ask before a file write outside the workspace, and rewrite one argument.
2. Record real payloads from each harness as fixtures, and write `decode` and `encode` for each target against them.
3. Emit the four outputs, and validate them with `claude plugin validate`, `agy plugin validate --strict` and a load under each OpenCode line.
4. Run each, and record what the unknowns above turn out to be: Antigravity's exit, timeout and bad-output behaviour first, since the fail-closed wrapper depends on it.
5. Run `agy plugin import claude` on the Claude Code output and compare it with the hand-emitted Antigravity output.

The pane is the second prototype, on Claude Code's mod API and OpenCode 2.x, with the text fallback on the other two.

## Sources

Keys used above:

- **H:** <https://code.claude.com/docs/en/hooks>
- **PR:** <https://code.claude.com/docs/en/plugins-reference>
- **PM:** <https://code.claude.com/docs/en/plugin-marketplaces>
- **MR:** <https://code.claude.com/docs/en/plugins/mods/reference.md>
- **ref:N, dts:N:** line N of `reference.md` and of `types/claude-code.d.ts` in the plugin-authoring reference bundled with Claude Code 2.1.332.
- Also read for Claude Code: <https://code.claude.com/docs/en/tools-reference>, <https://code.claude.com/docs/en/statusline> and <https://code.claude.com/docs/en/plugins>.
- **AH:** <https://antigravity.google/docs/hooks>
- **AB:** <https://antigravity.google/docs/build-plugins>
- **AP:** <https://antigravity.google/docs/plugins?tab=cli>
- **AM:** <https://antigravity.google/docs/mcp>
- **AS:** <https://antigravity.google/docs/settings>
- **AR:** <https://antigravity.google/docs/rules>
- **ASub:** <https://antigravity.google/docs/subagents>
- **ASL:** <https://antigravity.google/docs/cli/statusline>
- **AMig:** <https://antigravity.google/docs/cli/gcli-migration>
- **AMk:** <https://antigravity.google/docs/marketplace>
- **AC:** <https://antigravity.google/docs/changelog>
- **ASch:** <https://antigravity.google/schemas/v1/plugin.json>
- Also read for Antigravity: <https://antigravity.google/docs/permissions>, <https://antigravity.google/docs/skills>, <https://antigravity.google/docs/cli/reference>, <https://antigravity.google/docs/sandbox>, <https://antigravity.google/docs/slash-commands>, <https://antigravity.google/docs/faq> and <https://github.com/google-antigravity/antigravity-cli>.
- **AX,** secondary: a third-party copy of the documentation skill bundled with Antigravity, <https://github.com/asgeirtj/system_prompts_leaks/tree/main/Google/antigravity/skills/agy-customizations/docs>.
  It agrees with the official hooks page wherever both speak.
- **GF,** secondary: third-party adapter code, `packages/agent-hooks/src/agents/antigravity/outputs.ts` in <https://github.com/goodfoot-io/marketplace>.
- **O1:** <https://github.com/anomalyco/opencode/tree/v1.19.6>, paths given under `packages/`.
- **O2:** <https://github.com/anomalyco/opencode/tree/v2.4.2>, paths given under `packages/` unless they start with `services/`.
- **OR:** <https://opencode.ai/docs/rules>; also read, <https://opencode.ai/docs/build/plugins/server> and <https://opencode.ai/docs/custom-tools>.
- Gemini CLI's retirement: <https://developers.googleblog.com/en/an-important-update-transitioning-gemini-cli-to-antigravity-cli/>, dated 2026-05-19.
  Consumer tiers stop on 2026-06-18; Code Assist Standard and Enterprise licences and paid API keys keep access, with no end date given.

## Not verified

- Nothing was run: every behaviour here is from documentation or source, not observation.
- Web pages came through a summarising fetch, so Antigravity field names should be rechecked against a live hook's stdin before they are relied on.
  The `plugin.json` schema was read through the same fetch, not as raw JSON.
- No published schema for Antigravity's `hooks.json` or `mcp_config.json` was found, and no official example plugin with hooks.
- Antigravity's changelog was read through its first fifth, and its `mcp-apps` page not at all.
- OpenCode source was read at tags, not commit SHAs, and two call sites were rechecked directly: `tool.execute.before` in 1.19.6 and the permission `evaluate` hook in 2.4.2.
- OpenCode's 1.x terminal UI export, its MCP config key, the arguments of `opencode plugin add`, and whether `.agents/skills` is scanned by default were not read.
- Claude Code's mod type declarations run to about 29,000 lines and were read in the parts cited; the input types of most events and the options of `$.ui.open` were not.
- Claude Code's mods reference page was read through its first sections, and the page on the order mods run in was not read.
- agent-plugin-sdk was read without line numbers, and its capability flags for five of its eight targets were not opened.
- agenthooks, agent-guard and the Rust hook libraries were read as documentation, not source.
- A manifest standard named Agent Plugins 1.0.0 surfaced in search results only and was not opened; it reportedly leaves hooks out.
