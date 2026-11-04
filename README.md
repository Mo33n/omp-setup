# Hire a local coding crew. The contract is copy-paste.

One terminal. Your own models. A harness that treats coding like a team sport: a coder, a reviewer, a language server, a debugger, and a few specialists who know when to stay in their lane.

This repo is a copy-paste setup for **[omp](https://github.com/can1357/oh-my-pi)** (Oh My Pi), the MIT-licensed, bring-your-own-model coding agent. The full guide lives in **[setup.md](setup.md)**. It matches the current omp settings schema, CLI, and file formats, and every chat role in it runs on a model you pull and run locally.

Hosted APIs stay off until you opt in. Your machine does the thinking.

## Why this setup

A single model with a long prompt is a talented soloist. This setup is the band.

The same session can rename through a language server, stop a real process in a debugger, reject a patch whose anchors have gone stale, take a second model's review, check a pull request out into a worktree, read current library docs, and talk to the tracker you already use. A hook blocks `git push --force`. The model does not get to negotiate that one.

| You want | How this setup does it |
| --- | --- |
| Edits that understand the code | LSP rename, diagnostics, and code actions. Hashline rejects a patch when the file has moved on. |
| Runtime truth | DAP breakpoints, stacks, and variables. The agent watches the process. |
| A second opinion | An advisor reviews primary turns. On a machine that can hold two local models, the advisor is a different family from the coder. |
| Parallel work | Tasks fan out into git worktrees. The parent session integrates. |
| GitHub in the same session | Native GitHub through your `gh` login. Pull-request checkout is a worktree. |
| Docs that are actually current | Context7 when the task needs library docs. DeepWiki for how a public repository is built. |
| A real browser | omp's browser for public pages. Browser Relay for a logged-in Chrome tab. |
| Memory that survives the chat | Project memory and autolearn, so the next session starts wiser. |
| Weights that never leave the building | Ollama, llama.cpp, or LM Studio. 32 GB is the comfortable floor. 16 GB runs the smaller stack. |

Use a native omp tool when one exists. Add an MCP server only for a system the native tools do not already speak. Extra servers ride along on every turn, including subagent turns. A longer list is a heavier backpack.

## Five minutes to oriented

```sh
curl -fsSL https://omp.sh/install | sh
omp --version
```

Homebrew, Bun, Nix, and Windows installs are in [setup.md §4](setup.md#4-install).

Then open **[setup.md](setup.md)** and walk it in order. Section 22 is a master prompt you can hand to an assistant that can edit files and run commands on the machine where omp is installed. It applies the config and stops to verify after each block.

Before you write a single role, run:

```sh
omp models ollama
```

Copy the selector that command prints. An id omp cannot see stays unresolved. The guide does not invent model names, and neither should the setup.

## What you will configure

The interesting work is credentials, role assignment, the advisor, language servers, debug adapters, GitHub, search, Context7, DeepWiki, memory, and the per-repo connectors you actually operate. Strong defaults stay where omp already put them.

| Piece | Where it lands |
| --- | --- |
| Roles, advisor, memory, approvals | `~/.omp/agent/config.yml` |
| Personal MCP servers | `~/.omp/agent/mcp.json` |
| Specialists for every repo | `~/.omp/agent/agents/*.md` |
| One repository's overrides | `<repo>/.omp/config.yml` |
| Repo rules the agent loads | `<repo>/AGENTS.md` |

A complete global config, a directory map, a verification checklist, and a troubleshooting table are all in the guide. After the first week, `omp stats --summary` tells you whether the advisor should review every turn or only at the end.

## The guide, section by section

1. [Purpose](setup.md#1-purpose)
2. [Leave these defaults alone](setup.md#2-leave-these-defaults-alone)
3. [Prerequisites](setup.md#3-prerequisites)
4. [Install](setup.md#4-install)
5. [Credentials and config layout](setup.md#5-credentials-and-config-layout)
6. [Model routing](setup.md#6-model-routing)
7. [Advisor](setup.md#7-advisor)
8. [Language servers](setup.md#8-language-servers)
9. [Debugger](setup.md#9-debugger)
10. [Editing, approvals, and parallel safety](setup.md#10-editing-approvals-and-parallel-safety)
11. [Subagents](setup.md#11-subagents)
12. [Connectors](setup.md#12-connectors)
13. [Memory and autolearn](setup.md#13-memory-and-autolearn)
14. [Skills and project instructions](setup.md#14-skills-and-project-instructions)
15. [Compaction, context, and spend](setup.md#15-compaction-context-and-spend)
16. [Always-on coverage](setup.md#16-always-on-coverage)
17. [First session](setup.md#17-first-session)
18. [Verification checklist](setup.md#18-verification-checklist)
19. [Troubleshooting](setup.md#19-troubleshooting)
20. [Complete global config](setup.md#20-complete-global-config)
21. [Directory layout](setup.md#21-directory-layout)
22. [Master setup prompt](setup.md#22-master-setup-prompt)
23. [After the first week](setup.md#23-after-the-first-week)

## Official omp docs

This guide follows upstream. When a key and this file disagree, `omp config list` on the build you installed wins.

- [Settings](https://omp.sh/docs/settings)
- [Models](https://github.com/can1357/oh-my-pi/blob/main/docs/models.md)
- [Code intelligence](https://omp.sh/docs/code-intelligence)
- [Debugging](https://omp.sh/docs/debugging)
- [Subagents](https://omp.sh/docs/subagents)
- [MCP](https://omp.sh/docs/mcp)
- [GitHub](https://omp.sh/docs/github)
- [CLI](https://omp.sh/docs/cli)

## Share it forward

Fork it, trim it to the languages you actually ship, and send back the bit that saved you an afternoon. The advantage is the stack running together: language-server edits, a debugger, hashline, an advisor, worktree subagents, `gh`, current docs, and a hook the model cannot override.

## License

[MIT](LICENSE). omp itself is MIT-licensed. This guide is a setup on top of it, shared the same way.
