# AGENTS.md

Guidance for coding agents working in this repository. Each experiment's own
README stays authoritative for how that experiment is installed and run.

## What this is

《深入理解 AI Agent：设计原理与工程实践》 ("AI Agents in Depth"), a Chinese-language
open-source book, plus the companion experiments for its ten chapters. The
Chinese manuscript in `book/` is the source of truth; translated editions
(`book-en/`, `book-zhtw/`, `book-ja/`, ...) are synced from it after a change
merges. Prose (manuscript, READMEs, ledgers) is Chinese-first: write in the
language of the file you are editing.

## Layout

| Path | What it holds |
| --- | --- |
| `book/` | Canonical Chinese manuscript (`chapter1.md`…`chapter10.md`, `introduction.md`, `afterword.md`, `reference-answers.md`), SVG figures in `images/`, pandoc/XeLaTeX build files |
| `book-<locale>/` | Translated editions, one directory per language |
| `chapter1/` … `chapter10/` | Companion experiments, one subdirectory each. `chapterN/README.md` maps "实验 N-M" numbers to directories; `chapterN/EXPERIMENT_LEDGER.md` records gates, status and canonical evidence |
| `agentbook/` | The installable package: provider registry and resolution only |
| `tests/` | Repo-root regression tests, mostly `test_ch<N>_<experiment>_<topic>.py` |
| `docs/` | `EXPERIMENT_CONVENTIONS.md`, `EXPERIMENT_STATUS.md`, `STATIC_SITE_I18N.md`, plus per-locale `README` and `LEARNING` files |
| `mkdocs.yml`, `scripts/`, `extras/`, `overrides/` | MkDocs reading site: assembly script, build hooks, browser-side i18n, theme overrides |
| `web-astro/` | Astro web reader (npm, TypeScript) |
| `slides/` | Generated Slidev course (`.mjs` generators plus `lesson-NN.md`) |
| `skills/` | Chapter knowledge distilled into single-file `SKILL.md` packs that link to the matching experiment code |

## Environment

- Python 3.11–3.13. Install per chapter from the repo root with
  `uv sync --locked --extra chN` (add `--extra dev` for pytest) or
  `python -m pip install -e ".[chN,dev]"`. `uv sync` installs exactly the extras
  named, so combine them in one command. CI installs extras on 3.11 and 3.13 and
  runs tests on 3.12.
- Heavy stacks are opt-in extras, not part of `all`: `train`, `audio`, `unsloth`,
  `vllm`. `unsloth` and `vllm` conflict with each other, and some training
  experiments (`chapter7/sesame`, `chapter8/prompt-distillation`) need their own
  `requirements.txt` in a separate venv.
- Any `pyproject.toml` dependency change needs `uv lock`. CI runs
  `uv lock --check`, and the lock must resolve on macOS arm64, Linux x86_64 and
  Windows.
- Credentials: copy `.env.example` to `.env` at the repo root and set at least one
  provider key. Some experiments expect a `.env` next to them or exported
  variables instead; follow their README.

## Commands

Run from the repo root unless stated otherwise.

```bash
# Repo-root regression suite. Tests put experiment directories on sys.path with
# relative paths, so run it from the root, with API-key variables unset.
python -m pip install -e ".[dev]" "mkdocs>=1.6,<2" "psycopg2-binary>=2.9"
python -m pytest tests -q

# One experiment's own tests (layout varies: tests/ or root-level test_*.py)
cd chapter2/system-hint && python -m pytest -q

# Lint and format. Ruff (line length 100) is configured but not run in CI.
ruff check <paths> && ruff format <paths>
cd chapter1/web-search-agent && python -m black --check tests   # the only format check in CI

# Lockfile and multi-language README consistency
uv lock --check
python scripts/check_i18n_consistency.py
python -m pytest tests/test_reference_answer_coverage.py -q
python scripts/site_i18n.py                                   # needs mkdocs-material

# Reading site (assembles _web/, which mkdocs.yml uses as docs_dir)
python -m pip install -r requirements-docs.txt
python -m pytest tests/test_homepage_index.py -q
bash scripts/build_site.sh && mkdocs build -d site

# Astro reader (Node >= 22.12)
cd web-astro && npm ci && npm run check && npm run build && npm test

# Book artifacts: pandoc + XeLaTeX + ElegantBook for the PDF; poppler for the
# EPUB, which takes its cover from the PDF's first page.
cd book && bash build_pdf.sh
./build_epub.sh zh-CN
```

## Boundaries

- `agentbook/` is plumbing only. Teaching code stays inside the chapter
  experiment directory so a reader can follow it top to bottom.
- `agentbook.providers` modules import only downward: `models` → `openrouter` →
  `registry` → `resolution` → `legacy`. Data lives in `registry.py`; policy
  (precedence, fallback) lives in `resolution.py`.
