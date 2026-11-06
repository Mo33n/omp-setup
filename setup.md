# OMP World-Class Coding Agent Setup Guide

A copy-paste setup for [omp](https://github.com/can1357/oh-my-pi) (Oh My Pi) as a coding harness that outruns a single-model agent: role-routed models, a second model reviewing every turn, semantic navigation, a real debugger, hash-anchored edits, isolated worktree subagents, live GitHub, a real browser, current library docs, and project memory.

This guide matches the current omp settings schema, CLI, and file formats. `omp config list` remains the authority for every key on the build you installed. Every chat role in this guide is a model you pull and run locally. Copy the selector from `omp models ollama` after the server is up. Hosted APIs are covered only in the optional note in §6.

Official references:

- Settings: https://omp.sh/docs/settings and https://github.com/can1357/oh-my-pi/blob/main/docs/settings.md
- Models: https://github.com/can1357/oh-my-pi/blob/main/docs/models.md
- Code intelligence: https://omp.sh/docs/code-intelligence and https://github.com/can1357/oh-my-pi/blob/main/docs/lsp-config.md
- Debugging: https://omp.sh/docs/debugging
- Subagents: https://omp.sh/docs/subagents and https://omp.sh/docs/subagent-authoring
- MCP: https://omp.sh/docs/mcp and https://github.com/can1357/oh-my-pi/blob/main/docs/mcp-config.md
- GitHub: https://omp.sh/docs/github
- Web and browser: https://omp.sh/docs/web
- Hooks: https://omp.sh/docs/hooks
- Memory: https://omp.sh/docs/memory
- Skills: https://omp.sh/docs/skills
- CLI: https://omp.sh/docs/cli
- Editor integration: https://omp.sh/docs/acp and https://agentclientprotocol.com/get-started/clients

---

## 1. Purpose

omp is an MIT-licensed, bring-your-own-model terminal coding agent. Claude Code is a strong agent tied to one vendor, with MCP, skills, and hooks of its own. This setup is the harness around omp: the same session can rename through a language server, stop a real process in a debugger, reject a stale patch, take a second model's review, check a pull request out into a worktree, read the current library docs, and query the tracker and the error tracker you already use.

That is a better coding system when all of these are true in one session:

| Capability | How this setup does it |
| --- | --- |
| Semantic edits | LSP rename, diagnostics, and code actions for source, and for YAML, shell, Terraform, Dockerfiles, and Markdown once those binaries are installed. Hashline rejects a patch whose anchors no longer match the file. |
| Runtime truth | DAP breakpoints, stacks, and variables. The agent observes the process. |
| A second opinion | `advisor` reviews primary turns. On a machine that can hold two local models, the advisor is a different family from `default`. |
| Parallel implementation | `task` fans out into git worktrees. The parent integrates. |
| GitHub without a second GitHub tool | Native `github` uses your `gh` login. PR checkout is a worktree. Actions can be watched from the session. |
| Current library APIs | Context7 over MCP, loaded when the task needs docs. Training data is the fallback. |
| How a public dependency is built | DeepWiki over MCP, with no key. It answers how a public repository is put together. Private repositories stay off this default. |
| The systems around the code | Linear read-only, project-scoped Sentry, a restricted Postgres connection, and Figma only in a repo that implements from a Figma file and only after `/mcp test figma` succeeds. |
| A real browser | omp's browser for public pages. Browser Relay attaches a logged-in Chrome tab. Playwright MCP is left off, because omp drops those servers while its own browser is enabled. |
| A policy the model cannot talk its way past | A hook blocks `git push --force`. `bash.gitGuard` blocks destructive git in a shared checkout. |
| Local weights only | Every chat role runs on Ollama, llama.cpp, or LM Studio on this machine. Hosted models stay disabled until you opt in. |

Use a native omp tool when one exists. Add an MCP server only for a system the native tools do not speak. Each extra server adds tools and instructions to every turn, including subagent turns.

Leave a default in place when it is already the strong setting. The work is credentials, role assignment, the advisor, language servers, debug adapters, GitHub, search, Context7, DeepWiki, memory, and the per-repo connectors you actually operate.

---

## 2. Leave these defaults alone

Confirm them. Write them into `config.yml` only when `omp config get` shows something else.

| Setting | Default | Why it stays |
| --- | --- | --- |
| `lsp.enabled` | `true` | Semantic navigation is already on. `--no-lsp` disables one run. |
| `debug.enabled` | `true` | The debugger tool is already registered. The missing piece is the adapter binary. |
| `edit.mode` | `hashline` | Content-hash anchors. Other modes: `apply_patch`, `patch`, `replace`, `sloppy`. |
| `edit.fuzzyMatch` | `true` | Near-miss anchors can still apply, at `edit.fuzzyThreshold` 0.95. |
| `edit.enforceSeenLines` | `true` | Rejects anchors on lines a prior read never displayed in full. |
| `lsp.diagnosticsOnWrite` | `true` | Diagnostics come back after a full file write. |
| `lsp.diagnosticsDeduplicate` | `true` | The same diagnostic is not pasted back into context unless it changed. |
| `lsp.lazy` | `true` | Language servers start when a file needs them. Warming them all competes with the local coder for memory. |
| `lsp.shared` | `true` | One server per project, including worktree subagents. |
| `retry.modelFallback` | `true` | Fallback chains are allowed to run. |
| `retry.fallbackOn` | `any` | Provider errors may walk the chain. See the rate-limit exception in §6. |
| `compaction.enabled` | `true` | Automatic compaction. |
| `compaction.keepRecentTokens` | `20000` | Recent history kept verbatim during summary compaction. |
| `compaction.autoContinue` | `true` | The task continues after compaction. |
| `compaction.thresholdPercent` | `-1` | Reserve-based trigger: about 85% on large windows, with a reserve floor. |
| `defaultThinkingLevel` | `high` | Strong default on a large window. §6 sets `low` when the window is 16k or 32k, because the `high` budget is 16384 tokens. |
| `tools.approvalMode` | `yolo` | Product default. This guide changes it to `write` in §10. |

This guide turns on three things that default to off: `advisor.enabled`, `memory.backend` (the memory default is `off`), and `github.enabled`. `autolearn.enabled` and `browser.relay` also default to off. `browser.enabled` and `web_search.enabled` are already on. §8 sets `lsp.diagnosticsOnEdit` to true. Its default is false, and hashline edits never hit `diagnosticsOnWrite`.

---

## 3. Prerequisites

- macOS, Linux, or Windows. On Windows, WSL2 is the comfortable path.
- Git, configured with your name and email.
- A local inference server: Ollama, llama.cpp, or LM Studio. No hosted API key is required.
- Enough memory for a coder that returns real tool calls (§6). 32 GB holds `qwen3-coder:30b`. 24 GB unified memory holds `qwen3:14b` at a 16384 context. 16 GB needs a smaller model that still passes the tool-call probe.
- Language-server binaries for the languages you actually work in (§8).
- Debug-adapter binaries for the languages you want to step through (§9).
- Node.js if you will launch an MCP server with `npx`, or run the JavaScript debug adapter. The omp binary itself can be installed without Node.
- GitHub CLI (`gh`), authenticated, for the native GitHub tool (§12).
- Chrome, if you want Browser Relay to drive a logged-in tab.
- `uv` (`uvx`) if a repo should expose Postgres through `postgres-mcp`.

---

## 4. Install

macOS or Linux:

```sh
curl -fsSL https://omp.sh/install | sh
```

The script installs a prebuilt binary when Bun is missing, and checks its SHA-256. Pin a release with:

```sh
curl -fsSL https://omp.sh/install | sh -s -- --binary --ref vX.Y.Z
```

Alpine needs the dynamic C++ runtime before the musl build will start:

```sh
apk add libstdc++ libgcc
```

Windows PowerShell:

```powershell
irm https://omp.sh/install.ps1 | iex
```

Homebrew:

```sh
brew install can1357/tap/omp
```

Bun:

```sh
bun install -g @oh-my-pi/pi-coding-agent
```

Nix:

```sh
nix profile add github:can1357/oh-my-pi
```

Check the binary:

```sh
omp --version
omp --help
```

---

## 5. Credentials and config layout

### Local models

Chat roles in this guide call a server on `127.0.0.1`. omp discovers these engines with no API key:

| Engine | Provider id | Default endpoint |
| --- | --- | --- |
| Ollama | `ollama` | `http://127.0.0.1:11434` (`OLLAMA_BASE_URL`, then `OLLAMA_HOST`) |
| llama.cpp | `llama.cpp` | `http://127.0.0.1:8080` (`LLAMA_CPP_BASE_URL`) |
| LM Studio | `lm-studio` | `http://127.0.0.1:1234/v1` (`LM_STUDIO_BASE_URL`) |

Start the server, pull or load a model, then:

```sh
omp models ollama
omp models llama.cpp
omp models lm-studio
```

A selector is `provider/model-id` as that command prints it. An id omp cannot see stays unresolved. omp does not substitute another model for an explicit role.

Hosted login (`/login`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, and the rest) is unused by the default config. The optional note at the end of §6 is the only place this guide turns a hosted provider back on.

Startup warnings name broken fallback chains. Read them.

### Where files live

| Scope | Path | What belongs there |
| --- | --- | --- |
| Global settings | `~/.omp/agent/config.yml` | Roles, advisor, memory, approvals, compaction |
| Legacy global name | `~/.omp/agent/config.yaml` | Read and updated in place when it already exists |
| Project settings | `<repo>/.omp/config.yml` | Overrides for one repository |
| Custom providers | `~/.omp/agent/models.yml` | `providers:` entries only |
| User subagents | `~/.omp/agent/agents/*.md` | Specialists for every repo |
| Project subagents | `<repo>/.omp/agents/*.md` | Same `name` wins over the user file |
| User skills | `~/.omp/agent/skills/<name>/SKILL.md` | Playbooks for every repo |
| Project skills | `<repo>/.omp/skills/<name>/SKILL.md` | Playbooks for this repo |
| Managed skills | `~/.omp/agent/managed-skills/` | Autolearn output |
| User MCP | `~/.omp/agent/mcp.json` | Personal servers |
| Project MCP | `<repo>/.omp/mcp.json` | Servers everyone in the repo should get |
| User LSP | `~/.omp/agent/lsp.json` | Personal language-server overrides |
| Project LSP | `<repo>/.omp/lsp.json` | Project language-server overrides |
| User DAP | `~/.omp/agent/dap.json` | Personal debug-adapter overrides |
| Project DAP | `<repo>/.omp/dap.json` | Project debug-adapter overrides |
| Instructions | `<repo>/AGENTS.md` | Repo rules omp loads into context |
| Named profile | `~/.omp/profiles/<name>/agent/` | Isolated auth, settings, sessions, MCP |

Precedence, highest first: runtime flags, `--config` overlays, project settings, global settings, built-in defaults. Environment variables declared on a setting outrank the file for that process.

Useful commands:

```sh
omp config list
omp config get modelRoles
omp config set advisor.enabled true
omp config set tools.approvalMode write
```

`omp config set` writes the global file. Arrays and records take JSON. Booleans accept `true`/`false`, `yes`/`no`, `on`/`off`, `1`/`0`. Edit `<repo>/.omp/config.yml` by hand when a single repository needs a different value. With `modelRoleStorage: project`, the model picker writes `modelRoles` into the project file.

A named profile isolates user-level auth and settings:

```sh
omp --profile work
```

Project `.omp/` still applies under every profile.

---

## 6. Model routing

File: `~/.omp/agent/config.yml`.

Built-in chat roles: `default`, `smol`, `slow`, `vision`, `plan`, `commit`, `tiny`, `memory`, `task`, `advisor`, `widget`.

Built-in model-kind roles: `image`, `web`, `speech`, `dictation`, `judge`.

A selector is `provider/model-id`, optionally with a thinking suffix: `:off`, `:minimal`, `:low`, `:medium`, `:high`, `:xhigh`, `:max`, `:auto`, `:inherit`. A value may also be a role alias such as `"@smol"`, or a comma-separated list tried in order.

`modelRoles.<role>` is the primary. `retry.fallbackChains.<role>` is the ordered fallback. For chat roles, the effective chain is the primary plus those entries. A key that contains `/` is model-oriented and beats a role key: `provider/model-id` matches that model, `provider/*` matches every model of that provider. A `provider/*` entry keeps the failing model id and swaps the provider.

Rate limits are the exception. With `retry.rateLimitWalksChain` left at its default `false`, a 429 retries the same model until `retry.maxRetries` (default 10) is spent, because that turn's thinking is signed by the failing model. Set `retry.rateLimitWalksChain: true` when a 429 should walk the chain immediately. Quota and outage behavior is `retry.fallbackOn` (`any`, `usage-limit`, `except-usage-limit`). `retry.fallbackRevertPolicy: cooldown-expiry` returns to the primary after its cooldown. `never` stays on the fallback until you switch.

Rate limits matter only after you opt into a hosted model. With `retry.rateLimitWalksChain` left at `false`, a 429 retries that same model until `retry.maxRetries` (default 10) is spent. Local servers fail differently: the process is down, the model is not pulled, or the context does not fit. `retry.fallbackChains` then hands the turn to the next local model you listed.

### Pull the local coders

Ollama is the path this guide uses. Install it, start it, and pull weights that fit the machine. The main loop needs a model that returns tool calls, not only a model that writes good code in chat. Qwen3-Coder is the default on 32 GB. `qwen3:14b` is the verified coder on 24 GB unified memory. Devstral is a second local family for review when both fit. gpt-oss 20B is the 16 GB candidate only after the probe below succeeds.

```sh
ollama serve
```

```sh
# 32 GB unified memory, or a 24 GB GPU: the default coder
ollama pull qwen3-coder:30b

# Second local family, for the advisor, when the machine can swap or hold both
ollama pull devstral-small-2:24b

# 24 GB unified memory: the coder that fits and returns tool calls
ollama pull qwen3:14b

# 16 GB machines: candidate default. Run the tool-call probe before assigning it.
ollama pull gpt-oss:20b

# 64 GB and above: optional larger coder for the slow role
ollama pull qwen3-coder-next
```

If a tag is missing, open the Ollama library, pull the current coding tag, and copy the id `omp models ollama` prints. Do not invent a selector.

An agent needs a long context. Ollama's default is often a few thousand tokens, which drops the middle of the session. Raise it with a Modelfile and tell omp the same number. `OLLAMA_CONTEXT_LENGTH` sets omp's budget. It does not change Ollama's `num_ctx` by itself.

```text
# Modelfile.qwen3-coder-64k
FROM qwen3-coder:30b
PARAMETER num_ctx 65536
```

```text
# Modelfile.devstral-64k
FROM devstral-small-2:24b
PARAMETER num_ctx 65536
```

```text
# Modelfile.qwen3-16k
FROM qwen3:14b
PARAMETER num_ctx 16384
```

```sh
ollama create qwen3-coder-64k -f Modelfile.qwen3-coder-64k
ollama create devstral-64k -f Modelfile.devstral-64k
ollama create qwen3-16k -f Modelfile.qwen3-16k
export OLLAMA_CONTEXT_LENGTH=65536
```

On 24 GB, export `OLLAMA_CONTEXT_LENGTH=16384` instead, and use `qwen3-16k` as `default`. On 16 GB, use `32768` for both `num_ctx` and `OLLAMA_CONTEXT_LENGTH`, and build `default` from `gpt-oss:20b` only after the probe below succeeds.

### The model has to call a tool

omp is a loop. You give a goal. omp sends that goal and the list of tools. The model either talks, or it requests an action such as `write`, `read`, `edit`, or `bash`. omp runs the action and sends the result back. The model then takes another lap or stops. The model never touches the disk itself.

A reply that merely contains JSON is not a tool call. Ollama accepts a call only when the model emits the tags in its template:

```xml
<tool_call>
{"name": "write", "arguments": {"path": "hello.txt", "content": "ready"}}
</tool_call>
```

or when the HTTP response fills `tool_calls`. `qwen2.5-coder:7b` and `qwen2.5-coder:14b` print the inner JSON and leave the tags off. Ollama then leaves `tool_calls` empty. omp stores the paragraph and exits, so the file is never created. This happens on Ollama's own `/api/chat` as well as through omp. `tools.format` stays `auto`. Setting it to `hermes` does not insert the missing tags, and with the full tool list that request returned HTTP 500.

Do not assign `qwen2.5-coder` to `default`, `smol`, `task`, or `advisor`, and do not put it in `retry.fallbackChains`. A fallback onto that model ends the loop the same way. It can still sit in `ollama list` as a chat model. It cannot drive this agent.

`qwen3:14b` passes. On a 24 GB Mac, `qwen3-16k` loaded at about 10 GB, 100% GPU, context 16384, and `/api/chat` returned `tool_calls` with an empty `content`. `qwen3-coder:30b` is the stronger agent and its weights are about 19 GB. On 24 GB unified memory that size swaps. Use it when `ollama ps` can show it at 100% GPU without the machine paging.

Prove the assigned coder before a real task:

```sh
mkdir -p "$HOME/omp-probe"
cd "$HOME/omp-probe"
omp -p --auto-approve --thinking off --max-time 4m \
  "Use the write tool to create hello.txt. The contents must be exactly the word ready. Then read hello.txt and stop."
cat hello.txt
```

`hello.txt` must contain `ready`. A transcript that describes a write, with no file on disk, means this model cannot run the loop. Pick another model and probe again.

Leave the stock Ollama chat template in place. Rewriting it to force `/no_think`, or to pre-seed an empty `<think>` block, made omp discard the reply (`discarded-entry-branch`) and the write never ran.

`--thinking off` records the level for that run. It does not set Ollama's think flag, so Qwen 3 can still open a thinking block. On a short task the block and the tool call arrive in the same turn, and the file is written. On a large task the block can be the whole turn, and the write never starts.

Do not shut that off by replacing the built-in Ollama provider in `~/.omp/agent/models.yml`. `disableReasoningWithTools` is a compat flag for the OpenAI API family, and the models file schema rejects `api: ollama-chat`. A `providers.ollama` entry with no discovery removes the default model. The same entry with `api: openai-completions` and the Ollama root URL returns HTTP 404. That file is for an extra endpoint, not for patching the built-in coder.

Keep the built-in provider and the stock template. Give the model one file and one `write`, then a `read`, then the check. Prefer `write` over a hashline `edit`. Leave `defaultThinkingLevel` at `low`. `high` budgets 32768 tokens, the entire 16k window.

A page-sized task can still finish. Limit that print run to the tools it needs:

```sh
omp -p --auto-approve --thinking off --tools write,read,bash --max-time 10m \
  "Your first action is the write tool. Create the file, then read it, then run the check."
```

The first response stream may close inside the thinking block, with `OpenAI responses stream closed before a terminal response event was received`. omp continues that same session, and the next turn can call `write`. On this machine that sequence wrote `index.html`, read it back, and ran a `bash` check that printed `OK`. The run took about six minutes on `ollama/qwen3-16k:latest`. The name check does not prove the page plays. Read the file. A 9x9 grid has to be nine `tr` rows, and each stored puzzle has to be 81 characters.

Keep the check command free of nested quotes. A `node -e` line full of escaped quotes was rejected by the shell (`syntax error`), and the model repeated that same command until the time limit. The file from the `write` stayed on disk. A second write in that situation built the grid with `tr` rows, and the stored solutions were still not legal Sudoku. Judge the page by reading it, not by the model's last sentence.

### 24 GB unified memory

One coder fits. Build `qwen3-16k` as above. Copy the selector from `omp models ollama`. It looks like `ollama/qwen3-16k:latest`.

```yaml
modelRoles:
  default: ollama/qwen3-16k:latest
  slow: "@default"
  plan: "@default"
  smol: "@default"
  commit: "@smol"
  task: "@default"
  advisor: "@default"
  tiny: local/lfm2.5-230m
  memory: local/lfm2-1.2b
  web: web/duckduckgo

retry:
  fallbackChains:
    default: []
    web: []
    dictation: []

defaultThinkingLevel: low

advisor:
  enabled: true
  reviewMode: agent-end
```

`defaultThinkingLevel: high` budgets 16384 thinking tokens, which is the whole 16k window. `low` leaves room for the tools and the file. Qwen 3 still opens a thinking block. On a large first task that block can be the entire turn, and the write never starts. A print run that must act uses `--thinking off`.

`reviewMode: agent-end` reviews once, when the agent yields. `turn` runs a second full generation on the same GPU after every message. On this hardware that is the difference between a reply and a machine that is paging.

`ollama ps` after a probe should show `qwen3-16k`, about 10 GB, `100% GPU`, context `16384`. A 14 GB resident size on a 24 GB Mac means the context is too long. Lower `num_ctx` and `OLLAMA_CONTEXT_LENGTH` together.

On-device helpers for titles and memory extraction, so those turns do not load the coder:

```sh
omp tiny-models list
omp tiny-models download all
```

llama.cpp and LM Studio are the other local engines. Point llama-server at `127.0.0.1:8080` or LM Studio at `127.0.0.1:1234/v1`, load the same class of weights, and use the selector `omp models` prints (`llama.cpp/...` or `lm-studio/...`). Qwen on llama.cpp is discovered as `openai-completions` so its thinking controls still work. `apple/on-device` is local, and its window is too small for this coding loop. Use it only as `tiny` or `memory` when the omp tiny models are unavailable.

### Default local stack

Use this when `omp models ollama` lists the three created models. `default` is the coder. `advisor` is Devstral, a different local family, so review is a second set of weights. `task` stays on `@default`: one GPU should not fan out onto a second large model. `web` is `web/duckduckgo`, which is a search index, not a chat model. The query leaves the machine. Set `web_search.enabled: false` when the session must stay offline.

`disabledProviders` blocks hosted chat APIs. Local engines (`ollama`, `llama.cpp`, `lm-studio`, `apple`) stay available. Weights you pulled still appear under `ollama/` even when the vendor's hosted provider is in this list.

```yaml
modelRoles:
  default: ollama/qwen3-coder-64k
  slow: ollama/qwen3-coder-64k:high
  plan: "@slow"
  smol: "@default"
  commit: "@smol"
  task: "@default"
  advisor: ollama/devstral-64k
  tiny: local/lfm2.5-230m
  memory: local/lfm2-1.2b
  web: web/duckduckgo

retry:
  fallbackChains:
    default:
      - ollama/devstral-64k
    slow:
      - ollama/qwen3-coder-64k
    smol:
      - local/lfm2-1.2b
    advisor:
      - ollama/qwen3-coder-64k
    web: []
    dictation: []

disabledProviders:
  - anthropic
  - openai
  - openai-codex
  - google
  - google-vertex
  - google-antigravity
  - openrouter
  - groq
  - xai
  - deepseek
  - mistral
  - github-copilot
  - cursor

cycleOrder:
  - smol
  - default
  - slow
```

Drop the `:high` suffix if that role does not resolve. Some local servers have no thinking ladder. If `omp tiny-models` has no `local/lfm2.5-230m` or `local/lfm2-1.2b`, point `tiny` and `memory` at `"@smol"`.

### One-model machine

Use this when only one coder fits. The advisor uses the same weights, so set `reviewMode` to `agent-end` as in the 24 GB block above. `turn` would generate every reply twice on that one GPU. On 24 GB the coder is `ollama/qwen3-16k:latest`, not `qwen3-coder-64k`.

```yaml
modelRoles:
  default: ollama/qwen3-coder-64k
  slow: "@default"
  plan: "@default"
  smol: "@default"
  commit: "@smol"
  task: "@default"
  advisor: "@default"
  tiny: "@smol"
  memory: "@smol"
  web: web/duckduckgo

retry:
  fallbackChains:
    default: []
    web: []
```

`smol` stays `"@default"` until a smaller model passes the tool-call probe. On 16 GB, change `default` to the gpt-oss model you created with `num_ctx` 32768, after that probe. An 8 GB machine does not get this loop from `qwen2.5-coder:7b`. That model writes code in chat and does not return `tool_calls`.

### 64 GB and above

Point `slow` and `plan` at the larger local coder after you create it with `num_ctx` 65536 or higher:

```yaml
modelRoles:
  slow: ollama/qwen3-coder-next-64k
  plan: "@slow"
```

Leave `default` on `qwen3-coder-64k` so ordinary edits stay on the faster model. Move `slow` back to `@default` if the larger model will not stay resident.

### Role guide

| Role | Use | Local selection |
| --- | --- | --- |
| `default` | Main coding loop | Best coder that passes the tool-call probe and stays resident. `qwen3-coder:30b` at 32 GB. `qwen3:14b` at 16384 on 24 GB. `gpt-oss:20b` at 16 GB only after the probe. |
| `slow` | Hard review and architecture | The same coder with `:high` when the server has a thinking ladder, or `qwen3-coder-next` at 64 GB |
| `plan` | Plan mode | `"@slow"` |
| `smol` | Titles and mechanical edits | `"@default"` until a smaller model passes the tool-call probe. A model that cannot call tools cannot edit. |
| `commit` | Commit messages | `"@smol"` |
| `task` | Subagent fan-out | `"@default"` on one GPU. A second large model thrashes memory. |
| `advisor` | Reviews primary turns | `devstral-small-2:24b` when it fits. Otherwise `"@default"`. |
| `vision` | Image questions | A local vision tag you pulled, such as a Qwen-VL or Gemma vision model, after `omp models ollama` lists it. Leave unset until then. |
| `web` | Live search | `web/duckduckgo` only. This is not a local LLM. |
| `tiny` / `memory` | On-device titles and memory extraction | `local/lfm2.5-230m` and `local/lfm2-1.2b` after `omp tiny-models download` |

`web_search` follows `modelRoles.web` and `retry.fallbackChains.web`. This guide sets the chain to `[]` so search cannot fall through to a hosted search model. Confirm with `omp web-search --model web/duckduckgo "current Bun release"`.

Open `/model` and the Roles view and confirm every role is `ollama/`, `llama.cpp/`, `lm-studio/`, `local/`, or `apple/`. `omp models --kind chat` lists the chat catalog.

### Optional: a hosted model

The default config refuses hosted chat providers. Use a third-party LLM only when you choose to, for one role, after the local stack already works.

1. Remove that provider's id from `disabledProviders`.
2. Sign in with `/login` or export that provider's API key.
3. Run `omp models` and copy one selector that server lists.
4. Assign it to `slow` or `advisor`. Leave `default`, `task`, `smol`, and `commit` on local models.
5. Do not add the hosted selector to `retry.fallbackChains` unless a local outage should send the repository to that vendor.

Putting a hosted model on `default` makes the coding loop leave the machine. That is outside this guide's primary setup.

Custom local endpoints go in `~/.omp/agent/models.yml` under `providers:`, with `discovery.type` of `ollama`, `llama.cpp`, or `lm-studio`, and `auth: none`. Unknown root keys there do not configure model behavior.

---

## 7. Advisor

Setting `modelRoles.advisor` does not start review. The runtime is off until you enable it.

```yaml
advisor:
  enabled: true
  reviewMode: turn
  reviewInterval: 1
  continueSeverity: concern
  maxNotesPerUpdate: 4
  syncBacklog: "off"
  immuneTurns: 3
```

| Key | What to set | Effect |
| --- | --- | --- |
| `advisor.enabled` | `true` | Starts review when the advisor model resolves. `/advisor on` and `--advisor` enable one run. |
| `reviewMode` | `turn` | Review every primary turn. `agent-end` reviews only the final yield and costs less. |
| `reviewInterval` | `1` | Every eligible update. Raise it to review every Nth update. |
| `continueSeverity` | `concern` | After a final answer, a concern or blocker wakes the agent. `nit` stays on the transcript until your next message. Severities: `nit`, `concern`, `blocker`. |
| `maxNotesPerUpdate` | `4` | Non-blocker notes kept per review, from 1 to 32. |
| `syncBacklog` | `off` | The main turn does not wait on the advisor. `1`, `3`, or `5` wait up to 30 seconds at that backlog. `strict` waits with no wall-clock cap. |
| `immuneTurns` | `3` | After a concern or blocker interrupts, further concerns are asides for this many primary turns. Blockers still interrupt. |
| `evictStaleResults` | leave `true` | Older advisor `read` / `grep` / `glob` output is replaced by a placeholder before the next review. |

`syncBacklog: "off"` must be quoted so YAML does not turn it into a boolean.

`reviewMode: turn` plus `reviewInterval: 1` is a second local generation on every primary turn. Use that when the advisor is a second model that stays resident. When `advisor` is `"@default"`, set `reviewMode` to `agent-end`. One GPU then reviews once, at the end of the task, instead of generating every reply twice. A turn that only thinks still counts as a yield, so `turn` will spend another full generation on that plan and the write waits.

A roster of named advisors with their own models, tools, and cadence lives in `WATCHDOG.yml`. The single advisor above is the right start. Per-subagent override: `task.agentAdvisor`, or the `advisor` frontmatter field, editable from `/agents`.

Verify:

```sh
omp config get advisor.enabled
omp config get modelRoles.advisor
```

In a session, do a small edit and confirm an advisor card appears.

---

## 8. Language servers

`lsp.enabled` is already true. omp discovers bundled servers whose binary resolves and whose root markers match the project. It does not download those binaries.

Install the ones you use, then start a new session so detection runs again.

```sh
# TypeScript / JavaScript (TypeScript 6 and below)
npm install -g typescript typescript-language-server

# Python — pick one language server, plus ruff if you want lint diagnostics
# binaries omp looks for: ty, pyright-langserver, basedpyright-langserver, pylsp, ruff
pip install ty
# or: npm install -g pyright

# Rust
rustup component add rust-analyzer

# Go
go install golang.org/x/tools/gopls@latest

# Others omp already knows, once the binary is on PATH:
# clangd, zls, jdtls, kotlin-lsp, biome, vscode-eslint-language-server,
# vue-language-server, svelte, elixir-ls, ruby-lsp, gopls, sourcekit-lsp, …
```

Project virtualenvs are searched before `PATH` (`.venv/bin`, `venv/bin`). Confirm from the same shell that will launch omp:

```sh
command -v typescript-language-server
command -v gopls
command -v rust-analyzer
```

### YAML, shell, Terraform, Docker, and Markdown

These servers are already built in. omp starts one when its binary is on `PATH` and a root marker is in the directory you launched from. Do not add them to `lsp.json`.

After a write, diagnostics come back through `lsp.diagnosticsOnWrite`. The local model then fixes the file from that error instead of rereading it and guessing. Rename, definition, and references go through the same server.

| Built-in name | Binary omp looks for | Files | What a write should surface |
| --- | --- | --- | --- |
| `yamlls` | `yaml-language-server` | `.yaml`, `.yml` | An unknown key, a value of the wrong type, or bad indentation in workflows, manifests, and `config.yml` |
| `bashls` | `bash-language-server` | `.sh`, `.bash`, `.zsh` | A script or hook that will not parse |
| `terraformls` | `terraform-ls` | `.tf`, `.tfvars` | An invalid resource, a missing argument, or a wrong type |
| `dockerls` | `docker-langserver` | `Dockerfile`, `.dockerfile` | An unknown instruction or a flag that image does not accept |
| `marksman` | `marksman` | `.md`, `.markdown` | A broken link to another heading or file in the repo |

YAML, shell, and Markdown use `.git` as a root marker, so they start in any git checkout. Terraform starts when the launch directory contains a `.tf` file, `.terraform`, or `terraform.tfstate`. Docker starts when that directory contains a `Dockerfile`, `docker-compose.yml`, `docker-compose.yaml`, or `.dockerignore`. omp checks those markers in the launch directory only. It does not walk subdirectories. Launch omp in the directory that holds the file, or keep the marker at the repo root.

```sh
npm install -g yaml-language-server bash-language-server dockerfile-language-server-nodejs
brew install terraform-ls marksman
```

`dockerfile-language-server-nodejs` provides the `docker-langserver` binary. On Debian and Ubuntu, install the HashiCorp package named `terraform-ls`. When Homebrew is not available, Marksman is a single binary from https://github.com/artempyanykh/marksman/releases.

```sh
command -v yaml-language-server
command -v bash-language-server
command -v terraform-ls
command -v docker-langserver
command -v marksman
```

Start a new session after installing. A Helm chart also starts the built-in `helm-ls` server (`helm_ls`) when `Chart.yaml` is in the launch directory.

You usually do not need an LSP config file. Add `<repo>/.omp/lsp.json` to override a built-in server or to silence a duplicate. Overrides merge onto the built-in entry. The map key is `servers`, and the server key is the built-in name.

A custom server needs `command`, `args`, `fileTypes`, and `rootMarkers`. User and project files merge. Highest precedence is an `lsp.json` in the directory where omp starts, then `.omp/lsp.json`. The same directory also accepts `.lsp.json` and YAML variants.

### Diagnostics after a hashline edit

This harness edits with hashline. `lsp.diagnosticsOnWrite` runs after a full file write. `lsp.diagnosticsOnEdit` runs after the smaller edits, and it defaults to false. Turn it on, and cap how long a server may sit on one reply. `requestTimeout` is seconds.

```sh
omp config set lsp.diagnosticsOnEdit true
omp config set lsp.requestTimeout 10
```

The same block belongs in `~/.omp/agent/config.yml`. §20 includes it.

```yaml
lsp:
  diagnosticsOnEdit: true
  requestTimeout: 10
```

Leave `lsp.diagnosticsDeduplicate`, `lsp.lazy`, and `lsp.shared` at their defaults. Deduplication keeps a repeated error out of the local context. Lazy start keeps every server from loading beside the coder. Shared servers mean a worktree subagent reuses the one already running.

Leave `lsp.formatOnWrite` off unless that server's formatter is the one the repo uses, such as Ruff, gofmt, or rustfmt. A TypeScript server formatting a Prettier repo will fight the repo.

Rust-analyzer's built-in settings already set `checkOnSave` to false, so diagnostics on edit do not start a full cargo check.

### One type server, plus the linter the repo uses

Pick one Python type server and disable the others. Install that server and Ruff into the project virtualenv so `.venv/bin` wins over a global binary.

```sh
uv add --dev basedpyright ruff
# or: uv add --dev ty ruff
# or: uv add --dev pyright ruff
```

```json
{
  "servers": {
    "basedpyright": {
      "command": "basedpyright-langserver",
      "args": ["--stdio"],
      "fileTypes": [".py", ".pyi"],
      "rootMarkers": ["pyproject.toml", "pyrightconfig.json", "setup.py", "requirements.txt"]
    },
    "pyright": { "disabled": true },
    "ty": { "disabled": true },
    "pylsp": { "disabled": true }
  }
}
```

Swap which name stays enabled to match the package you installed. Ruff is a linter (`ruff server`). It starts when `pyproject.toml`, `ruff.toml`, or `.ruff.toml` is in the launch directory. Do not disable it when you wanted lint diagnostics.

TypeScript 6 and below uses `typescript-language-server`. `tsc` on `PATH` also matches `package.json` as the built-in `typescript-native` server (`tsc --lsp`). Disable that one. On TypeScript 7, when `tsc --lsp` is the server you want, disable `typescript-language-server` instead.

```json
{
  "servers": {
    "typescript-native": { "disabled": true }
  }
}
```

Install the lint server only when the repo has its config. Biome starts from `biome.json` or `biome.jsonc`. ESLint starts from an ESLint config. The type server does not report those lint results.

```sh
npm install -g @biomejs/biome
npm install -g vscode-langservers-extracted
```

`vscode-langservers-extracted` provides `vscode-eslint-language-server`. The same package provides the HTML, CSS, and JSON servers omp already knows. Those start in a git checkout.

### Schemas and a compile database

`clangd` starts when `CMakeLists.txt`, `Makefile`, `compile_commands.json`, `.clangd`, or `.clang-format` is in the launch directory. Flags still come from a compilation database. Generate it and link it at the repo root.

```sh
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Debug
ln -sfn build/compile_commands.json compile_commands.json
```

For a Makefile project, `bear -- make` writes `compile_commands.json` in the directory where you run it. `-DCMAKE_BUILD_TYPE=Debug` is what makes the debugger in §9 step through this binary.

A `settings` override on a built-in server replaces that object. For workflow files, keep the built-in YAML keys and add the schema. Point a Kubernetes schema only at the manifest directory. A glob of every `.yml` checks `config.yml` against the wrong schema.

```json
{
  "servers": {
    "yamlls": {
      "settings": {
        "yaml": {
          "validate": true,
          "format": { "enable": true },
          "hover": true,
          "completion": true,
          "schemas": {
            "https://json.schemastore.org/github-workflow.json": ".github/workflows/*.{yml,yaml}"
          }
        },
        "redhat": {
          "telemetry": { "enabled": false }
        }
      }
    }
  }
}
```

Add the schema URL for the cluster version under `yaml.schemas` when this repo has manifests.

### What to ask the language server

The `lsp` tool can go to definition, find references, find implementations, hover, read diagnostics, apply a code action, and rename. A rename goes through the language server, including `workspace/willRenameFiles`, so barrel files and aliased imports move with the symbol. Ask for a preview before a wide rename. If the server returns no edit, omp does not invent one.

```text
Explain the inferred type of result in src/parser.ts and show its definition.
Find every implementation of StorageAdapter before changing it.
Preview the rename of issueToken to mintToken, then apply it.
Move src/auth/token.ts to src/security/token.ts and update imports through the language server.
Apply the server's quick fix for the missing import in src/routes.ts, then read diagnostics again.
```

A specialist that renames or moves code needs `lsp` in its tool list. `test-writer` and `api-reviewer` have it. `migration-runner` applies a schema migration and does not.

---

## 9. Debugger

Debugger support is on by default. omp speaks DAP. Install at least one adapter and ask in ordinary language. You do not need a `dap.json` until an adapter lives off `PATH` or its defaults do not fit the project.

```sh
omp config get debug.enabled
```

| Target | Adapter name | Install |
| --- | --- | --- |
| C, C++, Rust | `gdb` | GDB built with DAP, on `PATH` as `gdb` |
| C, C++, Objective-C, Swift, Rust, Zig | `lldb-dap` | `lldb-dap` on `PATH` |
| C, C++, Rust, Zig | `codelldb` | `codelldb` on `PATH` |
| Python | `debugpy` | `python` on `PATH`, then `uv add --dev debugpy` in that interpreter |
| Go | `dlv` | `go install github.com/go-delve/delve/cmd/dlv@latest`, and `$GOBIN` or `~/go/bin` on `PATH` |
| Ruby | `rdbg` | `gem install debug` |
| JavaScript and TypeScript on Node | `js-debug-adapter` | vscode-js-debug's `dapDebugServer.js`. This is not an npm package named `js-debug-adapter` |
| .NET | `netcoredbg` | `netcoredbg` on `PATH` |
| Kotlin | `kotlin-debug-adapter` | binary on `PATH` |
| PHP | `php-debug-adapter` | binary on `PATH` |
| Bash | `bash-debug-adapter` | `bash-debug-adapter`, `bash`, and `bashdb` on `PATH` |
| Dart / Flutter | `dart-debug-adapter` / `flutter-debug-adapter` | Dart SDK |
| Elixir | `elixir-ls-debugger` | binary on `PATH` |

JavaScript: download a `js-debug-dap-*.tar.gz` from the vscode-js-debug releases and extract so this file exists:

```text
~/.local/opt/js-debug/src/dapDebugServer.js
```

That path is discovered automatically, as is a Neovim Mason install. Any other location:

```sh
export JS_DEBUG_DAP_SERVER=/absolute/path/to/js-debug/src/dapDebugServer.js
```

omp runs that server with `node` when `node` exists, otherwise with Bun.

Verify the adapter from the shell that starts omp:

```sh
command -v dlv
command -v lldb-dap
.venv/bin/python -m debugpy --version
```

`python` can exist when the `debugpy` module does not. The module check from the project interpreter is the one that matters.

### Adapter overrides

`<repo>/.omp/dap.json` for the repo, `~/.omp/agent/dap.json` for you. A `dap.json` in the start directory outranks `.omp/dap.json`. Maps merge. The key is `adapters`. Specify only what changes. This is not a VS Code `launch.json`: there is no top-level `"python": { "type": "debugpy", "program": ... }`.

```json
{
  "adapters": {
    "debugpy": {
      "command": ".venv/bin/python",
      "launchDefaults": {
        "justMyCode": true,
        "stopOnEntry": true
      }
    }
  }
}
```

Relative `command` values resolve from the debug working directory. Program path, working directory, arguments, pid, host, and port from the request win over `launchDefaults` and `attachDefaults`.

Use the virtualenv interpreter, not the first `python` on `PATH`. `justMyCode` skips library frames. Confirm the module from that interpreter:

```sh
.venv/bin/python -m debugpy --version
```

TypeScript on Node breaks in the file that actually runs. Set source maps on `js-debug-adapter`, set `outFiles` to where this repo emits JavaScript, and launch only after that build exists. A `.ts` breakpoint stays pending when the process is running the emitted `.js` and the map does not point back. omp does not expand VS Code `${workspaceFolder}` variables. Use a path relative to the debug working directory.

```json
{
  "adapters": {
    "js-debug-adapter": {
      "launchDefaults": {
        "sourceMaps": true,
        "outFiles": ["dist/**/*.js"]
      }
    }
  }
}
```

Change `dist/**/*.js` to this repo's emit directory.

A custom adapter needs a command plus enough matching data to be selected:

```json
{
  "adapters": {
    "acme-jvm": {
      "command": "./tools/acme-debug-adapter",
      "args": ["--stdio"],
      "languages": ["java", "kotlin"],
      "fileTypes": [".java", ".kt", ".kts"],
      "rootMarkers": ["pom.xml", "build.gradle", "build.gradle.kts"],
      "launchDefaults": {
        "request": "launch",
        "stopOnEntry": true,
        "projectRoot": "."
      }
    }
  }
}
```

### How to ask

Give a path, a stop, and the question. A launch target is a path. Ask omp to build first when the binary does not exist yet. Most built-in launches stop on entry so breakpoints can be installed before resume. Go can take a package directory. Other adapters want a file or executable.

```text
Debug etl/transform.py with sample.csv. Break on line 58 when i == 3,
inspect the locals and the running total, and explain where the bad value enters.
```

```text
Use lldb-dap to launch ./build/repro with arguments --seed 417.
Stop at main, continue until decode_packet, and inspect the packet header.
```

```text
Attach lldb-dap to PID 48120. Pause, show all threads, and identify
which thread is holding shutdown.
```

```text
Use debugpy to attach to 127.0.0.1:5678. When the request handler is reached,
inspect request.user and explain why authorization fails.
```

The process has to be listening before that attach. From the same virtualenv:

```sh
.venv/bin/python -m debugpy --listen 127.0.0.1:5678 path/to/app.py
```

An extensionless native binary is ranked `gdb`, then `lldb-dap`. Name the adapter for Rust:

```text
Use lldb-dap to launch ./target/debug/repro. Stop at main and step into decode_packet.
```

`codelldb` is the other Rust choice. Build the binary you launch with debug symbols. `cargo build` is a debug build. `cargo build --release` is the wrong one to step through. For C and C++, the Debug configure in §8 (`-DCMAKE_BUILD_TYPE=Debug`) is the matching build. A conditional breakpoint is an expression in the target language, as in `i == 3` in the Python example.

One root debug session at a time. Ask omp to end the current one before launching another. A stopped process is actually paused. Say "inspect without evaluating expressions" when evaluation might run target code.

Changing `debug.enabled` applies to a new session.

---

## 10. Editing, approvals, and parallel safety

### Hashline

```sh
omp config get edit.mode
```

The result should be `hashline`. The model points at lines, and the runtime checks that the surrounding content still hashes to what it read. A stale file rejects the patch. Re-read, then edit again. That rejection is the safety property.

The edit body has to start with `[path#HASH]` and a hunk header such as `PUT N.=M:`. A bare `PUT 3: new line` is rejected, the file stays unchanged, and omp sends the format error back so the model can try again. A full-file `write` does not need that shape. Prefer `write` when the model keeps missing the hunk header. `edit.modelVariants` can pin `replace` for one weaker model id if hashline never lands.

`PI_EDIT_VARIANT` pins the mode for one process (`hashline`, `apply_patch`, `patch`, `replace`, `sloppy`). `auto` leaves the setting unset. Per-model overrides:

```yaml
edit:
  modelVariants:
    some-weaker-model: replace
```

The first case-insensitive substring match wins. Keep `hashline` for the models you trust with line anchors.

Related defaults worth knowing:

- `edit.blockAutoGenerated: true` refuses generated and lockfile-like files.
- `edit.autoRepair.enabled` defaults to `false`. Set it `true` to let the `smol` model attempt a repair when an edit introduces an AST parse regression. omp accepts the repair only when the file parses again.

### Structural search

`ast_edit` is on by default: syntax-aware rewrites, previewed before apply. `ast_grep` is off by default. Turn it on when you want structural queries across 50+ grammars instead of regex:

```sh
omp config set astGrep.enabled true
```

### Approval mode

The product default `tools.approvalMode: yolo` auto-approves every tier, including shell. For professional work, use `write`: reads and workspace writes proceed, and exec still asks.

```sh
omp config set tools.approvalMode write
```

| Mode | Behavior |
| --- | --- |
| `always-ask` | Auto-approve read-only. Ask for the rest. |
| `always-ask-cached` | Also ask before file reads, and remember a file or folder. |
| `write` | Auto-approve read and workspace write. Ask for exec. |
| `yolo` | Auto-approve all tiers. |

Per-tool policy overrides the mode:

```yaml
tools:
  approvalMode: write
  approval:
    bash: prompt
    edit: allow
```

Each value is `allow`, `deny`, or `prompt`. `tools.restrictedFolders` still demands an explicit approval inside those folders, including under `yolo`.

One run can tighten or loosen without writing config:

```sh
omp --approval-mode write
omp --yolo
```

### Git guard for shared checkouts

Turn this on when more than one agent can see the same checkout. In the main checkout it refuses `git stash` (except `list`, `show`, `create`), `reset --hard`, `reset` onto another commit, branch switches, and working-tree restores that discard changes, unless a merge or rebase conflict is being resolved. In a linked worktree, discarded changes are saved to the `refs/omp/discarded` reflog first.

```sh
omp config set bash.gitGuard true
```

The guard is the embedded shell's `git` builtin. A git binary started by path, from a PTY, or from another program is outside it. Parallel implementation work still belongs in worktrees (§11). The hook below covers `git push --force` as well, including launches the git guard does not see.

### Force-push hook

Hooks are trusted TypeScript or JavaScript modules. omp loads every `.ts` and `.js` file directly inside `.omp/hooks/pre/`, `.omp/hooks/post/`, and the same folders under `~/.omp/agent/hooks/`. Discovery is not recursive. `pre` and `post` are folders, not event filters. Start a new session after adding a hook.

This is omp's documented guard. It blocks `git push --force` and a bare `-f`, and it leaves `--force-with-lease` alone. Put it in the user hooks directory so every repo gets it:

```ts
// ~/.omp/agent/hooks/pre/force-push.ts
import {
  isToolCallEventType,
  type ExtensionAPI,
} from "@oh-my-pi/pi-coding-agent";

const FORCE_PUSH = /\bgit\s+push\b[^\n]*(?:--force(?:\s|$)|(?:^|\s)-f(?:\s|$))/;

export default function forcePushGuard(pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    if (ctx.hasUI) ctx.ui.notify("Force-push guard active", "info");
  });

  pi.on("tool_call", (event) => {
    if (!isToolCallEventType("bash", event)) return;
    if (!FORCE_PUSH.test(event.input.command)) return;

    return {
      block: true,
      reason: "Blocked git push --force; use --force-with-lease after reviewing the remote branch.",
    };
  });
}
```

A project copy at `<repo>/.omp/hooks/pre/force-push.ts` applies when omp starts in that repo. Hook reference: https://omp.sh/docs/hooks

### Tool surface for a focused run

`--tools` limits built-ins for one process. Fewer tools means fewer mis-calls.

```sh
omp --tools read,edit,bash,lsp,debug,task --approval-mode write
```

`--no-tools` drops built-ins. Plugin tools can still load. Persistent on/off switches also live under `/settings` and as `*.enabled` keys (`bash.enabled`, `github.enabled`, `web_search.enabled`, `browser.enabled`, and the rest). `github.enabled` defaults to off and needs the `gh` CLI.

---

## 11. Subagents

Bundled specialists are already there. Use them before you write new ones.

| Name | Use |
| --- | --- |
| `scout` | Fast, read-only exploration |
| `designer` | UI implementation, accessibility, visual refinement |
| `reviewer` | Evidence-backed correctness and quality review |
| `security-reviewer` | Read-only vulnerability review |
| `librarian` | Source-verified research on external libraries and APIs |
| `task` | General multi-step work with the parent tool set |
| `sonic` | Mechanical edits and data collection |

`/agents` labels each definition Project, User, or Bundled. Names are case-sensitive. Project beats user, and a custom file beats the bundled agent of the same name. `omp agents unpack --user` writes the bundled files under `~/.omp/agent/agents` so you can edit them (`--project` writes `./.omp/agents`, `--force` overwrites).

### Custom specialists

Create `~/.omp/agent/agents/<name>.md` or `<repo>/.omp/agents/<name>.md`. Required frontmatter: `name` and `description`. The body is the durable instruction. An explicit `tools` list is the capability boundary. Omitting `tools` inherits the parent session, which is too wide for a reviewer.

Isolation is chosen per delegation, in the request. It is not a frontmatter field.

`api-reviewer.md`:

```markdown
---
name: api-reviewer
description: Review API changes for compatibility, missing tests, and contract drift.
tools: [read, grep, glob, lsp]
model: "@slow"
---

Review the assigned API change.

Check exported types, HTTP behavior, tests, and documented contracts.
Report only concrete findings with file and line references.
Severity levels: BLOCKER, WARNING, INFO.
Do not edit files.
```

`migration-runner.md`:

```markdown
---
name: migration-runner
description: Apply a database or framework migration and prove the rollback.
tools: [read, edit, bash]
model: "@default"
---

For the assigned migration:

1. Read the migration and its rollback.
2. Verify the rollback reverses the change.
3. Run the migration against the test database the project already uses.
4. Run the rollback and confirm the schema matches the pre-migration state.
5. Report success only after both directions have run.

Never mark a migration complete from reading it.
```

`test-writer.md`:

```markdown
---
name: test-writer
description: Write and run tests for one module, matching the project's existing style.
tools: [read, edit, bash, lsp]
model: "@default"
---

For the assigned module:

1. Read the implementation and the existing tests.
2. Identify uncovered branches and edge cases.
3. Write tests in the project's current style.
4. Run the project's test command for that module.
5. Report what passed and what you added.
```

Add `lsp` to any specialist that should navigate by symbol. Add `bash` only when it must run commands. Shell access can also modify the tree. Add `task` only when this specialist must delegate further, and set `spawns` to the agent names it may call. Granting `task` without `spawns` allows any discovered agent.

Other frontmatter you will actually use:

| Field | Meaning |
| --- | --- |
| `model` | Selector, list, or role alias such as `"@slow"`. Tried in order. |
| `thinking-level` | `inherit`, `off`, `auto`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max` |
| `spawns` | Agents this one may delegate to. `*` means any. |
| `autoload-skills` | Skill names loaded before the first assignment |
| `output` | JSON Schema when the parent should read a structured result |
| `prewalk` | `true` hands off to `smol` at the first edit. A selector chooses someone else. |
| `advisor` | `true` uses the configured advisor model for this agent |

Ask by name, and ask for a worktree when the worker will edit independently:

```text
Use the api-reviewer subagent on the branch diff. Report blockers only.

Use the migration-runner subagent in an isolated worktree to fix the failing migration test.
```

Parallel workers that might touch the same file are still safe to integrate: hashline rejects the second stale write, and that worker re-reads. Prefer worktrees when the tasks are truly independent, then review and merge from the parent session. The Agent Hub (`Ctrl+S` or `Alt+A`) can view a subagent mid-run.

`task.maxRecursionDepth` defaults to 2. `omp worktree list` shows managed worktrees. `omp worktree clear` removes them (`--dry-run` previews).

---

## 12. Connectors

Native tools first. MCP second, and only for a system omp does not already speak. omp also imports MCP servers, skills, and rules from Claude Code (`.claude/`), Cursor (`.cursor/`), Codex, Gemini CLI, OpenCode, Windsurf, and VS Code. Run `/mcp list` after the first launch in a repo that already has those files. Keep a server you use. Put the rest in `disabledServers`.

`/mcp disable <name>` writes that denylist when the definition lives in another tool's config. omp does not edit Claude's or Cursor's files.

### GitHub, native

The GitHub capability uses `gh`, so it sees the same repos, SSO, and branch protection as your account. PR checkout creates a worktree and does not switch the branch you have open. Reads and Actions watches are read-class. Creating a PR, pushing, and checking out are exec-class and follow `tools.approvalMode`.

```sh
gh auth login
gh auth status
omp config set github.enabled true
```

Start a new session after enabling it. `github.enabled` stays off until you set it, and the tool is hidden when `gh` is missing from `PATH`.

```text
Review PR 482 without checking it out.
Check out PR 482, run the tests, and report failures with file and line.
Watch the latest Actions run on this branch and tell me which job failed.
```

Leave the hosted GitHub MCP (`https://api.githubcopilot.com/mcp/`) disconnected while `github.enabled` is on. Two GitHub tool surfaces compete for the same request. If `/mcp list` shows a `github` server imported from Claude or Cursor, disable that name. Docs: https://omp.sh/docs/github

### Web search

`web_search.enabled` defaults to on. The tool follows `modelRoles.web` and `retry.fallbackChains.web` from §6. Exa is the coding search when its credential works. DuckDuckGo is the keyless fallback so a missing key still returns sources.

```sh
omp web-search --model web/duckduckgo "current Bun release"
omp web-search "site:docs.python.org pathlib read_text"
```

### Browser, native

`browser.enabled` defaults to on. Use it for public pages, local dev servers, and DOM checks. omp drops browser-automation MCP servers while that prelude is callable: servers named `playwright`, `puppeteer`, `browserbase`, `browser-tools`, `browser-use`, or `browser`, plus the usual Playwright, Puppeteer, Browserbase, and browser-use packages. A dropped server from Claude or Cursor never appears in `/mcp list`. Set `browser.enabled: false` only when you have a reason to run one of those servers instead. This setup does not.

Browser Relay is the logged-in path. It drives an existing Chrome tab, including that tab's cookies. That is the same account access you have. Install it once, then enable it:

```sh
omp browser-relay install
omp config set browser.relay true
```

In Chrome, open `chrome://extensions`, turn on Developer mode, and load `~/.omp/browser-relay/extension`. The relay starts when omp needs it. The extension badge reads on after it connects.

```text
Open the local app on port 3000, sign-in page through the relay tab,
and tell me which request returns 401.
```

Docs: https://omp.sh/docs/web

### Context7, every repo

Context7 and DeepWiki are the global MCP servers worth loading everywhere. Context7 returns current library documentation, so the agent writes against the API that ships now. DeepWiki returns how a public repository is put together.

Prefer OAuth so the Context7 key never sits in the file. Save this as `~/.omp/agent/mcp.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "context7": {
      "type": "http",
      "url": "https://mcp.context7.com/mcp/oauth"
    },
    "deepwiki": {
      "type": "http",
      "url": "https://mcp.deepwiki.com/mcp"
    }
  },
  "disabledServers": ["github", "playwright", "puppeteer", "browserbase"]
}
```

In a session, `/mcp reauth context7` (or the prompt omp shows on first use) completes login. Then:

```text
/mcp test context7
```

API-key alternative, if you already have a key from https://context7.com/dashboard. Export `CONTEXT7_API_KEY` in the shell that starts omp. omp expands `${VAR}` in native MCP files. An unset variable is left as the literal placeholder, which will fail the handshake, so export the key first. Keep the `deepwiki` entry from the file above when you switch only the Context7 block.

```json
{
  "mcpServers": {
    "context7": {
      "type": "http",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "Authorization": "Bearer ${CONTEXT7_API_KEY}"
      }
    }
  }
}
```

Ask the bundled `librarian`, and name the server:

```text
Use the librarian subagent and the Context7 server for the Next.js version in package.json.
Show the current API for the function I am about to call, with a source URL, before editing.
```

### DeepWiki, every repo

DeepWiki is already in the global file above. Public repositories need no key. The server exposes `read_wiki_structure`, `read_wiki_contents`, and `ask_question` for repositories indexed on DeepWiki. Docs: https://docs.devin.ai/work-with-devin/deepwiki-mcp

```text
/mcp test deepwiki
```

```text
Ask DeepWiki how the version of this dependency pinned in the lockfile is structured.
Read the wiki for the module I am about to change, then edit.
```

Context7 answers the current function signature. DeepWiki answers the layout of that upstream project. Use both when a dependency upgrade touches more than a call site.

Private repositories use a different endpoint and a Devin API key. Leave that off unless you need it. Export `DEVIN_API_KEY` in the shell that starts omp, and point `deepwiki` at `https://mcp.devin.ai/mcp` with header `Authorization: Bearer ${DEVIN_API_KEY}`. An unset variable stays as the literal placeholder and the handshake fails.

### Per-repo MCP

Put these in `<repo>/.omp/mcp.json`, not in the global file. A server you do not use still costs context. `stdio` is the default when `type` is omitted. Remote servers need `type` and `url`.

Linear, read-only. This endpoint exposes read tools only. The write endpoint is `https://mcp.linear.app/mcp`, and it can create and update issues. Stay on readonly until you want the agent to change the tracker.

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "linear": {
      "type": "http",
      "url": "https://mcp.linear.app/mcp/readonly"
    }
  }
}
```

Sentry, scoped to one project so discovery tools stay hidden. OAuth runs on first connect. Replace the slugs.

```json
{
  "mcpServers": {
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/mcp/ORG_SLUG/PROJECT_SLUG"
    }
  }
}
```

Postgres, restricted. [Postgres MCP Pro](https://github.com/crystaldba/postgres-mcp) (`uvx postgres-mcp`) runs SQL, `EXPLAIN`, and index checks. `--access-mode=restricted` is read-only and caps a query at 30 seconds. Point `DATABASE_URI` at a dev database. The value `"DATABASE_URI"` copies that variable from the environment at connect time. Do not paste a URI into the file.

```json
{
  "mcpServers": {
    "postgres": {
      "command": "uvx",
      "args": ["postgres-mcp", "--access-mode=restricted"],
      "env": {
        "DATABASE_URI": "DATABASE_URI"
      }
    }
  }
}
```

```sh
export DATABASE_URI="postgresql://user:pass@localhost:5432/devdb"
```

Unrestricted mode can change data and schema. Use it only on a database you can rebuild, and only by changing the flag on purpose.

Figma, only when this repo implements from a Figma file. The remote server is `https://mcp.figma.com/mcp`. Sign in with `/mcp reauth figma`. Figma accepts clients listed in its MCP catalog, and omp is not in that catalog, so the handshake can be refused. Run `/mcp test figma` before you depend on it. If the test fails, remove the server. A dead Figma entry still costs context.

