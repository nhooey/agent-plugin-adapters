# agent-plugin-adapters

A coding-agent plugin written once, against a harness-agnostic API, with an adapter per harness: Claude Code, Antigravity CLI and OpenCode.
Each adapter degrades a feature by what its harness can do, such as a pane drawn as plain text where a harness has no panes, rather than cutting the plugin down to the features every harness shares.
See `README.md`.

The repository holds research and plans only; there is no code yet.

## Layout

- `agents/reports/`: dated research and audits, one Markdown file each, named `<YYYY-MM-DD>-<slug>.md`, with `state: current` or `state: superseded` and `as-of:` in their frontmatter, as `coding-agent-plans-as-graph` lays out.
  A newer finding is a new report that supersedes the old one, never an edit to it.
- `agents/plans/`: the plan graph, one node per file, open nodes in `open/` and closed ones in `closed/`, a plan of several nodes in a folder of its own.
  `agents/plans/README.md` is the overview, written by the skill's `plans.py overview`; edit the nodes, never the overview.
  A node links to the report that justifies it.
- `nix/skills-devshell/`: the sub-flake that installs this repository's skills into `.agents/skills/`: skillspkgs' `claude-code-tools` combination, and `design-hygiene-find-implicit-models`.
- `.agents/skills/*.preferences.md`: the project's preferences for the skills of the same name, committed; the skills beside them are the dev shell's links into the Nix store, ignored by git.

## Build and check

Enter the dev shell with `nix develop` (or direnv); it installs the project's skills into `.agents/skills/`.
Then:

- `fmt`: `nix fmt`, treefmt with nixfmt.
- `check`: `nix flake check`, formatting included.
- `plans.py check` and `plans.py overview`, from `coding-agent-plans-as-graph`, after every change to `agents/plans/`; commit the overview with the nodes.

`menu` lists the dev shell's other commands.

## Writing

Reports and plans are Markdown with one sentence per line.
A report names no local path and no private repository or configuration: it says what a thing is instead.

## Pushing

Changes land through pull requests against `master`, as `.agents/skills/git-workflow-push-mode.preferences.md` says.
