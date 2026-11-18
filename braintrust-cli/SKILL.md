---
name: braintrust-cli
description: Braintrust CLI (bt) for Braintrust app URLs and API work, including logs, traces, spans, SQL over project logs, experiments, datasets, prompts, scorers, functions, evals, Topics and environments. Use bt instead of browser automation for braintrust.dev links, and whenever a task mentions Braintrust, a trace or root span id, an experiment comparison, or `bt` commands.
metadata:
  version: "1.0.0"
  bt-version: "0.32.0"
  tags: braintrust,cli,bt,llm-observability,evals
  alwaysApply: "false"
---

# bt (Braintrust CLI)

`bt` reads and manages Braintrust projects from the terminal: logs and traces, SQL, experiments, datasets, prompts, scorers and other functions, eval runs, Topics, and coding-agent tracing.

## Routing

Use this skill for Braintrust app URLs (`https://www.braintrust.dev/app/<org>/p/<project>/...`) and read-only Braintrust investigations. Most `view` commands take the URL directly, so pass it instead of parsing ids by hand. Prefer `bt` over browser automation unless the user asks for visual inspection.

The `mcp__braintrust__*` tools cover things `bt` does not: creating alerts, Loop automations, online-scoring rules, facets, dashboards, human-review queues, permalinks and Braintrust docs search. Use those (and their `load_braintrust_skill` guides) for that work, and `bt` for everything else.

## Quick Reference

| Task | Command |
|------|---------|
| Who am I / active org and project | `bt status` |
| List projects | `bt projects list --json` |
| Recent logs (last 1h) | `bt view logs -p <project> --limit 25 --non-interactive` |
| Open a Braintrust URL | `bt view trace '<url>' --non-interactive` |
| All spans in a trace | `bt view trace -p <project> --trace-id <root-span-id> --non-interactive` |
| LLM conversation for a trace | `bt view thread -p <project> --trace-id <root-span-id> --non-interactive` |
| Timing, tokens, cost, cache per span | `bt view waterfall -p <project> --trace-id <root-span-id> --non-interactive` |
| One span, full content | `bt view span -p <project> --id <span-row-id> --non-interactive` |
| Ad hoc query | `bt sql -p <project> --non-interactive "SELECT ... FROM logs WHERE created > now() - interval 1 day LIMIT 20"` |
| Compare experiments | `bt experiments compare <base> <challenger> -p <project>` |
| View a dataset | `bt datasets view <dataset> -p <project> --limit 20` |
| Prompt in an environment | `bt prompts view <slug> -p <project> --environment production` |
| Run an eval locally, no upload | `bt eval --no-send-logs my.eval.ts` |
| Topics automation status | `bt topics status -p <project> --full` |

## Required Input Resolution

1. Check context first: conversation, prior output, the URL the user gave.
2. No default org or project is assumed. Run `bt status`; if `Project: (unset)`, pass `-p <project>` on every command. Do not run `bt switch` to change the saved default unless the user asks, because it writes persistent config.
3. Discover names before using them: `bt projects list`, `bt experiments list -p <project>`, `bt datasets list -p <project>`, `bt prompts list -p <project>`, `bt functions list -p <project> -t <type>`.
4. If still ambiguous, ask. Never run a command with an unresolved placeholder like `<project>`.

## Agent Usage Rules

- Pass `--non-interactive` (on `view` and `sql`) or the global `--no-input`. Without them `bt` may open a TUI or picker and hang.
- Add `--json` when parsing output. `view logs|trace` return an envelope with `items`, `meta` and `paging.next_cursor`; `view span` returns `item` (singular); `sql --json` returns `{data, schema, cursor}`.
- Quote experiment and dataset names; they often contain spaces.
- Warnings (for example secret-rotation notices) go to stderr. Use `2>/dev/null` before piping JSON to `jq`.
- List rows are truncated to `--preview-length` (default 125). Fetch the single span, or raise `--preview-length`, before quoting content.
- Write large output to a file in the scratchpad rather than dumping it into context.
- Page with `--cursor <next_cursor>` when `paging.has_more` is true.

## Auth