```json
{
  "mcpServers": {
    "figma": {
      "type": "http",
      "url": "https://mcp.figma.com/mcp"
    }
  }
}
```

```text
Read the Figma frame for this screen, then implement the layout to match it.
```

### What to leave disconnected

| Server | Why it stays off |
| --- | --- |
| Filesystem MCP | `read`, `grep`, and `glob` already cover the workspace. A filesystem server runs as your user on whatever roots you pass. |
| Playwright, Puppeteer, Browserbase, browser-use | omp's browser and Browser Relay cover this. Those servers are dropped while `browser.enabled` is true. |
| GitHub MCP next to `github.enabled` | Duplicate tools for issues, PRs, and files. |
| A memory MCP | `mnemopi` is the memory store. A second memory server splits recall. |
| Every Claude or Cursor server you imported | Keep the ones in the table above. Disable the rest by name. |

### Secrets and checks

omp expands `${VAR}` and `${VAR:-default}` in `command`, `args`, `env`, `cwd`, `url`, `headers`, `auth`, and `oauth` for its own MCP files. A header or env value that is exactly an environment-variable name is replaced with that variable's value. A value that starts with `!` runs as a shell command and uses its stdout, which is how you build `Authorization` without writing the token down.

Do not commit tokens. Prefer OAuth (`/mcp reauth <name>`) or an env var. A `stdio` server is a program with your permissions. Read project `.omp/mcp.json` before you trust a fresh clone. Because OAuth credentials are stored per profile and server URL, an untrusted checkout that names the same URL can use that credential. Open untrusted repos under `omp --profile untrusted`.

