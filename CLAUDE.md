# claude-code-from-scratch

@./.claude/rules/chinese-greeting.md

A minimal coding agent inspired by Claude Code, built from scratch in ~3400 lines (TypeScript) and ~2920 lines (Python). Implements the core agent loop, tool system, memory, skills, sub-agents, MCP integration, plan mode, and multi-tier context compression.

This is both a learning project (with full docs in `docs/`) and a functional coding agent CLI.

---

## Repository Layout

```
claude-code-from-scratch/
├── src/                  # TypeScript implementation (primary)
│   ├── agent.ts          # Agent class: main loop, streaming, compression, plan mode (~1531 lines)
│   ├── cli.ts            # CLI entry point, REPL, argument parsing
│   ├── tools.ts          # 13 tool definitions, execution, permission system
│   ├── prompt.ts         # System prompt builder: templates, @include, CLAUDE.md loader
│   ├── memory.ts         # 4-type file-based memory with semantic recall
│   ├── skills.ts         # Skill discovery, frontmatter parsing, inline/fork execution
│   ├── subagent.ts       # Sub-agent config: built-in (explore/plan/general) + custom agents
│   ├── mcp.ts            # MCP client: JSON-RPC over stdio, tool discovery and routing
│   ├── session.ts        # Session persistence (JSON) and --resume support
│   ├── ui.ts             # Terminal output: colors, spinner, dividers
│   └── frontmatter.ts    # YAML frontmatter parser (used by skills/memory/agents)
├── python/               # Python implementation (parallel feature parity)
│   └── mini_claude/      # Package: agent.py, tools.py, memory.py, skills.py, etc.
├── docs/                 # Chapter-by-chapter tutorial (Chinese)
├── en/docs/              # English translation of docs
├── .claude/
│   ├── rules/            # Auto-loaded instruction files (*.md injected into system prompt)
│   │   └── chinese-greeting.md
│   ├── skills/           # Project-level skills (SKILL.md per subdirectory)
│   │   ├── commit/SKILL.md
│   │   ├── greet/SKILL.md
│   │   ├── karpathy-guidelines/SKILL.md
│   │   └── harness/
│   │       ├── SKILL.md
│   │       └── references/ # 6 pattern/template reference docs
│   └── agents/           # Custom sub-agent definitions (*.md with frontmatter)
│       └── reviewer.md
├── test/
│   ├── TEST-GUIDE.md     # Manual test guide (19 features, Chinese)
│   ├── setup.sh          # One-shot test environment setup
│   ├── cleanup.sh        # Tear down test artifacts
│   ├── mcp-server.cjs    # Test MCP server (add/echo/timestamp tools)
│   ├── large-file.txt    # 75KB file for large-result persistence test
│   └── quote-test.js     # File for curly-quote normalization test
├── .mcp.json             # MCP server config (test server wired by default)
├── package.json          # TS build scripts: build, start, dev
├── tsconfig.json         # Target ES2022, ESNext modules, outDir dist/
└── index.html            # Docsify documentation site entry
```

---

## Development Workflows

### TypeScript (primary)

```bash
npm install
npm run build          # tsc → dist/
node dist/cli.js       # interactive REPL
node dist/cli.js --yolo "read package.json"   # one-shot

# During development
npm run dev            # tsc + run in one step
```

### Python

```bash
cd python
pip install -e .
mini-claude-py         # interactive REPL
mini-claude-py --yolo "list files"
```

### Environment

API key via env var only (never CLI):
```bash
export ANTHROPIC_API_KEY=sk-ant-xxx          # Anthropic backend
# OR
export OPENAI_API_KEY=sk-xxx
export OPENAI_BASE_URL=https://...           # OpenAI-compatible backend
```

When both are present, `OPENAI_API_KEY + OPENAI_BASE_URL` takes precedence.

### Testing

There is no automated test suite. Testing is manual per `test/TEST-GUIDE.md` (19 feature tests). Set up the test environment first:

```bash
bash test/setup.sh     # configures MCP, skills, CLAUDE.md, large-file
node dist/cli.js --yolo
# ... run tests per TEST-GUIDE.md ...
bash test/cleanup.sh   # tear down
```

---

## CLI Options

```
mini-claude [options] [prompt]

--yolo, -y         bypassPermissions — skip all confirmation prompts
--plan             plan mode — read-only, write only to plan file
--accept-edits     auto-approve file writes/edits, still confirm shell
--dont-ask         auto-deny anything requiring confirmation (CI use)
--thinking         enable extended thinking (Anthropic models only)
--model, -m        model name (default: claude-opus-4-6, or $MINI_CLAUDE_MODEL)
--api-base URL     OpenAI-compatible endpoint
--resume           restore last saved session
--max-cost USD     stop when estimated spend exceeds limit
--max-turns N      stop after N agentic turns
```

### REPL Commands

