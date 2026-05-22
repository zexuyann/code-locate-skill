---
name: code-locate
description: Use this skill when a user gives a natural-language bug, issue, test failure, stack trace, or behavior report and wants an agent to locate the most relevant code using the local code-locate CLI. The skill guides the agent to rewrite the issue into a grep-friendly search plan, run deterministic code-locate retrieval, inspect evidence, follow context/expand/refs, and make root-cause judgments only after reading source code.
---

# code-locate

Use this skill to locate likely code entry points for an issue. `code-locate` is deterministic retrieval: it does not call an LLM, build embeddings, maintain a persistent index, rewrite issues, or decide root cause. The agent performs query rewriting, code reading, and judgment.

## Contract

- Generate a structured search plan before calling the CLI.
- Prefer `code-locate query --spec ... --top 5 --json`.
- Use `context`, `expand`, and `refs` to inspect promising results.
- Treat scores and confidence as retrieval signals, not proof of the bug.
- Execute `suggested_next_steps[].argv` when available; `display` is for human logs.
- Do not ask the CLI to reason, diagnose, or call an LLM.
- Do not use raw issue text as the only query unless no better terms exist.

## Workflow

1. Classify the issue: stack trace, test failure, frontend/state, API/backend, Python/library algorithm, docs/config, or unknown.
2. Extract high-signal code terms: APIs, functions, classes, modules, files, test names, assertion text, error messages, UI labels, routes, config keys, and domain terms.
3. Write a narrow JSON search plan with 5-15 useful terms. Use `include_globs` when the subsystem is clear.
4. Save the plan to a temporary file.
5. Run:

```bash
code-locate query --spec /tmp/code-locate-search-plan.json --top 5 --json
```

If the current working directory is not the target repository root, pass `--repo /path/to/repo`.

6. Read `results[].evidence`, `matched_terms`, `symbol`, `confidence`, and `suggested_next_steps`.
7. Inspect the top 1-3 implementation-looking results before doing more searches:

```bash
code-locate context path/to/file.py:123 --radius 80 --symbol
code-locate expand path/to/file.py:123 --depth 2 --top 20 --json
```

8. Follow key symbols and caller-like references:

```bash
code-locate refs symbol_name --top 20 --json
```

9. Decide the likely root cause only after reading actual code.

## Search Plan Schema

Use these fields. Omit fields that are not useful.

```json
{
  "issue": "short issue summary",
  "exact_phrases": ["literal UI text, error text, API names, config keys"],
  "identifiers": ["functionName", "ClassName", "module_name", "test_name"],
  "concept_terms": ["domain", "behavior", "algorithm"],
  "api_terms": ["/route", "rpcMethod", "queryKey"],
  "storage_terms": ["localStorage", "sessionStorage", "table_name"],
  "framework_terms": ["useEffect", "onMounted", "loader", "action"],
  "include_globs": ["src/**/*.ts"],
  "exclude_globs": ["coverage/**"]
}
```

All fields except `issue` are lists of strings. `exclude_globs` is appended to the CLI's built-in excludes, not a replacement.

## Plan Rules

- Start narrow. Broaden only if results are empty or irrelevant.
- Preserve exact names from the issue: public APIs, private helpers, class names, test names, module paths, error strings, option names, and route fragments.
- Convert user phrases into likely identifiers: `save settings` -> `saveSettings`, `saveConfig`, `persistSettings`, `loadSettings`.
- Use `include_globs` for clear subsystems, such as `astropy/modeling/**/*.py`, `src/**/*.ts`, or `packages/*/src/**/*.tsx`.
- Exclude generated, vendored, build, coverage, docs, and changelog files unless the issue is about those files.
- Prefer exact identifiers and phrases over generic concept terms.
- Avoid very short or broad terms alone: `id`, `x`, `to`, `on`, `data`, `value`, `state`, `handle`, `model`, `result`, `error`.
- Avoid substring traps. For example, do not search `sqf` alone if it can match `LSQFitter`; use `sqf_list`, `def sqf`, or package-specific globs.
- Do not add hidden-file globs unless hidden files are clearly relevant; hidden globs can widen search scope.

## Issue-Specific Hints

Stack trace:

- Put exception class/message in `exact_phrases`.
- Put failing function/class names in `identifiers`.
- Use stack frame paths as `include_globs`.

Test failure:

- Include test name, assertion helper, expected/actual literals, fixture names, and production identifiers referenced by the test.
- If top results are only tests, inspect the test briefly, extract production terms, and rerun `query`.

Python/library algorithm bug:

- Include public API names, helper names, classes, and domain-specific terms.
- Prefer subsystem globs when obvious.
- Keep generic math words out unless paired with exact identifiers.

Frontend/state bug:

- Include UI labels and user actions.
- Add handler and persistence terms such as `save*`, `load*`, `persist*`, `restore*`.
- Add storage/API/framework terms such as `localStorage`, endpoint paths, `useEffect`, `loader`, `hydrate`.

API/backend bug:

- Include route fragments, controller/service/repository names, schema names, query keys, table names, and exact error messages.

Docs/config issue:

- Include config keys, flags, environment variables, option names, and exact doc phrases.
- Include docs globs when the issue is documentation-related.

Unknown:

- First query exact names only.
- Second query adds broader concept terms and looser globs.

## Reading JSON Results

For `query`, prioritize:

- `results[].path` and `lines`
- `results[].symbol`
- `results[].matched_terms`
- `results[].evidence`
- `results[].suggested_next_steps`

`suggested_next_steps` entries are structured:

```json
{
  "argv": ["code-locate", "context", "src/settings.ts:10", "--radius", "80"],
  "display": "code-locate context src/settings.ts:10 --radius 80"
}
```

Use `argv` for execution. Do not split or execute `display`.

For `expand`, inspect:

- `imports`
- `dependencies`
- `dependents`
- `outgoing_calls`
- `local_callees`
- `imported_callees`
- `incoming_references`
- `related_files`
- `graph`
- `analysis`

`expand` and `refs` are syntax-static/grep-based signals, not precise language-server analysis.

When `--json` is present and the CLI fails, it writes a JSON error object to stderr and exits non-zero:

```json
{"error": {"type": "ValueError", "message": "query requires either a raw query or --spec"}}
```

Treat this as a failed retrieval step, adjust inputs, and retry only when the correction is clear.

## Safety And Limits

- The CLI is meant for local source repositories, not arbitrary filesystem roots.
- `context` and `expand` reject targets that resolve outside `--repo`.
- Hidden files and directories are not searched by default.
- Built-in excludes cover common generated, vendored, virtualenv, cache, coverage, `.env*`, and private-key paths.
- Files larger than the CLI read limit are skipped or rejected depending on command.
- `--top` max is 100 for `query`, `refs`, and `expand`.
- `context --radius` max is 500.
- `query --max-matches-per-term` max is 2000.
- `query --parse-limit` max is 500.
- `expand --depth` max is 3.

If results are irrelevant, tighten terms or `include_globs` before raising limits.

## Interpreting Results

- If top results point to implementation code, inspect `context` before running more searches.
- If top results point only to tests, use test names/assertions to create a better plan, then rerun `query`.
- If top results are docs/changelogs and the issue is not documentation, tighten `include_globs` and remove generic terms.
- If results are empty, add synonyms, API paths, likely identifiers, and subsystem globs.
- If a result identifies a central helper, run `expand` and `refs`, then inspect callers/tests.
- Stop once enough source context has been read to support a concrete next engineering action.

For a concrete search plan example, read `examples/search-plan.settings-persistence.json` only when needed.