After an edit while omp is running:

```text
/mcp reload
/mcp list
/mcp test context7
```

A successful test shows the server name, version, tool count, and up to ten tool names. Set `"instructions": false` on a server whose own guidance fights yours. That field is honored on omp-native files.

### How a task should use the stack

```text
Implement the issue. Read it from Linear first.
Before calling any third-party API, pull the current docs from Context7 and cite the URL.
When the change depends on how a public dependency is structured, ask DeepWiki and name the repository.
When this repo has a working Figma server, read the frame before editing the UI.
Use LSP for the rename. Preview it, then apply it. Apply the server's quick fix and read diagnostics again, including YAML, shell, Terraform, Dockerfiles, and Markdown when those files change. Run the tests.
If a test fails for a reason you cannot see statically, debug it.
If the failure is in production, read the matching Sentry issue and, for a data question, query Postgres in restricted mode.
Then have the reviewer subagent check the diff, and open the PR with the GitHub tool.
```

---

## 13. Memory and autolearn

Memory is off by default. Compaction shortens one session. Memory is what later sessions can see. Pick one backend.

| Backend | Use it when | Trade-off |
| --- | --- | --- |
| `off` | You want no cross-session knowledge | Transcripts under `~/.omp/agent/sessions/` remain. Turning memory off does not delete them. |
| `mnemopi` | You want remember, recall, and forget on this machine | Local SQLite, per-project by default. Default `llmMode: smol` sends extraction to your `tiny` or `smol` model. |
| `local` | You want periodic project summaries distilled from past sessions | A batch summary pipeline. You cannot review or forget one fact. Extraction still uses your configured model roles. |
| `hindsight` | You already run a Hindsight server | `autoRetain` sends conversation text to that server. Tool calls and thinking blocks are omitted. |

