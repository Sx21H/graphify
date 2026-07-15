# CLAUDE.md

Guidance for AI coding assistants working in this repository.

## What this project is

**graphify** (PyPI package `graphifyy`, CLI `graphify`) turns any folder of code,
docs, PDFs, images, or videos into a queryable **knowledge graph**. It ships as a
Python library *and* as a skill/plugin for 20+ AI coding assistants (Claude Code,
Cursor, Codex, Gemini CLI, Copilot, and more).

- **Code** is parsed deterministically with **tree-sitter ASTs** — no LLM, nothing
  leaves the machine. **Docs / PDFs / images / video** get an optional semantic
  pass through the host assistant's model or a configured API key.
- Every edge is labelled `EXTRACTED` (explicit in source), `INFERRED` (resolved by
  graphify), or `AMBIGUOUS` (flagged for human review).
- Output is a real graph you traverse (`graph.json`, `graph.html`, `GRAPH_REPORT.md`
  in `graphify-out/`), not a vector index.

Read `README.md` for the user-facing pitch, `ARCHITECTURE.md` for the pipeline, and
`SECURITY.md` for the threat model. `AGENTS.md` is the always-on instruction block
that graphify installs into host assistants (not repo-dev guidance).

## Development environment

This project uses **`uv`**. Everything runs through it — do not invoke `pip`/`python`
directly for project tasks.

```bash
uv sync --all-extras --frozen   # install deps exactly from the committed uv.lock
uv run pytest tests/ -q         # run the test suite
uv run graphify --help          # run the CLI
uv run pre-commit install       # enable local pre-commit hooks (skillgen + ruff)
```

- **`--frozen` matters.** `uv.lock` is committed and CI runs with `--frozen`. Never
  let a command re-resolve and churn the lock. If you add a dependency, edit
  `pyproject.toml` and update the lock deliberately.
- Python support floor is **3.10**; CI tests on 3.10 and 3.12.
- Optional features live behind extras in `pyproject.toml` (`mcp`, `pdf`, `watch`,
  `svg`, `leiden`, `video`, `neo4j`, `falkordb`, per-provider LLM backends, and
  language extras like `pascal`, `dm`, `terraform`, `sql`). Keep them optional —
  the default `uv tool install graphifyy` must stay lean.

## Repository layout

```
graphify/              the library + CLI (one module per pipeline stage)
  __main__.py          console entry point; handles install/uninstall dispatch
  cli.py               dispatch_command() — every non-install subcommand
  install.py           installs the skill into 20+ host assistants
  detect.py            collect_files(): which files to include, language classification
  extract.py           extract(): tree-sitter AST extraction (5k lines, the core)
  extractors/          per-language extractors migrated out of extract.py (see MIGRATION.md)
  build.py             build_graph(): extraction dicts -> nx.Graph
  cluster.py           community detection (Leiden/louvain)
  analyze.py           god nodes, surprising connections, suggested questions
  report.py            GRAPH_REPORT.md rendering
  export.py            graph.json / graph.html / graph.svg / Obsidian vault
  exporters/           html + graphdb (neo4j/falkordb) exporters
  serve.py             MCP server (stdio + HTTP) exposing the graph
  llm.py               semantic extraction backends (Anthropic, OpenAI, Ollama, ...)
  cache.py             semantic extraction cache (avoid re-paying for unchanged files)
  security.py          all external-input validation (URLs, paths, labels)
  validate.py          extraction-schema enforcement before build_graph()
  watch.py / hooks.py  file-watch + host-assistant hook integration
  skill*.md            generated skill bodies (one per host) — DO NOT hand-edit
  skills/<host>/       generated per-host reference sidecars — DO NOT hand-edit
  always_on/           generated always-on instruction blocks — DO NOT hand-edit
tools/skillgen/        generator + fragments that PRODUCE the skill*.md files
tests/                 pytest suite, one file per module; fixtures/ holds sample sources
worked/                worked-example corpora
docs/                  design notes, translations, how-it-works
```

## The pipeline (ARCHITECTURE.md is the source of truth)

```
detect() → extract() → build_graph() → cluster() → analyze() → report() → export()
```

Each stage is one function in its own module. Stages communicate through plain dicts
and NetworkX graphs — **no shared state, no side effects outside `graphify-out/`**.
The output directory name comes from `graphify.paths.GRAPHIFY_OUT` (overridable via
the `GRAPHIFY_OUT` env var); never hardcode the literal `"graphify-out"`.

### Extraction output schema

Every extractor returns `{"nodes": [...], "edges": [...]}`. Nodes carry
`id`, `label`, `source_file`, `source_location`; edges carry `source`, `target`,
`relation`, and a `confidence` of `EXTRACTED` / `INFERRED` / `AMBIGUOUS`.
`validate.py` enforces this schema before `build_graph()` consumes it.