| Command | Effect |
|---------|--------|
| `/clear` | Clear conversation history |
| `/plan` | Toggle plan mode on/off |
| `/cost` | Show token usage and estimated cost |
| `/compact` | Manually compact conversation |
| `/memory` | List saved memories |
| `/skills` | List available skills |
| `/<skill-name> [args]` | Invoke a skill |

---

## Architecture: Agent Loop

`Agent.chat(userMessage)` drives the loop (`src/agent.ts`):

1. **Push user message** to conversation history
2. **Auto-compact** if context approaching limit (85% of effective window)
3. **Start memory prefetch** — async semantic recall, non-blocking
4. **Run compression pipeline** (tiers 1–3, zero-API-cost) before each API call
5. **Stream API call** (Anthropic or OpenAI format)
   - For Anthropic: fires `onToolBlockComplete` callback as each `tool_use` block finishes, allowing read-only tools to start executing while the model is still generating
6. **Execute tool calls** with permission checks
7. **Push tool results** and loop back to step 4 unless no tool calls remain

The loop exits when the model returns a response with no tool calls.

---

## Tools (src/tools.ts)

| Tool | Category | Notes |
|------|----------|-------|
| `read_file` | Read | Returns content with line numbers; tracks mtime for write-before-edit check |
| `write_file` | Write | Requires prior `read_file` for existing files; auto-creates parent dirs |
| `edit_file` | Write | Requires prior `read_file`; unique `old_string` match; curly-quote normalization |
| `list_files` | Read | Glob pattern search; ignores `node_modules`, `.git` |
| `grep_search` | Read | Regex search; uses system `grep` on Linux/macOS, JS fallback on Windows |
| `run_shell` | Shell | Dangerous patterns require confirmation; 30s default timeout |
| `web_fetch` | Read | Strips HTML tags; 50KB default limit |
| `skill` | Meta | Invokes a skill (inline mode injects prompt; fork mode spawns sub-agent) |
| `agent` | Meta | Spawns sub-agent with isolated context; types: `explore`/`plan`/`general` |
| `tool_search` | Meta | Activates deferred tools by name/keyword |
| `enter_plan_mode` | Mode | Switch to read-only plan mode (deferred — load via `tool_search`) |
| `exit_plan_mode` | Mode | Exit plan mode, trigger approval flow (deferred) |

**Concurrency-safe tools** (can run in parallel during streaming): `read_file`, `list_files`, `grep_search`, `web_fetch`.

**Large result handling**: Results >30 KB are persisted to `~/.mini-claude/tool-results/` and replaced with a path + 200-line preview in context.

**Read-before-edit**: Writing or editing an existing file without a prior `read_file` in this session returns an error.

---

## Permission System (src/tools.ts)

Five permission modes (`PermissionMode`):

| Mode | Effect |
|------|--------|
| `default` | Reads always allowed; writes to existing files allowed; new files and dangerous shell need confirm |
| `acceptEdits` | Auto-approve file writes/edits; still confirm dangerous shell |
| `bypassPermissions` | Allow everything (--yolo); deny rules still apply |
| `dontAsk` | Auto-deny anything requiring confirmation (CI) |
| `plan` | Read-only; only the plan file is writable; shell blocked entirely |

**Custom rules** via `.claude/settings.json` or `~/.claude/settings.json`:
```json
{
  "permissions": {
    "allow": ["run_shell(npm test)", "run_shell(git log*)"],
    "deny": ["run_shell(rm *)"]
  }
}
```
Deny rules always win, even in `bypassPermissions` mode.

---

## Context Compression

Multi-tier pipeline runs before each API call (tiers 1–3 are zero API cost):

| Tier | Trigger | Action |
|------|---------|--------|
| 1 Budget | >50% context utilization | Truncate large tool results (budget: 30KB→15KB as fill increases) |
| 2 Snip | >60% context utilization | Replace old/duplicate read results with `[Content snipped]` |
| 3 Microcompact | Idle >5 min (cache cold) | Clear all but 3 most recent tool results |
| 4 Autocompact | >85% context utilization | API call to summarize conversation into 2-message summary |

---

## Memory System (src/memory.ts)

Persistent file-based memory stored at `~/.mini-claude/projects/<cwd-hash>/memory/`.

**Memory types**: `user`, `feedback`, `project`, `reference`

**File format** (YAML frontmatter + markdown body):
```markdown
---
name: memory name
description: one-line description
type: user
---
Memory content here.
```

**Semantic recall**: On each user turn, a non-blocking `sideQuery` call selects relevant memories from the index using the same model. Up to 5 memories are injected as `<system-reminder>` into the next user message. Session budget: 60 KB cumulative.

`MEMORY.md` index is auto-updated when any memory file is written.

---

## Skills System (src/skills.ts)

Skills are prompt templates discovered from:
1. `~/.claude/skills/<name>/SKILL.md` (user-level, lower priority)
2. `.claude/skills/<name>/SKILL.md` (project-level, higher priority)