Recommended start:

```sh
omp config set memory.backend mnemopi
```

```yaml
memory:
  backend: mnemopi
```

Privacy-oriented variant, with no LLM extraction and no embedding download:

```yaml
memory:
  backend: mnemopi
mnemopi:
  scoping: per-project
  noEmbeddings: true
  llmMode: none
```

`local` writes generated files under `~/.omp/agent/memories/<project>/`: `memory_summary.md`, `MEMORY.md`, `learned.md`, and `skills/`. Later consolidation can replace hand edits. Clear and rebuild instead of treating those files as source.

Inspect with `/memory`. `recall`, `retain`, `reflect`, and `memory_edit` appear once the matching backend is active.

### Autolearn

Experimental. With a memory backend active, `autolearn.enabled` exposes lesson capture (`learn`) and managed-skill maintenance (`manage_skill`).

```yaml
autolearn:
  enabled: true
  minToolCalls: 5
  autoContinue: false
```

`autoContinue: false` keeps standing guidance and does not spend a hidden capture turn. Set `autoContinue: true` only when you want omp, after a primary turn with at least `minToolCalls` completed tool calls, to run a private capture turn and write managed skills. Aborted turns, plan mode, and goal-loop turns are skipped. Managed skills live in `~/.omp/agent/managed-skills`. Leave that directory to the tool.