- Adding a provider is one `PROVIDERS` entry in `registry.py`; chapter CLIs
  build their `--provider` choices from `SUPPORTED_PROVIDERS`. The same change
  also updates `.env.example`, the key tables in `README.md` and
  `docs/en/README.md`, and `tests/test_providers.py`. A provider that can serve
  `gpt-5*` directly must be listed in `supported_gateways` in `resolution.py`,
  or those requests are rerouted through OpenRouter.
- Never hand-edit generated output: `_web/` (assembled from sources by
  `scripts/build_site.sh`), `site/`, `_web/extras/site-i18n.generated.js`, and
  the chapter cards that `scripts/homepage_index.py` expands between the
  `<!-- book-chapter-index -->` markers in `index*.md`.
- Vendored third-party trees (`chapter2/prompt-engineering/tau_bench`,
  `chapter9/browser-use-rpa/browser-use`) are excluded from ruff and from the
  bare-`except` check. Do not reformat or restyle them.

## Conventions

- Experiments are independent teaching projects. Use the target layout in
  `docs/EXPERIMENT_CONVENTIONS.md` for new ones; when editing an old one, match
  its existing layout instead of restructuring it. Resolve endpoints and keys
  with `agentbook.providers.resolve_backend` rather than adding provider
  branches inside an experiment.
- No bare `except:` in book-owned code (`tests/test_no_bare_except.py`).
- Automated tests are offline: no live API calls, network, GPU-only paths or
  model downloads. Use fixtures and mocks. Live smoke scripts go in
  `tests/manual/` and must not be named `test_*.py`.
- `.gitignore` ignores `.*` (every dotfile except a few named ones), `results/`,
  `output/`, `out/`, `*.wav`, `*.mp3`, `*.mp4`, `site/` and `_web/`. Run outputs
  are not committed unless they are deliberate fixtures or retained evidence,
  and a new dotfile or dot-directory needs an explicit `!` rule to be tracked.
- Never put API-key values in committed files or evidence. Record only the
  environment-variable name.
- A fallback model is a separate experimental arm, not a reproduction of the
  reference model. Fail closed when an endpoint drops required modalities,
  schemas or tools.
- Installing, authenticating, launching a browser or passing a smoke test does
  not make an experiment complete. Statuses in `docs/EXPERIMENT_STATUS.md` and
  the `chapterN/EXPERIMENT_LEDGER.md` files reflect retained evidence only.
- Manuscript headings carry no manual numbers; pandoc's document class numbers
  them. Every reflection question in `book/chapterN.md` needs a matching answer
  in `book/reference-answers.md`.
- README translations must stay aligned. If a language has its own main README,
  CI requires all ten chapter READMEs, a `LEARNING.md`, the same per-chapter
  project count as the Chinese edition, and the same number of `git clone`
  commands. Adding or removing an experiment row means editing every
  `chapterN/README.<locale>.md`.
- The site's languages are defined twice and must agree: `extra.languages` in
  `mkdocs.yml` and `extras/site-nav-i18n.json`. The MkDocs hook reruns
  `scripts/site_i18n.py` and refuses to build on drift.

## CI and pull requests

Workflows live in `.github/workflows/` and most are path-filtered. `agentbook/**`
triggers the repo-root suite (in `web-search-agent-tests.yml`), the
provider-adoption suite and chapter 1's web-search-agent suite; `tests/**`
triggers the repo-root suite. Publishing (Pages deploy, release upload, the
star-history job) is gated to the canonical repository `bojieli/ai-agent-book`
and never runs elsewhere.

This checkout is a personal copy: `origin` is `d16ug-a1l/ai-agent-book` and
`upstream` is the original author's repository. Push only to `origin`.

Upstream history is squash-merged, with PR titles in conventional-commit form
(`fix(ch5/video-edit): …`, `feat(providers): …`, `docs(book): …`).

## Read before changing sensitive areas

- Experiment layout, tests, fixtures or evidence:
  `docs/EXPERIMENT_CONVENTIONS.md`, then the chapter's `EXPERIMENT_LEDGER.md`.
- Experiment status or evidence claims: `docs/EXPERIMENT_STATUS.md`.
- Site languages or navigation: `docs/STATIC_SITE_I18N.md`.
- EPUB editions: `EPUB.md`.
- Chapter 4 MCP servers: `chapter4/DOCKER_DEPLOYMENT.md`.
- A topic's design rules: the matching `skills/<name>/SKILL.md`.

## Platform notes

The shell scripts here are bash. This workspace is on Windows, so run them from
Git Bash and invoke Python as `python`; scripts that call `python3` need it on
`PATH`. The PDF/EPUB toolchain is only exercised on macOS in CI.