**SKILL.md frontmatter**:
```yaml
---
name: skill-name
description: what it does
user-invocable: true   # false = AI-only
context: inline        # inline (inject prompt) or fork (run in sub-agent)
allowed-tools: read_file,grep_search   # optional, fork mode only
---
```

**`$ARGUMENTS`** in the prompt template is replaced with args passed by the user.

**Invocation**: User types `/<name> [args]`, or AI uses the `skill` tool.

### Project Skills

| Skill | Invocation | Description |
|-------|-----------|-------------|
| `commit` | `/commit [context]` | Stage 확인 후 git commit 메시지 작성 및 커밋 |
| `greet` | `/greet [name]` | 짧고 창의적인 인사말 생성 |
| `karpathy-guidelines` | `/karpathy-guidelines` | LLM 코딩 실수를 줄이기 위한 행동 지침 (Andrej Karpathy 기반): 코딩 전 가정 명시, 단순성 우선, 최소 변경, 검증 가능한 목표 설정 |
| `harness` | `/harness` | 에이전트 팀 & 스킬 아키텍트 메타 스킬. 도메인 설명을 에이전트 팀과 스킬 세트로 자동 변환 (6가지 아키텍처 패턴). `references/`에 오케스트레이터 템플릿, 스킬 작성 가이드, 테스트 가이드, QA 가이드, 팀 예시 포함 |

---

## Sub-Agent System (src/subagent.ts)

The `agent` tool spawns an isolated sub-agent (separate `Agent` instance, separate message history) and returns its text output.

**Built-in types**:
- `explore` — read-only (`read_file`, `list_files`, `grep_search`); fast search specialist
- `plan` — read-only; structured implementation plan output
- `general` — full tools except `agent` (no recursive nesting)

**Custom agents** in `.claude/agents/<name>.md`:
```yaml
---
name: reviewer
description: Code review specialist
allowed-tools: read_file,list_files,grep_search
---
System prompt body here...
```

Sub-agent token costs aggregate to the parent's totals.

---

## MCP Integration (src/mcp.ts)

Reads server config from (merged in order, project overrides user):
1. `~/.claude/settings.json` — `mcpServers` key
2. `.claude/settings.json` — `mcpServers` key
3. `.mcp.json` — top-level object treated as `mcpServers`

**Config format**:
```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["path/to/server.js"],
      "env": { "KEY": "value" }
    }
  }
}
```

MCP tools are exposed with `mcp__<serverName>__<toolName>` prefix. Connection happens lazily on first `chat()` call.

---

## System Prompt (src/prompt.ts)

Built dynamically each session by `buildSystemPrompt()`:
- Static instruction template (embedded in `prompt.ts`)
- Working directory, date, platform, shell, git branch/status/log
- `CLAUDE.md` contents (walked from cwd up to filesystem root; `@include` directives resolved recursively up to depth 5)
- `.claude/rules/*.md` files (sorted, auto-loaded)
- Memory index and system prompt section from `memory.ts`
- Skill descriptions from `skills.ts`
- Custom agent descriptions from `subagent.ts`
- Deferred tool names (loaded on demand via `tool_search`)

---

## Session Persistence (src/session.ts)

Sessions auto-save to `~/.mini-claude/sessions/<uuid>.json` after each `chat()` call. Resume with `--resume` (loads the most recent session by start time).

---

## Key Conventions for AI Assistants

- **Always read before editing**: The tool layer enforces this (`readFileState` mtime check). Do not attempt to write or edit an existing file without first reading it.
- **Prefer dedicated tools over `run_shell`**: Use `read_file` not `cat`, `list_files` not `find`, `grep_search` not `grep`, `write_file`/`edit_file` not `sed`.
- **Parallel reads are safe**: `read_file`, `list_files`, `grep_search`, and `web_fetch` can all be called in parallel.
- **No automated tests**: Validate changes manually using `test/TEST-GUIDE.md`. Running `npm run build` (TypeScript) or Python import checks are the only automated checks.
- **Both implementations must stay in sync**: TypeScript in `src/` is primary; Python in `python/mini_claude/` mirrors all features. A change to one should be mirrored in the other.
- **Do not edit `MEMORY.md` directly**: It is auto-rebuilt from memory files.
- **Deferred tools**: `enter_plan_mode` and `exit_plan_mode` are not included in the active tool list by default. Use `tool_search` to activate them.
- **Cost tracking**: Cost is estimated at $3/M input + $15/M output tokens. The model can check via `/cost`.

---

## Harness 포인터

- **트리거**: 에이전트 팀 구축, 멀티 에이전트 설계, 스킬 아키텍처, 하네스 구성/확장/재설계/업데이트 요청 시 `/harness` 스킬 활성화
- **변경 이력**:
  - 2026-07-03: harness v1.2.0 추가 (revfactory/harness) — 에이전트 팀 & 스킬 아키텍트 메타 스킬, 6가지 아키텍처 패턴, 참조 문서 6종