`learn` requires both `autolearn.enabled` and a `local`, `hindsight`, or `mnemopi` backend.

---

## 14. Skills and project instructions

A skill is a directory containing `SKILL.md`. omp advertises the name and description at session start and loads the body when the request matches. Discovery is one directory level deep: `skills/pre-merge/SKILL.md` is found, `skills/checklists/pre-merge/SKILL.md` is not.

```text
~/.omp/agent/skills/<name>/SKILL.md
<repo>/.omp/skills/<name>/SKILL.md
```

omp also discovers skills from `~/.claude/skills`, `~/.codex/skills`, `.claude/skills`, `.github/skills`, and the other compatible paths listed in the skills doc. `/extensions` shows which copy won when names collide. Read a third-party `SKILL.md` before you rely on it. A playbook can tell the agent to run commands with the session's existing permissions.

The frontmatter field that matters is `description`. Write it as the condition for loading the skill.

`<repo>/.omp/skills/pre-merge/SKILL.md`:

```markdown
---
description: Use before merging to the default branch. Run the project's checks and report a pass/fail checklist.
---

# Pre-merge verification

Use the project's own commands, in this order. Stop on the first failure.

1. Typecheck.
2. Lint.
3. Tests for the change, then the full suite when the change is wide.
4. Production build, when the repo has one.
5. Changelog or release-note entry, when the repo requires one.
6. Scan the diff for leftover debug output.
7. Confirm the branch is rebased on the latest default branch.

Report pass or fail for each step, with the command you ran.
```

