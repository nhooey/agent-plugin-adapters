# agent-plugin-adapters

A coding-agent plugin written once, against a harness-agnostic API, with adapters for Claude Code, Gemini CLI and OpenCode.

Each harness offers a different set of extension points: hooks, tools, slash commands, panes, subagents.
An adapter maps the plugin's features onto its harness and degrades each one by what that harness can do, rather than limiting every harness to the features they all share.

There is no code yet.
The repository holds the research the design rests on, in `agents/reports/`, and the plans built on it, in [`agents/plans/`](agents/plans/README.md).

## Development

`nix develop` enters the dev shell, which installs the project's agent skills.
`nix fmt` formats the repository and `nix flake check` checks it.
