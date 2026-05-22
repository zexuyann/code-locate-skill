# code-locate Skill

![Skill](https://img.shields.io/badge/AI%20Agent-skill-blue.svg)
![CLI](https://img.shields.io/badge/requires-code--locate--cli-orange.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)

An AI agent skill that teaches agents how to locate likely code entry points from bug reports, test failures, stack traces, and behavior issues using the local `code-locate` CLI.

[English](#features) | [中文](#功能特性)

## Features

- **Issue rewriting** - guides agents to turn natural-language issues into grep-friendly search plans
- **CLI-driven retrieval** - depends on `code-locate-cli` for deterministic code search
- **Evidence-first workflow** - teaches agents to inspect matched lines, symbols, and source context
- **Follow-up navigation** - uses `context`, `expand`, and `refs` to move from candidates to relevant code
- **Structured output aware** - tells agents to execute `suggested_next_steps[].argv`, not parse shell strings
- **Safety-aware search** - encourages narrow `include_globs`, careful term selection, and bounded exploration
- **No root-cause shortcuts** - keeps diagnosis in source-code reading, not score interpretation

> **AI Agent Tip:** Generate a structured search plan first, run `code-locate query --spec ... --json`, then inspect source before deciding root cause.

## Dependency

This skill depends on the `code-locate-cli` tool being installed and available on `PATH` as `code-locate`.

- CLI repository: [code-locate-cli](https://github.com/zexuyann/code-locate-cli)
- Skill repository: [code-locate-skill](https://github.com/zexuyann/code-locate-skill)

Install and verify the CLI first:

```bash
uv tool install git+https://github.com/zexuyann/code-locate-cli.git
code-locate --version
```

The CLI provides deterministic retrieval. It does not call an LLM, build embeddings, create a persistent index, rewrite natural-language issues, or decide the final root cause.

## Installation

Install this skill into the skill directory used by your agent tool:

```bash
mkdir -p /path/to/agent-skills/code-locate
cp -R /path/to/code-locate-skill/. /path/to/agent-skills/code-locate/
```

Replace `/path/to/agent-skills` with the skill directory configured by your agent tool.

Restart or refresh the agent tool after installation so the skill metadata is loaded.

You can also ask your agent to install it for you. For example:

```text
Install the code-locate skill from https://github.com/zexuyann/code-locate-skill into my local agent skills directory. It depends on code-locate-cli from https://github.com/zexuyann/code-locate-cli.
```

## Usage

Once installed, the skill should trigger when the user asks the agent to locate code for:

- bug reports
- test failures
- stack traces
- frontend or backend behavior issues
- docs/config issues where relevant code or config must be found

For the most reliable trigger, mention `code-locate` by name:

```text
Use code-locate to find the code related to this bug: settings disappear after refresh.
```

Other examples:

```text
Use code-locate to locate the implementation related to this pytest failure:
<paste test failure>
```

```text
Use code-locate to find the likely code entry points for this stack trace:
<paste stack trace>
```

```text
Use code-locate to investigate this behavior issue:
Clicking Save works, but settings are lost after reloading the page.
```

Typical agent workflow:

```bash
# 1. Write a search plan to /tmp/code-locate-search-plan.json
code-locate query --spec /tmp/code-locate-search-plan.json --repo /path/to/repo --top 5 --json

# 2. Inspect promising source locations
code-locate context src/settings/saveConfig.ts:41 --repo /path/to/repo --radius 80 --symbol

# 3. Expand dependency and caller-like signals
code-locate expand src/settings/saveConfig.ts:41 --repo /path/to/repo --depth 2 --top 20 --json

# 4. Follow key symbols
code-locate refs saveConfig --repo /path/to/repo --top 20 --json
```

## Search Plan Example

```json
{
  "issue": "settings disappear after refresh",
  "identifiers": ["saveConfig", "loadSettings", "persistSettings"],
  "concept_terms": ["settings", "persist", "refresh"],
  "storage_terms": ["localStorage"],
  "framework_terms": ["useEffect"],
  "include_globs": ["src/**/*.ts", "src/**/*.tsx"]
}
```

The skill tells agents to start narrow, prefer exact identifiers and phrases, and broaden only when results are empty or irrelevant.

## How It Works

`SKILL.md` contains the actual agent instructions. It tells the agent to:

- classify the issue type
- extract high-signal search terms
- create a JSON search plan
- run `code-locate query --spec ... --json`
- read evidence before following more results
- use `context`, `expand`, and `refs` for iterative investigation
- treat scores as ranking hints, not proof
- decide root cause only after reading actual code

## Structured Output Contract

The companion CLI emits structured next steps:

```json
{
  "argv": ["code-locate", "context", "src/settings.ts:10", "--radius", "80"],
  "display": "code-locate context src/settings.ts:10 --radius 80"
}
```

Agents should execute `argv`. `display` is only for logs and copy/paste.

When `--json` is present and the CLI fails, it writes a JSON error object to stderr:

```json
{"error": {"type": "ValueError", "message": "query requires either a raw query or --spec"}}
```

## Files

```text
code-locate-skill/
├── README.md
├── SKILL.md
└── examples/
    └── search-plan.settings-persistence.json
```

## Development

There is no build step for this skill. Edit `SKILL.md`, then refresh your agent tool so the skill metadata is reloaded.

When the companion CLI JSON contract changes, update both:

- `SKILL.md`
- this `README.md`

## Troubleshooting

**Q: The skill triggers, but `code-locate` is not found**

Install the companion CLI first:

```bash
uv tool install git+https://github.com/zexuyann/code-locate-cli.git
```

**Q: The agent gets irrelevant results**

Use narrower identifiers, exact phrases, and `include_globs`. Avoid generic terms such as `data`, `state`, `error`, `model`, or `result` unless paired with high-signal terms.

**Q: The top results are only tests**

Inspect the test briefly, extract production identifiers or stack-frame names, then rerun `query` with a better plan.

**Q: The CLI returns structured next steps**

Execute `argv`; do not split or execute `display`.

## License

MIT

---

## 功能特性

- **Issue 改写** - 指导 agent 将自然语言问题改写成 grep-friendly search plan
- **依赖 CLI 检索** - 使用本地 `code-locate-cli` 做确定性代码检索
- **证据优先** - 要求 agent 阅读 matched lines、symbols 和源码上下文
- **迭代导航** - 使用 `context`、`expand`、`refs` 继续追踪相关代码
- **理解结构化输出** - 使用 `suggested_next_steps[].argv` 执行下一步
- **安全搜索边界** - 鼓励使用精确 terms 和 `include_globs` 控制搜索范围
- **不跳过根因判断** - 分数只作为排序信号，根因必须基于源码阅读

## 依赖

此 skill 依赖 companion CLI：

- CLI 仓库：[code-locate-cli](https://github.com/zexuyann/code-locate-cli)
- Skill 仓库：[code-locate-skill](https://github.com/zexuyann/code-locate-skill)

先安装 CLI：

```bash
uv tool install git+https://github.com/zexuyann/code-locate-cli.git
code-locate --version
```

## 安装

```bash
mkdir -p /path/to/agent-skills/code-locate
cp -R /path/to/code-locate-skill/. /path/to/agent-skills/code-locate/
```

将 `/path/to/agent-skills` 替换为你的工具配置的 skill 目录。安装后重启或刷新对应 agent 工具，让 skill metadata 生效。

也可以直接让 agent 帮你安装，例如：

```text
请把 https://github.com/zexuyann/code-locate-skill 安装到我的本地 agent skills 目录。它依赖 https://github.com/zexuyann/code-locate-cli，请先确保 code-locate CLI 可用。
```

## 使用流程

最稳的触发方式是在请求里直接提到 `code-locate`：

```text
用 code-locate 帮我定位这个 bug 相关代码：点击保存后刷新页面配置丢失
```

也可以这样：

```text
这个测试失败了，帮我用 code-locate 找相关实现代码：
<粘贴测试失败信息>
```

```text
根据这个 stack trace，用 code-locate 定位相关代码：
<粘贴 stack trace>
```

skill 会指导 agent：

1. 分析 bug report、test failure、stack trace 或行为问题
2. 提取高信号 identifiers、exact phrases、API/storage/framework terms
3. 写入 JSON search plan
4. 调用 `code-locate query --spec ... --json`
5. 阅读 evidence、context、expand 和 refs
6. 在阅读真实源码后再判断根因

## 许可证

MIT