`<repo>/.omp/skills/rollback/SKILL.md`:

```markdown
---
description: Use for an emergency rollback of a bad deploy. Identify the last good release, confirm it, revert, and watch health.
---

# Rollback

Replace the command examples with this project's deploy tooling before relying on this skill.

1. Identify the last known-good commit or release tag.
2. Confirm the rollback target with the on-call owner.
3. Revert the deploy with the project's documented command.
4. Verify health checks and error rate for an agreed window.
5. Open an incident record with the timeline.

Do not skip confirmation. Do not skip the health window.
```

Author the next skill the second time you walk a checklist by hand: release, incident, migration review, dependency bump.

`<repo>/.omp/skills/verify-change/SKILL.md` is the skill that makes the harness stricter than a single model that declares victory:

```markdown
---
description: Use when implementing or fixing code. Finish only after language-server diagnostics, tests, and a review.
---

# Verify a change

1. Read the relevant ticket from Linear when this repo has that server. Otherwise use the issue the user named.
2. Before calling a third-party library API, load its current docs from Context7 and keep the source URL.
3. When the change depends on how a public dependency is structured, ask DeepWiki and keep the repository name. When this repo's Figma server passes `/mcp test figma`, read the frame before editing the UI.
4. Navigate with the language server. Preview a rename before applying it. Move a file through the language server so imports update. Apply the server's quick fix, then read diagnostics again, including YAML, shell, Terraform, Dockerfiles, and Markdown when those files change.
5. Run the project's test command for the files you changed.
6. If the failure depends on a runtime value, use the debugger instead of adding print statements.
7. Read diagnostics for every file you edited.
8. Run `semgrep scan --config auto` when Semgrep is installed, and report findings that touch the diff.
9. Ask the reviewer subagent to check the diff.
10. Report the commands, the diagnostic result, and any blocker. A green claim requires a command you actually ran.
```

### AGENTS.md

Put repository rules in `AGENTS.md` at the repo root: how to build, how to test, which directories are generated, what must not be edited, and how commits are written. omp loads discovered instruction files into context. Keep the file short. Stable policy belongs there. Repeatable procedures belong in skills.

---

## 15. Compaction, context, and spend

The defaults are already tuned for large windows: compaction on, recent tail of 20,000 tokens, automatic continue, and a reserve-based threshold (`thresholdPercent: -1`) that lands near 85% on windows above roughly 109k tokens. Writing `thresholdPercent: 85` replaces that reserve policy with a flat percentage. Do that when you have measured a problem, not as part of first-time setup.

Override per model when one provider's window should compact earlier:

```yaml
compaction:
  modelThresholds:
    "deepseek/*": "80%"
```

A bare token count is a base the normal policy scales. `"f400000"` is an exact trigger at 400k. `"80%"` is a percentage of the real window. In `/model` Roles view, select a row and press `k` to set this from the picker.

`/extended-context on` opts the session into a model's larger window, including that window's pricing. Leave it off until a task needs it.

Spend:

```sh
omp stats --summary
omp stats
```

`omp stats` opens the usage dashboard. Rebalance `task` and `advisor.reviewMode` from that report, not from a guess on day one.

Background jobs are on (`async.enabled`, cap `async.maxJobs` 100). Long shell commands auto-background after `bash.autoBackground.thresholdMs` (60 seconds). `launch.enabled` lets omp supervise named long-running processes through `proc://`.

---

## 16. Always-on coverage

This machine stays on. The steps below are the open-source stand-ins for the Claude Code surfaces that can be covered that way. An MCP server can act only inside a session that is already running. Timers and a self-hosted GitHub runner are what start work on their own.

Unattended jobs use a second config file so they cannot drive the desktop or the browser:

```yaml
# ~/.omp/unattended.yml
tools:
  approvalMode: write
  approval:
    computer: deny
    browser: deny
```

Run one by hand before you schedule it:

```sh
cd /path/to/repo
omp --config ~/.omp/unattended.yml -p --approval-mode write \
  "Review what changed on the default branch today. Write the report to .omp/reports/daily.md"
```

`omp -p` prints the answer and exits. Ollama must already be running.

### Editor

An editor that speaks the Agent Client Protocol hosts omp. omp keeps the local models, tools, and settings. Zed and JetBrains speak ACP. VS Code needs a client from https://agentclientprotocol.com/get-started/clients.

Confirm the binary the editor will see:

```sh
command -v omp
```

Zed: Agent Panel, Add Custom Agent, then:

```json
{
  "agent_servers": {
    "Oh My Pi": {
      "type": "custom",
      "command": "omp",
      "args": ["acp"]
    }
  }
}
```

Use the absolute path from `command -v omp` when a GUI-launched editor does not inherit your shell `PATH`.

JetBrains (the AI Assistant chat, no JetBrains AI subscription required): Add Custom Agent writes `~/.jetbrains/acp.json`.

```json
{
  "agent_servers": {
    "Oh My Pi": {
      "command": "/absolute/path/to/omp",
      "args": ["acp"]
    }
  }
}
```

VS Code: install an ACP client, then set its agent command to `omp` and its arguments to `acp`. Transport is stdio. The session directory is the project root. Docs: https://omp.sh/docs/acp

### Sandbox for unattended runs

Interactive sessions stay on your user account. Timers and the GitHub runner run as a separate account, `ompagent`, so a scheduled shell cannot read the rest of the home directory.

1. Create that user and install omp for it.
2. Give it its own `~/.omp` (the local config from §20) and clones under its home.
3. Let it reach Ollama on `127.0.0.1:11434`. Do not copy your SSH keys, browser profile, or password store into that account.
4. Point cron, launchd, systemd, and the GitHub runner at that user.
5. Pass `~/.omp/unattended.yml` on every unattended command.

On Linux, bubblewrap or a container can wrap that same command. On macOS or Linux, a devcontainer can wrap it instead. Mount only the clone and that user's `.omp`. Point Ollama at the host:

```json
{
  "name": "omp",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "runArgs": ["--add-host=host.docker.internal:host-gateway"],
  "remoteEnv": {
    "OLLAMA_BASE_URL": "http://host.docker.internal:11434",
    "OLLAMA_CONTEXT_LENGTH": "65536"
  }
}
```

Install omp inside the container the same way as on the host. Do not mount your interactive `~/.omp` into the container. That directory holds GitHub and MCP credentials.

### Dictation

Local speech-to-text. This is not a hosted voice service. Leave `/live` unused. That command calls a hosted realtime voice path.

```sh
omp models --kind stt
omp config set stt.enabled true
omp config set stt.language en
omp config set stt.submitTrigger release
```

Add this to `modelRoles` only after `omp models --kind stt` lists the model:

```yaml
dictation: local/parakeet-tdt-0.6b-v3
```

```yaml
retry:
  fallbackChains:
    dictation: []
```

`release` submits the prompt when you finish speaking. `never` leaves the text in the composer.

### Checkpoints

```sh
omp config set checkpoint.enabled true
```

This turns on checkpoints and rewind together. In a session you can return the conversation, the files, or both to an earlier point.

### Desktop control

Off by default, and denied in `unattended.yml`. Turn it on only for an interactive session you are watching:

```sh
omp config set computer.enabled true
```

Grant the OS accessibility permission it asks for. `/computer off` disables it for the current session. The browser relay from §12 remains the way to drive a logged-in Chrome tab.

### A daily job

macOS, with the machine awake. Save `~/Library/LaunchAgents/local.omp.daily-review.plist`. Replace the paths with `command -v omp` and the real repo.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.omp.daily-review</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/zsh</string>
    <string>-lc</string>
    <string>cd /path/to/repo && omp --config ~/.omp/unattended.yml -p --approval-mode write "Review what landed on the default branch since yesterday. Write the report to .omp/reports/daily.md"</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>7</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
  <key>StandardOutPath</key>
  <string>/tmp/omp-daily-review.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/omp-daily-review.err</string>
</dict>
</plist>
```

```sh
launchctl load ~/Library/LaunchAgents/local.omp.daily-review.plist
```

Linux, as the unattended user:

```ini
# ~/.config/systemd/user/omp-daily-review.service
[Service]
Type=oneshot
WorkingDirectory=/path/to/repo
ExecStart=/absolute/path/to/omp --config %h/.omp/unattended.yml -p --approval-mode write "Review what landed on the default branch since yesterday. Write the report to .omp/reports/daily.md"
```

```ini
# ~/.config/systemd/user/omp-daily-review.timer
[Timer]
OnCalendar=*-*-* 07:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```sh
systemctl --user enable --now omp-daily-review.timer
```

`/loop` repeats a prompt inside a session you already have open. A timer is what runs when no terminal is attached.

Keep the interactive session alive across SSH disconnects:

```sh
tmux new -s omp
cd /path/to/repo
omp
```

From a phone on the same network, or on Tailscale, run `/collab` and open the browser link, or SSH in and attach with `tmux attach -t omp`.

### Pull-request review on this machine

The runner has to be this computer. A GitHub-hosted runner cannot see Ollama.

1. In the repository, open Settings, Actions, Runners, and add a self-hosted runner.
2. Install and run it as `ompagent`, with Ollama already running on that machine.
3. Add `.github/workflows/omp-review.yml`:

```yaml
name: omp review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - name: Review
        run: |
          omp --config "$HOME/.omp/unattended.yml" -p --approval-mode write \
            "Review the pull request diff. Write a markdown review to review.md. Do not push."
      - name: Comment
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh pr comment "${{ github.event.pull_request.number }}" --body-file review.md
```

The `if` skips forks. A self-hosted runner that executes fork code can read the machine. This posts a review comment. It is not Anthropic's GitHub App.

### Slack, from a session that is already open

Add Slack's server to `~/.omp/agent/mcp.json`, then `/mcp reauth slack`:

```json
{
  "mcpServers": {
    "slack": {
      "type": "http",
      "url": "https://mcp.slack.com/mcp"
    }
  }
}
```

```text
/mcp test slack
```

The agent can then search Slack and send a message. A Slack message does not start omp. When a mention should start work, a Slack app you host on this machine should run the same `omp -p` command the timer uses.

### Security review with Semgrep

Install the official Semgrep CLI. The old standalone `semgrep/mcp` repository is deprecated.

```sh
brew install semgrep
semgrep scan --config auto
```

On Linux: `pipx install semgrep`. Ask the bundled `security-reviewer` to run that scan and report only findings that match the diff.

---

## 17. First session

From a real repository:

```sh
cd /path/to/repo
gh auth status
omp --approval-mode write
```

1. `/model`, Roles view. Every role you set resolves, including `web`, and each fallback row resolves.
2. `omp web-search --model web/duckduckgo "current Bun release"` returns sources. Skip this when you set `web_search.enabled` to false.
3. Ask for a definition, then a previewed rename of a local symbol. The language server does the rename. After a hashline edit, diagnostics come back without a full file rewrite.
4. Ask for a breakpoint and one step in a small program whose adapter you installed. Name `lldb-dap` or `codelldb` for a Rust binary. For Node, break in a `.ts` file only after the source-map build from §9.
5. `/mcp test context7` and `/mcp test deepwiki`, then ask for the current docs of one dependency and, when the task needs the upstream layout, ask DeepWiki before editing it.
6. Ask omp to review an open PR with the GitHub tool, without checking it out.
7. Ask the bundled `reviewer` to review the current diff.
8. Make an edit and confirm an advisor card.
9. `/mcp list`. The connected set is Context7 and DeepWiki, plus Linear, Sentry, Postgres, or Figma only when this repo uses them. Imported GitHub and browser servers are disabled. Figma stays only after `/mcp test figma` succeeds.
10. `/memory` and confirm the backend.
11. Start a new session and confirm the force-push guard notification.

A comparison task against Claude Code on the same repo: a bug whose cause is a runtime value, plus a dependency API that has changed since the model's training cutoff, plus a pull request to open. omp should rename through the language server, stop the process in the debugger, cite a Context7 URL, take the advisor's note, and open the PR through `gh` into a worktree. Claude Code can write the patch. This harness can show the value, the current API, and the review before the patch lands.

