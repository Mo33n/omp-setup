# Hire a local coding crew. The contract is copy-paste.

One terminal. Your own models. A harness that treats coding like a team sport: a coder, a reviewer, a language server, a debugger, and a few specialists who know when to stay in their lane.

This repo is a copy-paste setup for **[omp](https://github.com/can1357/oh-my-pi)** (Oh My Pi), the MIT-licensed, bring-your-own-model coding agent. The full guide lives in **[setup.md](setup.md)**. It matches the current omp settings schema, CLI, and file formats, and every chat role in it runs on a model you pull and run locally.

Hosted chat stays off until you opt in.

## Purpose

Claude Code is a harness: the loop, the tools, the skills, the hooks, and the subagents around Anthropic's model. omp is a harness too. The difference is that omp's harness is yours to change. Each role, tool, hook, and skill is a setting, and the model in each seat is one of those settings. Swap the coder and the language server, the debugger, the advisor, and the subagents stay. That is the customization. In Claude Code the model in that seat stays Claude.

The power is the loop those settings build. The agent calls the language server and the debugger, a stale edit is rejected, and a second model reviews from its own context. A local coder is not automatically a stronger Claude. The same weights get more of the machine.

Claude Code and Cursor already have subagents, skills, hooks, and a model picker. Claude can also isolate a subagent in a worktree and remember a repo between sessions. Use omp for who the model is, where the weights run, and which tools the agent can call.

**Local weights, any family.** Claude's picker is Claude. Cursor can switch vendors, and the turn still goes to a hosted model. Chat roles here run on Ollama, llama.cpp, or LM Studio. When Qwen and Devstral both stay resident, Qwen codes and Devstral reviews. When one model fits, the advisor is that model and reviews once, at the end. A frontier model is one role, opt-in. Change the weights and the language server, debugger, hooks, and subagents stay.

**A second context, already seated.** Claude Code can spawn a reviewer subagent. Since 2.1.300, `/advisor` can also seat another Claude model for the session. You pick it. If you pick none, none runs. omp's advisor reads the primary turn from its own context, and it can be a different family. A concern or a blocker wakes the coder. The coder can answer the note. A hook is the rule with no appeal. This guide refuses `git push --force`.

**The agent calls the IDE.** With an LSP plugin, Claude Code can read diagnostics and navigate. A rename in omp goes through the language server, so re-exports move with the symbol. A crash is a stopped debug session, once that adapter is installed. The Claude Code changelog shows no debugger tool. Hashline rejects a patch whose anchors no longer match. Claude's edit is a text replacement. Since 2.1.208 it still applies after the file changes when the old text matches the current file exactly and once, and Claude Code can read the file without prompting. On [Can Bölük's edit-format check](https://blog.can.ac/2026/02/12/the-harness-problem/), Grok Code Fast 1 went from 6.7% on patch to 68.3% on the original hashline, same weights. That post's chart lists the model at +64.6 points over patch, and a later hashline revision improves 12 of 16 models further. A [time-traveling stream rule](https://omp.sh/docs/ttsr) matches the live stream and can abort it mid-stream, then generate again. The same agent runs in the terminal or, over [ACP](https://omp.sh/docs/acp), in Zed.

Skills, rules, and MCP servers from Claude, Cursor, Codex, Cline, and Copilot load as they are. Claude's saved memory stays in Claude.

| | Claude Code | Cursor | This setup |
| --- | --- | --- | --- |
| Models | Claude models | Many vendors, hosted | Any family. Localhost until you opt in |
| Where chat runs | Anthropic, or your cloud account | The model provider | Weights on this machine |
| Review | A subagent, or `/advisor` if you turn it on. Still Claude | The agent, in the editor | An advisor on its own context. A different family when it fits |
| Edits | Text replacement. Since 2.1.208, an exact unique match of the current file still applies | Text, in the IDE | Hashline. Stale anchors rejected |
| Language server, debugger | LSP plugin: diagnostics and navigation. No debugger tool | The editor has them | The agent renames through LSP and drives DAP |
| Parallel work | Worktree subagents | Agents and worktrees | Worktrees, typed results, the parent integrates |

Search, GitHub, and doc lookups still leave the machine. The weights do not. What carries over from Claude and Cursor is [§1](setup.md#1-purpose).

## Why this setup

One session can rename through a language server, stop a process in a debugger, reject a stale patch, take a second model's review, check a pull request out into a worktree, read current docs, and talk to the tracker you already use. A hook blocks `git push --force` ([§10](setup.md#force-push-hook)).

| You want | How this setup does it |
| --- | --- |
| Edits that understand the code | LSP rename, diagnostics, and code actions ([§8](setup.md#8-language-servers)). Hashline rejects a stale patch ([§10](setup.md#10-editing-approvals-and-parallel-safety)). |
| Runtime truth | DAP breakpoints, stacks, and variables ([§9](setup.md#9-debugger)). |
| A second opinion | An advisor reviews primary turns ([§7](setup.md#7-advisor)). A second local family only when both stay resident ([§6](setup.md#default-local-stack)). |
| Parallel work | Tasks fan out into git worktrees and return typed results ([§11](setup.md#11-subagents)). |
| GitHub in the same session | Native GitHub through your `gh` login. Pull-request checkout is a worktree ([§12](setup.md#github-native)). |
| Docs that are actually current | Context7 for library docs. DeepWiki for how a public repository is built ([§12](setup.md#context7-every-repo)). |
| A real browser | omp's browser for public pages. Browser Relay for a logged-in Chrome tab ([§12](setup.md#browser-native)). |
| Memory that survives the chat | Project memory and autolearn ([§13](setup.md#13-memory-and-autolearn)). |
| Weights that never leave the building | Ollama, llama.cpp, or LM Studio ([§6](setup.md#get-the-loop-running)). 32 GB is the comfortable floor. 16 GB runs the smaller stack. |

Use a native omp tool when one exists. Add an MCP server only for a system the native tools do not already speak. Extra servers ride along on every turn, including subagent turns. A longer list is a heavier backpack.

## Five minutes

```sh
curl -fsSL https://omp.sh/install | sh
omp --version
```

Homebrew, Bun, Nix, and Windows installs are in [§4](setup.md#4-install).

Then open **[setup.md](setup.md)** and walk it in order. The local loop starts at [Get the loop running](setup.md#get-the-loop-running). [§22](setup.md#22-master-setup-prompt) is a master prompt you can hand to an assistant that can edit files and run commands on the machine where omp is installed. It applies the config and stops to verify after each block.

Before you write a single role, run:

```sh
omp models ollama
```

Copy the selector that command prints. An id omp cannot see stays unresolved. The guide does not invent model names, and neither should the setup. The loop is real when the [tool-call probe](setup.md#the-model-has-to-call-a-tool) writes `hello.txt`, and a task in a repo changes a file ([§17](setup.md#17-first-session)). A reply that describes the edit does not count.

## What you will configure

The interesting work is credentials, role assignment, the advisor, language servers, debug adapters, GitHub, search, Context7, DeepWiki, memory, and the per-repo connectors you actually operate. Strong defaults stay where omp already put them.

| Piece | Where it lands |
| --- | --- |
| Roles, advisor, memory, approvals | `~/.omp/agent/config.yml` ([§6](setup.md#6-model-routing), [§7](setup.md#7-advisor), [§10](setup.md#10-editing-approvals-and-parallel-safety), [§13](setup.md#13-memory-and-autolearn)) |
| Personal MCP servers | `~/.omp/agent/mcp.json` ([§12](setup.md#context7-every-repo)) |
| Specialists for every repo | `~/.omp/agent/agents/*.md` ([§11](setup.md#11-subagents)) |
| One repository's overrides | `<repo>/.omp/config.yml` ([§5](setup.md#5-credentials-and-config-layout)) |
| Repo rules the agent loads | `<repo>/AGENTS.md` ([§14](setup.md#agentsmd)) |
| The file to copy when two models fit | [§20](setup.md#20-complete-global-config) |

The [verification checklist](setup.md#18-verification-checklist), [troubleshooting table](setup.md#19-troubleshooting), and [directory map](setup.md#21-directory-layout) are in the guide. After the first week, [§23](setup.md#23-after-the-first-week) uses `omp stats --summary` to decide whether the advisor reviews every turn or only at the end.

## The guide, section by section

1. [Purpose](setup.md#1-purpose)
2. [Leave these defaults alone](setup.md#2-leave-these-defaults-alone)
3. [Prerequisites](setup.md#3-prerequisites)
4. [Install](setup.md#4-install)
5. [Credentials and config layout](setup.md#5-credentials-and-config-layout)
6. [Model routing](setup.md#6-model-routing). The local loop is [Get the loop running](setup.md#get-the-loop-running), then the [tool-call probe](setup.md#the-model-has-to-call-a-tool). One model: [24 GB](setup.md#24-gb-unified-memory) or [one-model](setup.md#one-model-machine). Two models: [default stack](setup.md#default-local-stack).
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