```bash
bt status                 # identity, org, project, profile
bt status --all           # every saved profile and whether it connects
bt login --oauth          # browser OAuth (opens a browser; ask the user to run it)
bt login --oauth --refresh   # force-refresh an expired OAuth token
bt profiles list
bt --profile <name> ...   # use a specific saved profile
```

`BRAINTRUST_API_KEY` in the environment takes precedence over a saved profile. Pass `--prefer-profile` to use the profile anyway. Interactive login needs a browser, so ask the user to run `! bt login --oauth` rather than running it yourself.

## Command Reference

### Logs, traces and spans (`bt view`)

```bash
# Summary rows (one per trace) or spans mode; default window is 1h
bt view logs -p <project> --window 1d --limit 50 --non-interactive
bt view logs -p <project> --list-mode spans --search "timeout" --non-interactive
bt view logs -p <project> --filter "metadata.user_id = 'u_123'" --since 2026-11-01T00:00:00Z --non-interactive
bt view logs --object-ref experiment:<experiment-id> --non-interactive
bt view logs '<braintrust-logs-url>' --non-interactive

bt view trace -p <project> --trace-id <root-span-id> --limit 200 --non-interactive
bt view thread -p <project> --trace-id <root-span-id> --non-interactive
bt view waterfall -p <project> --trace-id <root-span-id> --non-interactive   # alias: timeline
bt view span -p <project> --id <span-row-id> --non-interactive
bt view span '<braintrust-span-url>' --non-interactive
```

`--trace-id` is the **root span id** (`root_span_id`), not a span's row `id`. `view waterfall` prints summary totals plus outliers (slowest, highest cost, highest tokens, lowest cache hit, errors) first, which is usually the fastest way to triage a slow or expensive trace. `--print-queries` shows the underlying query for reuse in `bt sql`.

### SQL (`bt sql`)

```bash
bt sql -p <project> --non-interactive "SELECT count(*) FROM logs WHERE created > now() - interval 1 day"
bt sql --non-interactive --json "SELECT root_span_id, span_attributes.name AS name, created
  FROM project_logs('<project>') WHERE is_root AND created > now() - interval 1 day LIMIT 20"
bt sql --non-interactive "SELECT * FROM experiment('<experiment-id>') LIMIT 5"
cat query.sql | bt sql -p <project> --non-interactive

# Long-running queries
bt sql -p <project> --async --wait-timeout 10m --non-interactive "..."
```

`FROM logs` is shorthand for `project_logs(<project>)` when `-p` is set. Always bound `created` and add `LIMIT`; unbounded scans over project logs are slow. Useful columns: `id`, `root_span_id`, `span_id`, `span_parents`, `is_root`, `span_attributes.name`, `span_attributes.type`, `input`, `output`, `expected`, `error`, `metadata.*`, `metrics.*` (tokens, cost, start/end), `scores.*`, `tags`, `created`. If the linter rejects a query you know is correct, `--force-ignore-linter` runs it anyway.

### Experiments

```bash
bt experiments list -p <project> --json
bt experiments view <name> -p <project>
bt experiments compare <base> <challenger-a> <challenger-b> -p <project>   # up to 8
bt experiments compare '<braintrust-comparison-url>'
bt experiments compare --base <base> -c <challenger> --all -p <project>     # include unchanged rows
```

### Datasets

```bash
bt datasets list -p <project>
bt datasets view <name> -p <project> --limit 20 [--full] [--verbose]
bt datasets create <name> -p <project> --file records.jsonl
bt datasets update <name> -p <project> --file records.jsonl --id-field metadata.case_id   # upsert by id
bt datasets snapshots list <name> -p <project>
bt datasets snapshots create <name> <snapshot-name> -p <project>
bt datasets snapshots restore <name> --name <snapshot-name> -p <project>   # previews, then asks
bt datasets pipeline run pipeline.ts --window 1d --limit 100               # logs -> transform -> dataset
```

Snapshot before bulk `update` or `restore` so the change can be undone.

### Prompts, functions, scorers, tools, preprocessors