Continue the same project later with `omp -c`. Pick another session with `omp -r`. Share the live session with `/collab` when a teammate should watch or steer it. Guests get the transcript. Mutations stay on the host. Docs: https://omp.sh/docs/collab

---

## 18. Verification checklist

```sh
omp --version
omp models
omp config get lsp.enabled          # true
omp config get lsp.diagnosticsOnEdit # true
omp config get lsp.requestTimeout    # 10
omp config get debug.enabled        # true
omp config get edit.mode            # hashline
omp config get advisor.enabled      # true
omp config get memory.backend       # mnemopi, or the backend you chose
omp config get autolearn.enabled    # true, if you opted in
omp config get tools.approvalMode   # write, if you followed §10
omp config get astGrep.enabled      # true, if you opted in
omp config get bash.gitGuard        # true, if agents share a checkout
omp config get github.enabled        # true
omp config get browser.enabled       # true
omp config get web_search.enabled    # true
omp config get stt.enabled           # true
omp config get checkpoint.enabled    # true
```

- [ ] `omp models ollama` lists the coder you assigned, and every chat role resolves to `ollama/`, `llama.cpp/`, `lm-studio/`, `local/`, or `apple/`.
- [ ] The §6 probe created `hello.txt` containing `ready`. A transcript that only describes the write does not count.
- [ ] `omp web-search --model web/duckduckgo "current Bun release"` returns sources, unless web search is off.
- [ ] `gh auth status` is logged in, and a new session exposes the GitHub tool.
- [ ] Startup prints no warning about an unknown fallback entry.
- [ ] `command -v` finds each language server and debug adapter you depend on, including `yaml-language-server`, `bash-language-server`, `terraform-ls`, `docker-langserver`, and `marksman` when you edit those files.
- [ ] `~/.omp/agent/config.yml` contains `modelRoles`, `retry.fallbackChains`, `advisor.enabled: true`, and `github.enabled: true`.
- [ ] `/mcp test context7` and `/mcp test deepwiki` succeed.
- [ ] `/mcp list` has no GitHub MCP beside the native GitHub tool, and no Playwright or Puppeteer server you meant to keep.
- [ ] Linear, Sentry, Postgres, and Figma appear only in repos that use them. Postgres is `--access-mode=restricted` and its URI is not in the file. Figma remains only after `/mcp test figma` succeeds.
- [ ] `/agents` lists the bundled specialists plus any custom files, enabled, with the path you expect.
- [ ] `verify-change` and `pre-merge` skills exist.
- [ ] `~/.omp/agent/hooks/pre/force-push.ts` exists, and a new session reports the guard.
- [ ] `AGENTS.md` exists in repositories that have non-obvious commands or forbidden paths.
- [ ] One live task has used LSP, one has used the debugger, one has used Context7, one has used a subagent, and one edit went through hashline.
- [ ] After a week, `omp stats --summary` still fits the budget you intended for `task` and `advisor`.
- [ ] An ACP editor starts a thread with `omp acp` and the local coder.
- [ ] `omp --config ~/.omp/unattended.yml -p "Reply with the word ready"` prints `ready` and exits.
- [ ] The daily timer is loaded (`launchctl list` or `systemctl --user list-timers`).
- [ ] A test pull request on the self-hosted runner receives the review comment.

---

## 19. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Roles fall back or show as unset | `modelRoles` missing, invalid YAML, or the selector is not available | `omp config list`. Fix the selector to one `omp models` can call. Read startup warnings. |
| Advisor never appears | `advisor.enabled` is false, or `modelRoles.advisor` does not resolve | `omp config get advisor.enabled`. An explicit unavailable advisor is not replaced automatically. |
| LSP tool missing | `lsp.enabled` false, or this run used `--no-lsp` | `omp config get lsp.enabled`. Start a session without `--no-lsp`. |
| Rename or diagnostics do nothing | Binary not on `PATH`, or root markers do not match | `command -v` the server. Start a new session. Check `.omp/lsp.json` for `"disabled": true`. |
| YAML, shell, Terraform, Docker, or Markdown diagnostics do nothing | The binary is missing, or the marker is not in the launch directory | `command -v` `yaml-language-server`, `bash-language-server`, `terraform-ls`, `docker-langserver`, or `marksman`. Terraform and Docker markers are checked only in the directory where omp started. |
| Two Python servers fight | `pyright`, `basedpyright`, `ty`, and `pylsp` can all match one repo | Keep one type server. Disable the others in `.omp/lsp.json` as in §8. |
| Two TypeScript servers fight | `typescript-language-server` and `typescript-native` both match `package.json` | Disable `typescript-native` on TypeScript 6. Disable `typescript-language-server` when `tsc --lsp` is the server you want. |
| Hashline edit gets no diagnostics | `lsp.diagnosticsOnEdit` is false | `omp config set lsp.diagnosticsOnEdit true`. `diagnosticsOnWrite` covers a full file write. |
| clangd diagnostics are empty or wrong | No compilation database at the launch directory | Generate `compile_commands.json` and link it at the repo root, as in §8. |
| Debug tool missing | `debug.enabled` false | `omp config set debug.enabled true`, then start a new session. |
| Adapter unavailable | Binary not visible to the omp process | `command -v` from that shell. For debugpy, `python -m debugpy --version`. Set `command` in `dap.json` when the binary is off `PATH`. |
| Wrong adapter chosen | Extension and root markers tie | Name the adapter in the request, or fix `fileTypes` and `rootMarkers`. |
| Breakpoint never hits | Source does not match the loaded binary, the condition language is wrong, or Node is running emitted JavaScript without a source map | Rebuild with debug symbols (`cargo build`, or CMake `Debug`). Name `lldb-dap` or `codelldb` for a Rust binary. For TypeScript, set `sourceMaps` and `outFiles` on `js-debug-adapter` and launch after that build. A condition is an expression in the target language. |
| Attach fails | Dead pid, unreachable port, or unnamed adapter | Include pid or host:port and the adapter name. |
| "Another debug session" | One root session is still active | Ask omp to terminate it, then retry. |
| Hashline patch rejected | File changed since the read | Re-read and edit. The rejection is expected. |
| The machine swaps models constantly | Advisor on every turn is a second large local model, or `task` points at a different large model | Set `advisor` to `@default` and `advisor.reviewMode` to `agent-end`. Keep `task` on `@default`. |
| The transcript shows a JSON `write` and the file was never created | The model printed a tool call as chat text. `tool_calls` was empty | The coder cannot drive the loop. `qwen2.5-coder` does this. Run the probe in §6 and assign a model that returns `tool_calls`. Leave `tools.format` at `auto`. |
| A turn is only a thinking block, or the response stream closes before a terminal event | Qwen 3 planned inside the thinking channel. `--thinking off` does not set Ollama's think flag | Keep `defaultThinkingLevel: low`. Limit the run to `--tools write,read,bash` and ask for one `write`. omp can continue the same session and call the tool on the next turn. Do not force `/no_think` in the Modelfile, and do not replace the built-in Ollama provider in `models.yml`. |
| `ollama ps` shows about 14 GB on a 24 GB Mac | `num_ctx` is 32768 for a 14B model | Create the model at `16384` and set `OLLAMA_CONTEXT_LENGTH=16384`. |
| HTTP 500 from Ollama right after a tool-format change | `tools.format: hermes` with the full tool list overflowed the request | `omp config set tools.format auto`. |
| Edit says the payload has no hunk header | The model sent a bare `PUT` instead of a hashline patch | Re-read, then edit with `[path#HASH]`, or `write` the whole file. |
| 429 stays on the same model | `retry.rateLimitWalksChain` is false | Expected until `retry.maxRetries` is spent. Set `retry.rateLimitWalksChain: true` to walk immediately. |
| A timer or runner cannot see Ollama | The job's environment has no `OLLAMA_BASE_URL`, or Ollama is not running | Start Ollama before the job. In Docker, use `http://host.docker.internal:11434`. |
| Dictation does nothing | `stt.enabled` is false, or `modelRoles.dictation` is not an `stt` model | `omp models --kind stt`, then set `dictation` to a listed id. |
| Fallback never runs | Unknown model in the chain, or `retry.modelFallback: false` | Startup warnings. `omp config get retry.modelFallback`. |
| Subagent missing | Wrong directory, or frontmatter has no `name` and `description` | `~/.omp/agent/agents/*.md` or `<repo>/.omp/agents/*.md`. Project file wins on a name clash. |
| Skill never loads | Nested more than one level, or the description does not match the request | `skills/<name>/SKILL.md`. Name the skill in the request once to confirm it loads. |
| Autolearn writes nothing | `autoContinue` is false, or the turn had fewer than `minToolCalls` tool calls | Expected until you set `autolearn.autoContinue: true`. Confirm a memory backend is on, or `learn` stays unavailable. |
| Imported MCP servers appear | Discovery from Claude, Cursor, Codex, and others | `/mcp list`. Add names to `disabledServers`, or `/mcp disable <name>`. |
| GitHub tool missing | `github.enabled` is false, or `gh` is not on `PATH` | `gh auth status`, then `omp config set github.enabled true`. Start a new session. |
| Context7 test fails | OAuth not finished, or `CONTEXT7_API_KEY` unset while the header still contains `${CONTEXT7_API_KEY}` | `/mcp reauth context7`, or export the key and restart omp. |
| DeepWiki test fails | The URL is not `https://mcp.deepwiki.com/mcp`, or the repository is private | Public repos use that URL with no key. Private repos use `https://mcp.devin.ai/mcp` and `DEVIN_API_KEY`. |
| Figma test fails | OAuth did not finish, or Figma refused the client | `/mcp reauth figma`. Figma allows catalog clients, and omp is not listed. If the test still fails, remove `figma` from `<repo>/.omp/mcp.json`. |
| Playwright MCP never connects | `browser.enabled` is true, so omp drops those servers | Use the native browser and Browser Relay. |
| Browser Relay does nothing | Extension not loaded, or `browser.relay` is false | `omp browser-relay install`, load `~/.omp/browser-relay/extension`, `omp config set browser.relay true`. |
| Web search has no provider | `modelRoles.web` points at a provider with no credential and the chain has no keyless entry | Set `modelRoles.web` to `web/duckduckgo` or add that entry under `retry.fallbackChains.web`. |
| Postgres MCP fails at launch | `uvx` missing, or `DATABASE_URI` unset | Install `uv`. Export `DATABASE_URI` to a dev database. Keep `--access-mode=restricted`. |
| Settings file rejected on startup | Invalid YAML or a non-mapping root | omp moves it to a `.broken-*` backup and exits with that path. |

---

## 20. Complete global config

Save as `~/.omp/agent/config.yml` on a 32 GB machine where `qwen3-coder:30b` and Devstral both stay resident. Replace every model id with one `omp models` lists as available. On 24 GB, use the `qwen3-16k` block in §6 instead of this file: one model, `defaultThinkingLevel: low`, and `advisor.reviewMode: agent-end`. This file sets only the values this guide changes. Everything in §2 stays at the built-in default on purpose, except `defaultThinkingLevel` on a 16k or 32k window.

```yaml
# ~/.omp/agent/config.yml
# Every chat selector must appear in: omp models ollama
# Hosted providers are disabled. See the optional note in §6 to opt in.

modelRoles:
  default: ollama/qwen3-coder-64k
  slow: ollama/qwen3-coder-64k:high
  plan: "@slow"
  smol: "@default"
  commit: "@smol"
  task: "@default"
  advisor: ollama/devstral-64k
  tiny: local/lfm2.5-230m
  memory: local/lfm2-1.2b
  web: web/duckduckgo
  # Set only after `omp models --kind stt` lists this id.
  dictation: local/parakeet-tdt-0.6b-v3

cycleOrder:
  - smol
  - default
  - slow

retry:
  fallbackChains:
    default:
      - ollama/devstral-64k
    slow:
      - ollama/qwen3-coder-64k
    smol:
      - local/lfm2-1.2b
    advisor:
      - ollama/qwen3-coder-64k
    web: []
    dictation: []

disabledProviders:
  - anthropic
  - openai
  - openai-codex
  - google
  - google-vertex
  - google-antigravity
  - openrouter
  - groq
  - xai
  - deepseek
  - mistral
  - github-copilot
  - cursor

advisor:
  enabled: true
  reviewMode: turn
  reviewInterval: 1
  continueSeverity: concern
  maxNotesPerUpdate: 4
  syncBacklog: "off"
  immuneTurns: 3

memory:
  backend: mnemopi

autolearn:
  enabled: true
  minToolCalls: 5
  autoContinue: false

tools:
  approvalMode: write

bash:
  gitGuard: true

astGrep:
  enabled: true

lsp:
  diagnosticsOnEdit: true
  requestTimeout: 10

github:
  enabled: true

# Install the relay extension before relying on this. It drives a logged-in Chrome tab.
browser:
  relay: true

stt:
  enabled: true
  language: en
  submitTrigger: release

checkpoint:
  enabled: true
```

Single-model and 64 GB variants are in §6. Replace a selector that `omp models ollama` does not list. Drop `:high` when the local server has no thinking ladder.