## The CLI

The entry point is `graphify.__main__:main`. Install/platform subcommands are handled
in `install.py`; everything else flows through `graphify.cli.dispatch_command(cmd)`,
a large `if/elif cmd == ...` chain. Key commands: `install` / `uninstall` / `status`,
`extract`, `update`, `query`, `explain`, `path`, `reflect`, `affected`, `diagnose`,
`watch`, `export` (`html` / `obsidian` / `wiki` / `svg` / `graphml` / `callflow-html`),
`serve` (via the `graphify-mcp` script → `serve._main`), `benchmark`, `global`,
`merge-graphs` / `merge-driver`, and `provider`. When adding a command, add an
`elif` branch in `cli.py` and mirror argument parsing to the surrounding branches.

## The skill files are GENERATED — do not hand-edit

`graphify/skill*.md`, `graphify/skills/<host>/references/*.md`, and
`graphify/always_on/*.md` are **build artifacts**. The single source of truth is the
fragments under `tools/skillgen/fragments/`, rendered by `tools/skillgen/gen.py`.

```bash
uv run python -m tools.skillgen            # regenerate all host artifacts
uv run python -m tools.skillgen --platform claude
uv run python -m tools.skillgen --check    # fail on drift (this is what CI/pre-commit run)
uv run python -m tools.skillgen --bless    # rewrite tools/skillgen/expected/ after a fragment edit
```

A hand-edit to a generated file fails both the pre-commit hook and the CI
`skillgen-check` job. To change skill content: edit a fragment, run the generator,
then `--bless`, and commit the fragments + regenerated artifacts together.

## Adding a new language extractor

Follow `graphify/extractors/MIGRATION.md` and `ARCHITECTURE.md`:

1. Add `extract_<lang>(path: Path) -> dict` (new-style extractors go in
   `graphify/extractors/`; register in `extractors/__init__.py`'s `LANGUAGE_EXTRACTORS`).
2. Wire the suffix into `extract()` dispatch and `collect_files()` (`detect.py`).
3. Add the suffix to `CODE_EXTENSIONS` in `detect.py` and `_WATCHED_EXTENSIONS` in
   `watch.py`.
4. Add the tree-sitter package to `pyproject.toml` (keep it optional if it needs a C
   toolchain or ships platform-limited wheels — see the `dm` / `pascal` comments).
5. Add a fixture to `tests/fixtures/` and tests (`tests/test_languages.py` or a
   dedicated `test_<lang>.py`).

## Security

All external input passes through `graphify/security.py` before use: URLs via
`validate_url()` + a file:// redirect guard, fetched content via `safe_fetch()`
(size cap + timeout), graph paths via `validate_graph_path()` (must resolve inside
the output dir), node labels via `sanitize_label()`. Route new external input through
these helpers rather than validating inline. See `SECURITY.md`.

## Testing conventions

- One test file per module under `tests/`. Run `uv run pytest tests/ -q`.
- Tests are **pure unit tests**: no network calls, no filesystem side effects outside
  `tmp_path`. Keep new tests that way.
- Fixtures for extractor tests live in `tests/fixtures/`.
- `norecursedirs` (in `pyproject.toml`) excludes `worked/`, scratch corpora, etc.

## Lint & type checking

- **ruff** (`line-length = 100`, `target py310`) — conservative lint set; runs via
  pre-commit. `uv run ruff check --config pyproject.toml`.
- **pyright** basic mode over `graphify` + `tests`.
- **bandit** + **pip-audit** run in CI (currently non-blocking).

## Git & CI

- Development for this task happens on branch `claude/claude-md-docs-8t95jm`.
- CI (`.github/workflows/ci.yml`) runs three jobs: **skillgen-check** (artifact
  drift + coverage/round-trip validators, needs full history), **test** (3.10 & 3.12,
  plus an end-to-end `graphify install` smoke test), and **security-scan**.
- Commit fragment edits *and* regenerated skill artifacts together, or skillgen-check
  fails. Run `uv run pre-commit run --all-files` before pushing to catch drift early.
- `graph.json` has a registered merge driver (`graphify merge-driver`) — see
  `.gitattributes`.

## Working in this repo with graphify itself

Per `AGENTS.md`, this project keeps its own knowledge graph. If `graphify-out/`
exists: read `graphify-out/GRAPH_REPORT.md` before answering architecture questions,
prefer `graphify-out/wiki/index.md` over raw files when present, and run
`graphify update .` (AST-only, no API cost) after modifying code to keep the graph
current.