```bash
bt prompts list -p <project> [--environment production]
bt prompts versions <slug> -p <project>
bt prompts view <slug> -p <project> [--version <id> | --environment <env>]
bt prompts assign <slug> --environment <env> --version <id> -p <project>

bt functions list -p <project> -t <llm|scorer|task|tool|preprocessor|facet|classifier|...>
bt functions view <slug> -p <project>          # or --id fn_...
bt functions invoke <slug> -p <project> --input '{"key":"value"}'
bt functions invoke <slug> -p <project> -m "user message" --mode json
bt functions pull <slug> -p <project> --output-dir ./braintrust --language typescript
bt functions push ./functions -p <project> --if-exists replace

bt scorers create "Helpfulness" -p <project> --model <model> --messages @messages.json \
  --choice-scores '{"A":1,"B":0}'
bt preprocessors preview ./conversation.preprocessor.ts --url '<trace-url>'
```

`tools`, `scorers` and `preprocessors` are typed views over `functions` and take the same `list/view/delete/invoke` flags. `invoke` calls a live model and may cost money; say so before running it in a loop.

### Evals

```bash
bt eval my.eval.ts                         # runs and uploads an experiment
bt eval --no-send-logs my.eval.ts          # local only
bt eval --list tests/                      # list evaluators without running
bt eval my.eval.ts --first 10              # quick, non-final run
bt eval my.eval.ts --sample 50 --sample-seed 1
bt eval my.eval.ts --param model='"<model>"'
bt eval my.eval.ts --matrix-param model='["<model-a>","<model-b>"]'
bt eval --language python my_eval.py
bt eval my.eval.ts -- --shard=1/4          # args after -- go to the eval file
```

Uploading creates a visible experiment in the shared project. Use `--no-send-logs` for scratch runs unless the user wants the result recorded.

### Topics and observability templates

```bash
bt topics status -p <project> [--full] [--progress-window 7d]
bt topics config -p <project>
bt topics report <topic-map-function-id> -p <project> --output report.json
bt topics poke -p <project>                         # queue a run
bt topics rewind 7d -p <project>                    # reprocess history
bt observability template pull -p <project> --output template.json
bt observability template push template.json -p <other-project>
```

### Environments, projects, sync

```bash
bt environments list
bt projects list
bt sync pull project_logs:<project> --window 24h --traces 100     # to ./bt-sync as JSONL
bt sync pull experiment:<name> -p <project>
bt sync status project_logs:<project-id>
```

### Coding-agent tracing

```bash
bt trace doctor claude                     # explain current tracing config
bt trace enable claude --tag <tag>         # installs a plugin; ask first
bt trace import claude <session-id> --destination project_logs:<project-id>
bt trace run claude -- <claude-args>       # trace a single invocation
```

## Mutations

These change shared state that teammates and production read. Confirm with the user before running them, and show the exact command:

- `delete` on anything (`projects`, `experiments`, `datasets`, `prompts`, `functions`, `scorers`, `tools`, `preprocessors`, `environments`, `topics config`, dataset snapshots). Never pass `--force`/`-f` unless the user approved that specific deletion.
- `prompts assign` / `unassign` (changes what an environment serves), `environments update`.
- `functions push`, `scorers create --if-exists replace`, `preprocessors push`, `custom-views push`, `observability template push`.
- `datasets update`, `datasets snapshots restore`, `datasets pipeline run|push`, `sync push`.
- `topics config enable|set|delete`, `topics rewind`.
- `trace enable|disable` (edits local agent config), `bt switch`, `bt init`, `bt login`, `bt logout`.

## Subcommand Discovery

```bash
bt --version
bt <command> --help
bt <command> <subcommand> --help
bt docs fetch --workflow evaluate --output-dir <scratch-dir>   # pull Braintrust docs as markdown
```

If local help differs from this skill, follow the installed binary's `--help`. `bt update` upgrades in place; ask before running it.

## Error Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Command hangs | Interactive picker or TUI opened | Add `--non-interactive` or `--no-input`; supply the missing name |
| 401 / auth error | OAuth token expired | Ask the user to run `! bt login --oauth --refresh` |
| Project not found | No default project, or wrong name | `bt projects list`, then pass `-p <exact-name>` |
| Empty `items` / `data` | Time window too short (`view logs` defaults to 1h) | Widen `--window` or the `created` bound |
| Trace not found | Passed a span row id as `--trace-id` | Use `root_span_id`, or `bt view span --id <row-id>` |
| `jq` parse error | stderr warnings mixed into stdout | Redirect `2>/dev/null` |