---

## 21. Directory layout

```text
~/.omp/
├── agent/
│   ├── config.yml
│   ├── models.yml
│   ├── mcp.json                 # Context7, DeepWiki, Slack, and disabledServers
│   ├── lsp.json                 # optional personal overrides
│   ├── dap.json                 # optional personal overrides
│   ├── hooks/
│   │   └── pre/
│   │       └── force-push.ts
│   ├── agents/
│   │   ├── api-reviewer.md
│   │   ├── migration-runner.md
│   │   └── test-writer.md
│   ├── skills/
│   │   └── <name>/SKILL.md
│   ├── managed-skills/          # autolearn output
│   ├── memories/                # mnemopi or local stores
│   └── sessions/
├── unattended.yml               # denies computer and browser for timers and CI
└── profiles/<name>/agent/       # omp --profile <name>

<repo>/
├── AGENTS.md
├── .github/
│   └── workflows/
│       └── omp-review.yml       # self-hosted runner, same machine
└── .omp/
    ├── config.yml               # project overrides only
    ├── mcp.json                 # Linear, Sentry, Postgres, Figma — only when this repo uses them
    ├── lsp.json
    ├── dap.json
    ├── hooks/
    │   └── pre/                 # optional project hooks
    ├── agents/
    └── skills/
        ├── pre-merge/SKILL.md
        ├── rollback/SKILL.md
        └── verify-change/SKILL.md
```

---

## 22. Master setup prompt

Paste this to an assistant that can edit files and run commands on the machine where omp is installed. It applies the configuration in this guide and stops to verify after each block.

````markdown
You are configuring omp (Oh My Pi) on this machine. Follow this brief exactly. Do not invent setting names. After each block, verify with `omp config get`, `omp models`, or the slash command named below. Every chat role must be a model running on this machine (`ollama/`, `llama.cpp/`, `lm-studio/`, `local/`, or `apple/`). If `omp models ollama` does not list a selector, stop and ask which local id to use. Do not assign a hosted model unless the user explicitly opts into the optional hosted note in the setup guide.

Paths:
- Global settings: `~/.omp/agent/config.yml` (if `config.yaml` already exists, edit that file in place)
- Project overrides: `<repo>/.omp/config.yml`
- Custom providers: `~/.omp/agent/models.yml`, root key `providers` only
- User agents: `~/.omp/agent/agents/*.md`
- Project agents: `<repo>/.omp/agents/*.md`
- User skills: `~/.omp/agent/skills/<name>/SKILL.md`
- Project skills: `<repo>/.omp/skills/<name>/SKILL.md`
- MCP, user: `~/.omp/agent/mcp.json` (Context7 and DeepWiki)
- MCP, project: `<repo>/.omp/mcp.json` (Linear, Sentry, Postgres, Figma only when used)
- Hooks: `~/.omp/agent/hooks/pre/*.ts` and `<repo>/.omp/hooks/pre/*.ts`
- LSP overrides: `<repo>/.omp/lsp.json` with a top-level `servers` object
- DAP overrides: `<repo>/.omp/dap.json` with a top-level `adapters` object

1. Install and authenticate.
   - `omp --version` must succeed.
   - Install and start Ollama (or llama.cpp, or LM Studio). Pull the local coders from §6 and raise `num_ctx` to at least 32768, 65536 on a 32 GB machine.
   - `omp models ollama` must list those models before you write roles. Do not assign a hosted model.
   - On 24 GB, pull `qwen3:14b`, create `qwen3-16k` with `num_ctx` 16384, and set `OLLAMA_CONTEXT_LENGTH=16384`. Do not assign `qwen2.5-coder` to any role that calls tools.
   - Run the §6 probe. `hello.txt` must exist before you continue.

2. Model roles and fallbacks in the global config.
   - Set `modelRoles` for `default`, `slow`, `plan`, `smol`, `commit`, `task`, `advisor`, `tiny`, `memory`, and `web` using only selectors from `omp models ollama`, `omp models llama.cpp`, `omp models lm-studio`, `omp tiny-models list`, or `web/duckduckgo`.
   - The default coder is a local model that passed the tool-call probe: Qwen3-Coder on 32 GB, `qwen3-16k` on 24 GB, or gpt-oss 20B on 16 GB. The advisor is local Devstral when it fits, otherwise `"@default"`. `task` and `smol` are `"@default"` until a smaller model passes the same probe.
   - Set `retry.fallbackChains` to other local models only. Set `retry.fallbackChains.web` to `[]`.
   - Set `disabledProviders` to the hosted list in §6, including `anthropic`, `openai`, `openai-codex`, `google`, and `openrouter`.
   - Verify: `omp models ollama`, `omp web-search --model web/duckduckgo "current Bun release"`, then `/model` → Roles. Every chat role is local. Startup warnings about chains are fixed before continuing.

3. Advisor.
   - Set `advisor.enabled: true`, `advisor.reviewInterval: 1`, `advisor.continueSeverity: concern`. Use `reviewMode: turn` when the advisor is a second resident model. Use `agent-end` when `advisor` is `"@default"`.
   - On a 16k or 32k window, `omp config set defaultThinkingLevel low`.
   - Quote `advisor.syncBacklog: "off"` if you write it.
   - Verify: `omp config get advisor.enabled` is true, and a live session shows an advisor card on a trivial edit.

4. LSP and DAP.
   - Confirm `omp config get lsp.enabled` and `omp config get debug.enabled` are true. Do not treat those as features you must switch on.
   - Confirm `omp config get edit.mode` is `hashline`. Do not set `edit.format`. The key is `edit.mode`.
   - Install language-server binaries for the user's languages. TypeScript 6 uses `typescript-language-server`, and `.omp/lsp.json` disables `typescript-native`. Python uses one of `ty`, `pyright-langserver`, `basedpyright-langserver`, or `pylsp`, installed in the project virtualenv, and disables the other three. Add `ruff` in that venv. Rust uses `rust-analyzer`. Go uses `gopls`.
   - Set `lsp.diagnosticsOnEdit` to true and `lsp.requestTimeout` to 10. Leave `lsp.lazy`, `lsp.shared`, `lsp.diagnosticsDeduplicate`, and `lsp.formatOnWrite` at their defaults unless that server's formatter is the one the repo uses.
   - Install the built-in config and docs servers. `npm install -g yaml-language-server bash-language-server dockerfile-language-server-nodejs` and `brew install terraform-ls marksman`. Confirm `command -v` finds `yaml-language-server`, `bash-language-server`, `terraform-ls`, `docker-langserver`, and `marksman`. Do not add `lsp.json` entries that restate `yamlls`, `bashls`, `terraformls`, `dockerls`, or `marksman`. Add `yamlls` settings only to attach a schema, and keep the built-in `validate`, `hover`, `completion`, and telemetry keys in that object.
   - For C or C++, generate `compile_commands.json` and link it at the repo root. Configure that build as Debug.
   - Install Biome or `vscode-langservers-extracted` only when the repo has that linter's config.
   - Install debug adapters the same way. Python: `uv add --dev debugpy`, point `debugpy` `command` at `.venv/bin/python`, and check `.venv/bin/python -m debugpy --version`. Go: `dlv` from `github.com/go-delve/delve/cmd/dlv@latest`. Native: `gdb` or `lldb-dap`. Name `lldb-dap` or `codelldb` for Rust. Node: vscode-js-debug extracted so `dapDebugServer.js` is discoverable, with `sourceMaps: true` and a concrete `outFiles` glob. Do not `npm install -g js-debug-adapter`.
   - Add `.omp/lsp.json` only to override or disable a built-in server. Shape: `{ "servers": { "<builtin-name>": { ... } } }`. A `settings` key replaces that object.
   - Add `.omp/dap.json` only when the adapter command or launch defaults differ. Shape: `{ "adapters": { "<adapter-name>": { "command": "...", "launchDefaults": {} } } }`. Do not write a VS Code launch.json.

5. Safety.
   - `omp config set tools.approvalMode write`
   - `omp config set bash.gitGuard true` when agents may share a checkout
   - `omp config set astGrep.enabled true`
   - Install the force-push hook from the guide at `~/.omp/agent/hooks/pre/force-push.ts`.

6. Native connectors.
   - `gh auth login` and `gh auth status`, then `omp config set github.enabled true`.
   - Leave `browser.enabled` at its default `true`. Do not add a Playwright or Puppeteer MCP server.
   - `omp browser-relay install`, then `omp config set browser.relay true`, and tell the user to load `~/.omp/browser-relay/extension` in Chrome.
   - Verify GitHub in a new session. Verify search with `omp web-search`.

7. MCP.
   - Run `/mcp list`. Disable imported `github`, `playwright`, `puppeteer`, and `browserbase` servers via `disabledServers` or `/mcp disable`.
   - Write `~/.omp/agent/mcp.json` with Context7 at `https://mcp.context7.com/mcp/oauth` and DeepWiki at `https://mcp.deepwiki.com/mcp`. Do not put a raw API key in the file. Verify with `/mcp test context7` after `/mcp reauth context7`, and with `/mcp test deepwiki`. Leave the private DeepWiki endpoint (`https://mcp.devin.ai/mcp`) off unless the user asks for private repositories.
   - Add Linear readonly (`https://mcp.linear.app/mcp/readonly`), project-scoped Sentry (`https://mcp.sentry.dev/mcp/ORG/PROJECT`), restricted `postgres-mcp`, or Figma (`https://mcp.figma.com/mcp`) only when the user says that repo uses them. Postgres uses `uvx`, `--access-mode=restricted`, and `DATABASE_URI` copied from the environment. For Figma, run `/mcp reauth figma` and `/mcp test figma`. If Figma refuses the client, remove the server. omp is not in Figma's MCP catalog.
   - Do not add a filesystem MCP, a memory MCP, or the GitHub Copilot MCP.

8. Subagents.
   - Use the bundled `scout`, `reviewer`, `security-reviewer`, and `librarian` before adding files.
   - Add `api-reviewer` (tools `read`, `grep`, `glob`, `lsp`, model `"@slow"`), `migration-runner` (`read`, `edit`, `bash`), and `test-writer` (`read`, `edit`, `bash`, `lsp`).
   - Frontmatter requires `name` and `description`. Isolation is requested per task, not in frontmatter.
   - Verify in `/agents`.

9. Memory and autolearn.
   - `omp config set memory.backend mnemopi`
   - Set `autolearn.enabled: true`, `autolearn.minToolCalls: 5`, `autolearn.autoContinue: false`.
   - Verify: `omp config get memory.backend`.

10. Skills and instructions.
    - Create `<repo>/.omp/skills/pre-merge/SKILL.md`, `rollback/SKILL.md`, and `verify-change/SKILL.md` from the guide. One directory level under `skills/`. The `description` states when to load the skill.
    - Create or update `<repo>/AGENTS.md` with build, test, and do-not-touch rules if the repo has them.

11. Do not "tune" defaults that are already right.
    - Leave `compaction.thresholdPercent` at `-1`, `compaction.keepRecentTokens` at `20000`, and `edit.mode` at `hashline`.
    - Leave `browser.enabled` and `web_search.enabled` on.
    - Spend is inspected with `omp stats --summary`, not with `npx` packages.

12. Always-on coverage, from §16.
    - Write `~/.omp/unattended.yml` denying `computer` and `browser`.
    - Set `stt.enabled` true and `checkpoint.enabled` true. Set `modelRoles.dictation` only to an id from `omp models --kind stt`. Leave `/live` unused.
    - Register `omp acp` in Zed, JetBrains, or a VS Code ACP client.
    - Install a launchd or systemd timer that runs `omp -p` with the unattended config.
    - Add the self-hosted GitHub Actions workflow only on this machine. Skip fork pull requests.
    - Add Slack at `https://mcp.slack.com/mcp` when the user wants Slack from an open session.
    - Install the Semgrep CLI for the security review step.

Deliverable: global `config.yml` as in the setup guide, Context7 and DeepWiki connected, GitHub enabled through `gh`, the force-push hook installed, three custom subagents, the `verify-change` skill, memory on, advisor on, the unattended config, language servers (including YAML, shell, Terraform, Docker, and Markdown) and at least one debug adapter verified with `command -v` or the module version check, and a short report of every key you changed and why.
````

---

## 23. After the first week

Run `omp stats --summary`. If the advisor pass or subagent fan-out makes the machine swap the local model on every turn, switch review to `agent-end`. `task` already points at `@default`. If Context7, DeepWiki, Sentry, Postgres, or Figma show up as noise in transcripts, set `"instructions": false` on that server before you disconnect it.

Add a subagent the second time you delegate the same kind of review. Add a project MCP server the second time you paste output from a system the agent cannot see. Raise `autolearn.autoContinue` to `true` only after captured lessons are worth the extra turn.

The advantage over Claude Code is the stack running together: language-server edits, a debugger, hashline, an advisor, worktree subagents, `gh`, Context7, and a hook the model cannot override. A longer MCP list does not add to that. A duplicate GitHub or browser server subtracts from it.
