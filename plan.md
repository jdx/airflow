## Goal

Achieve 100% compatibility with the current `.pre-commit-config.yaml` when using `hk migrate pre-commit`, so that running `hk pre-commit` (and `hk pre-push`) produces equivalent behavior.

References: migrate branch and features in `hk` PR: [feat: migrate pre-commit #318](https://github.com/jdx/hk/pull/318/files)

## Repo Context

- Source configs: `.pre-commit-config.yaml` in this repo
- Generated config: `hk.pkl` on branch `hk`

## Global configuration parity

- [ ] Fix `amends` / `import` to use the local hk pkl root (no absolute paths)
- [ ] Mirror top-level pre-commit `exclude: ^.*/.*_vendor/` as a global exclude in `hk.pkl`
- [ ] Confirm default stages include `pre-commit` and `pre-push` mapping as intended

## Stage modeling (manual vs automatic)

Pre-commit `stages: ['manual']` hooks must not run in normal `pre-commit`/`pre-push`.

- [ ] Introduce a manual-only mechanism (separate `hooks["manual"]` or gating) and move all manual hooks there
- [ ] Remove manual hooks from default `pre-commit`/`pre-push`/`check`/`fix` steps
- [ ] Document how to run manual hooks via hk

## Serial and always_run semantics

Pre-commit supports `require_serial: true` and `always_run: true`.

- [ ] Map `require_serial: true` to hk step grouping or an `exclusive` flag so only one step in the group runs at a time and ordering is preserved
- [ ] Emulate `always_run: true` for the few hooks that rely on it

## Hook mappings: builtins vs commands

Some hooks are currently placeholders and do nothing. Each requires a concrete `check` (and `fix` if applicable), or a reliable builtin mapping.

Placeholders to implement:

- [ ] `doctoc` (thlorenz/doctoc) with `--maxlevel 2` and scoped files
- [ ] Lucas-C `insert-license` variants (SQL, RST, CSS/JS/TS/TSX/PUML, Shell, TOML, Python, XML, Helm templates, YAML (non-Helm), Markdown, “other files“) honoring comment styles and license paths
- [ ] `blacken-docs` (with `black==25.9.0` and target versions/line length)
- [ ] `pretty-format-json` (with `--autofix --no-sort-keys --indent 4` and files scope)
- [ ] `check-xml` using xmllint behavior
- [ ] `check-builtin-literals`
- [ ] `rst-backticks` and `python-no-log-warn` (pygrep-hooks)
- [ ] `codespell` (echo+exec wrapper semantics and args)
- [ ] `zizmor` (GitHub workflow YAMLs)

For each above:

- [ ] Decide: hk builtin vs shell command
- [ ] Implement `check` (and `fix` if needed)
- [ ] Wire `files`/`exclude`/`types` exactly
- [ ] Ensure exit codes match pre-commit behavior

## Pygrep hooks: execution model

Many pygrep hooks were converted as raw regex text, not executable checks.

- [ ] Route all pygrep-like checks through a deterministic runner (e.g. `rg`/`grep`) with failure-on-match or failure-on-mismatch behavior that matches upstream
- [ ] Respect `pass_filenames`, include/exclude, and file type filters
- [ ] Verify regex flags (`(?x)`, case-insensitive, multiline patterns) are preserved

## Globs, types and file selection

- [ ] Fix malformed globs (e.g. `**/*.py` instead of `**/.py` or `**.py`)
- [ ] Apply `types` / `types_or` equivalents where used (e.g. YAML for `yamllint`, Python and Pyi for `ruff`)
- [ ] Ensure `files:` regexes remain regexes (not mistakenly treated as globs)
- [ ] Ensure `exclude:` regexes are faithfully included and scoped

## Arguments and runtime parity

Match pre-commit command arguments and behaviors:

- [ ] `ruff check`: include `--fix` in fix-mode (or equivalent) and `--force-exclude`
- [ ] `yamllint`: use `-c yamllint-config.yml --strict` and restrict to YAML types
- [ ] `bandit`: add `--skip B101,B301,B324,B403,B404,B603 --severity-level high`
- [ ] `pylint`: add `--disable=all --enable=W0133` (scope files as in pre-commit)
- [ ] `pretty-format-json`: add `--autofix --no-sort-keys --indent 4`
- [ ] `codespell`: add `--ignore-words=docs/spelling_wordlist.txt`, `--skip=...`, `--exclude-file=.codespellignorelines`

## Docker-based hooks

`shellcheck` uses `language: docker_image` in pre-commit.

- [ ] Implement `docker run` wrapper for `koalaman/shellcheck:v0.8.0 -x -a` (or depend on local binary)
- [ ] Ensure we pass files and handle excludes correctly

## Meta hooks

`meta` repo hooks are present in pre-commit:

- [ ] Map `identity` and `check-hooks-apply` or explicitly document their omission

## Regex vs glob fidelity

- [ ] Keep complex `(?x)` patterns as regex; avoid misplacing under `glob`
- [ ] For alternation patterns that can be expressed as lists, prefer `List()` where that improves clarity without changing behavior

## Global excludes and per-hook excludes

- [ ] Add repo-level `exclude: ^.*/.*_vendor/`
- [ ] Validate all per-hook excludes replicate upstream behavior (notably long `(?x)` excludes)

## Validation plan

- [ ] Run `pre-commit run --all-files` and capture results
- [ ] Run `hk pre-commit --all-files` and capture results
- [ ] Diff outputs and iterate until no meaningful differences remain (exit codes, modified files, messages)

### Per-hook behavior verification (post parity smoke test)

For each hook, verify failure and fixer behavior by modifying files to trigger failures, then running fixers and confirming identical outcomes vs pre-commit.

- [ ] For each hook: prepare minimal before/after fixtures to trigger failure
- [ ] Validate: `pre-commit run <hook> --all-files` failure output and exit code
- [ ] Validate: `hk pre-commit <hook> --all-files` matches failure output and exit code
- [ ] Validate fixer parity: run pre-commit fix vs `hk fix` and compare diffs
- [ ] Record status in the Hook Behavior Verification Matrix below

## Hook Behavior Verification Matrix

Columns: Hook ID | Repo/Origin | Entry/Builtin | Files | Exclude | Types | Has Fixer | Args | Pre-commit Fail Verified | hk Fail Verified | Pre-commit Fix Verified | hk Fix Verified | Notes

| Hook ID | Repo/Origin | Entry/Builtin | Files | Exclude | Types | Has Fixer | Args | PC Fail | HK Fail | PC Fix | HK Fix | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| check-merge-conflict | pre-commit-hooks | builtin | – | – | – | no | – | [ ] | [ ] | n/a | n/a | |
| debug-statements | pre-commit-hooks | builtin | – | – | python | no | – | [ ] | [ ] | n/a | n/a | |
| detect-private-key | pre-commit-hooks | builtin | – | providers/ssh/... | text | no | – | [ ] | [ ] | n/a | n/a | |
| end-of-file-fixer | pre-commit-hooks | builtin | – | see cfg | text | yes | – | [ ] | [ ] | [ ] | [ ] | |
| mixed-line-ending | pre-commit-hooks | builtin | – | – | text | yes | – | [ ] | [ ] | [ ] | [ ] | |
| check-executables-have-shebangs | pre-commit-hooks | builtin | – | – | file | no | – | [ ] | [ ] | n/a | n/a | |
| trailing-whitespace | pre-commit-hooks | builtin | – | see cfg | text | yes | – | [ ] | [ ] | [ ] | [ ] | |
| yamllint | adrienverge/yamllint | external | types: yaml | see cfg | yaml | no | -c ... --strict | [ ] | [ ] | n/a | n/a | |
| ruff | local | external | python,pyi | see cfg | python | yes | check --fix | [ ] | [ ] | [ ] | [ ] | |
| ruff-format | local | external | python,pyi | see cfg | python | yes | format | [ ] | [ ] | [ ] | [ ] | |
| bandit | local | external | airflow-core/src/... | exclude example_dags | python | no | --skip ... --severity-level high | [ ] | [ ] | n/a | n/a | |
| pylint | local | external | airflow-core/src/... | exclude example_dags | python | no | --disable=all --enable=W0133 | [ ] | [ ] | n/a | n/a | |
| pretty-format-json | pre-commit-hooks | external | chart/...schema.json | – | json | yes | --autofix --no-sort-keys --indent 4 | [ ] | [ ] | [ ] | [ ] | |
| doctoc | thlorenz/doctoc | external | see files | – | md,rst | yes | --maxlevel 2 | [ ] | [ ] | [ ] | [ ] | |
| codespell | codespell | external | text | see cfg | text | no | --ignore-words ... | [ ] | [ ] | n/a | n/a | |
| zizmor | woodruffw/zizmor | external | .github/{workflows,actions} | – | yaml | no | – | [ ] | [ ] | n/a | n/a | |
| shellcheck | docker image | external | .sh,.bash,... | exclude breeze autocomplete | shell | no | docker run koalaman/shellcheck -x -a | [ ] | [ ] | n/a | n/a | |
| rst-backticks | pygrep-hooks | grep | .rst | – | rst | no | regex | [ ] | [ ] | n/a | n/a | |
| python-no-log-warn | pygrep-hooks | grep | .py | – | python | no | regex | [ ] | [ ] | n/a | n/a | |
| check-xml | pre-commit-hooks | external | – | see cfg | xml | no | xmllint | [ ] | [ ] | n/a | n/a | |
| check-builtin-literals | pre-commit-hooks | external | – | – | py | no | – | [ ] | [ ] | n/a | n/a | |
| insert-license (variants) | Lucas-C | external | various | see cfg | various | yes | comment-style, license-filepath | [ ] | [ ] | [ ] | [ ] | one row per variant |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | |

Note: this table should be auto-augmented from `.pre-commit-config.yaml`. Initial rows cover critical/representative hooks; we will expand programmatically.

## Glob/Exclude edge-case verification

Add targeted checks to ensure matching behavior:

- [ ] Files in nested directories (e.g., `a/b/c/file.py`) match include globs
- [ ] Dot-prefixed files and directories (`.github/...`) are matched/excluded appropriately
- [ ] Alternation and verbose regex `(?x)` excludes include spaces and comments correctly
- [ ] Ensure global exclude `^.*/.*_vendor/` wins over per-hook includes
- [ ] Files under `**/dist/**` and `**/openapi-gen/**` are properly excluded where configured
- [ ] Literal filename matches (e.g., `LICENSES-ui.txt`, `pnpm-lock.yaml`) are respected

## Hook introspection and replication plan (doctoc, zizmor, etc.)

Goal: precisely replicate hook behavior by inspecting installed pre-commit hooks.

- [ ] Create a temp sandbox repo with minimal files per target hook
- [ ] Generate a focused `.pre-commit-config.yaml` containing only the target hook
- [ ] Run `pre-commit install` and `pre-commit run -a -v` to capture exact invocations
- [ ] Locate installed hook entry points in the pre-commit cache; capture wrapper and args
- [ ] Record command-lines, working directory assumptions, environment variables
- [ ] Prototype equivalent `hk` step command and validate outputs on the sandbox
- [ ] Port the working command into `hk.pkl` in this repo
- [ ] Add row(s) to the Hook Behavior Verification Matrix and mark verification boxes

## Notes

- Where `fix` behavior exists in upstream hooks, ensure `hk fix` reproduces it (including adding `--fix` flags for tools that require it)
- For performance, batch grep-based checks where possible without changing semantics
