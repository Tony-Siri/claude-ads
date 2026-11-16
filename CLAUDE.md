# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

`AGENTS.md` (imported above) is the canonical portable repository contract.
`ads/SKILL.md` is the product entrypoint. Load focused skills and references
only when the task routes to them.

## Commands

There is no task runner. Everything is plain CPython 3.11 or 3.12 plus pytest,
and dependencies install only from hash-pinned lock files:

```bash
python -m venv .venv            # on Windows use .venv/Scripts/python below
.venv/bin/python -m pip install --no-deps -e .
.venv/bin/python -m pip install --require-hashes --only-binary=:all: -r requirements.lock
.venv/bin/python -m pip install --require-hashes --only-binary=:all: -r requirements-dev.lock
.venv/bin/python -m pip install --require-hashes --only-binary=:all: -r .github/requirements-schema-tests.lock
```

| Task | Command |
| --- | --- |
| Full suite (run before a PR) | `python -m pytest -q` |
| Core subset (CI Python matrix) | `python -m pytest tests/core tests/release` |
| One test | `python -m pytest tests/core/test_scoring_engine.py -k <name>` |
| Repository audit: JSON, YAML, frontmatter, paths, sensitive content (run before a PR) | `python scripts/release.py audit` |
| Syntax checks | `python -m compileall -q claude_ads_core evals scripts tests mods/claude-ads-cockpit/tests` and `bash -n install.sh uninstall.sh` |
| Regenerate control registry and scoring profiles | `python scripts/build_control_registry.py` |
| Repository status, or the single next blocker (offline) | `python -m claude_ads_core status --root . --as-of YYYY-MM-DD` (or `next`) |
| Validate one contract file | `python -m claude_ads_core validate finding path/to/finding.json` |
| Deterministic release package | `python scripts/release.py package --output-dir DIR`, then `verify --output-dir DIR` |
| Cockpit companion | `python scripts/package_cockpit.py package\|verify --output-dir dist/cockpit` |

Test directories map to surfaces: `tests/core` (claude_ads_core), `tests/routing`
(skills, agents, evals), `tests/references` and `tests/audit` (reference
grounding, catalog, scoring math), `tests/scripts` (helpers and installers),
`tests/release` (packaging, docs hygiene, release gates). Installer tests are
slow and OS-sensitive; CI shards them with `.github/scripts/run_installer_ci.py`.
Required remote CI (`.github/workflows/ci.yml`, final `validate` job) must pass;
local success does not substitute for it.

## Architecture

Three layers, each checked by tests against the others:

1. **Instructions (Markdown).** `ads/SKILL.md` is the conductor. Its routing
   table maps every `/ads <subcommand>` to exactly one `skills/ads-*/SKILL.md`.
   Shared detail lives in `ads/references/` (per-platform `*-audit.md` control
   catalogs, creative specs, scoring). `agents/*.md` are bounded workers
   (per-platform `audit-*`, creative, research, verifiers) that return
   schema-valid JSON and never write final artifacts.
   `ads/agents/openai.yaml` is the Codex-facing interface metadata.
2. **Deterministic core (`claude_ads_core/`, stdlib only).** `cli.py` exposes
   `validate`, `score`, `portfolio`, `status`, `next`, `render`,
   `ingest-export`, and `cockpit`. Versioned schemas live in
   `claude_ads_core/schemas/v1/`. `scoring.py` and `control_registry.py` score;
   `reporting.py` renders Markdown, HTML, and PDF from one validated JSON run
   bundle; `adapters/` normalize CSV and native exports. `policy_root.py`
   locates policy beside the checkout, in a curated install, or via
   `--policy-root`, never from an unrelated working directory.
3. **Control plane (`control-plane/`).** Manifests are the source of truth that
   code and tests read: `capability-manifest.json` (what can actually be read or
   changed), `source-ledger.json` and `claim-ledger.json` (dated evidence),
   `control-registry.json` and `scoring-profiles.json`, `safety-policy.json`,
   `maturity-status.json`, and `ecosystem-dispositions.json`. Their schemas are
   in `control-plane/schemas/`.

### Cross-cutting rules enforced by tests

- **The control registry is generated.** `scripts/build_control_registry.py`
  builds `control-registry.json` and `scoring-profiles.json` from
  `tests/fixtures/check-catalog.yaml`, the per-platform reference tables, and the
  claim ledger. `tests/core/test_control_registry.py` fails if the checked-in
  files differ from a fresh build. Edit the inputs and rerun the script; never
  hand-edit those two manifests.
- **Scoring fails closed.** All twelve platform health profiles are disabled and
  every control is `informational` with `watchlist` or `opportunity` behavior.
  Results are `pass`, `fail`, `unknown`, or `not_applicable`; health, evidence
  coverage, regulatory exposure, and opportunities stay separate. Enabling a
  profile or adding a threshold needs the sourcing rules in `CONTRIBUTING.md`.
- **Reference claims are grounded.** `tests/audit/test_source_grounding.py`
  requires runtime references to cite registered official HTTPS sources,
  reciprocal source and claim ledger entries, fresh claims, and no brittle
  thresholds or mutable dates in registry rows.
- **Routing is evaluated.** `tests/routing/` checks skills against
  `evals/platform-workflow-routing.json` and `evals/v2-behavior-evals.json`. A new
  skill or route needs positive, near-miss, ambiguous, and collision cases;
  high-risk behavior also goes in `v2-behavior-evals.json`.
- **Skill and agent frontmatter.** `SKILL.md` frontmatter holds only `name` and
  `description`; keep it under about 500 lines. Agents declare a
  least-privilege tool allowlist and must not set `model:` (the host selects
  it). `tests/audit/test_agent_frontmatter.py` also expects exactly 25 agents
  and requires `ads/SKILL.md` to keep its host-model-inheritance wording.
- **Installers own files by manifest.** `install.sh` and `install.ps1` are
  parallel implementations (PowerShell must also run on Windows PowerShell 5.1).
  They create a managed venv with `managed-runtime-receipt.json` and an
  ownership manifest; `uninstall.*` removes only manifest-owned files and must
  leave unrelated skills untouched. A change to one installer usually needs the
  mirror change and coverage in `tests/scripts/test_installer_security.py`.
- **Root docs are tested.** `tests/release/test_public_docs_hygiene.py` bans en
  and em dashes in README, CONTRIBUTING, SECURITY, SUPPORT, the installers, and
  the PR template, and checks that README status claims match the scoring
  manifest.

### Conventions

- Keep platform facts and volatile guidance in dated references, not in skill
  entrypoints or root instructions; use progressive disclosure.
- Derive counts and capability status from the manifests instead of
  hand-maintaining them in prose.
- Python tools expose a CLI, structured output, and tests proportional to risk.

### Packaging and plugins

`.claude-plugin/marketplace.json` lists two plugins: `claude-ads` (repository
root) and the optional `ads-cockpit` companion in `mods/claude-ads-cockpit/`
(Python side in `claude_ads_core/cockpit*.py`). `claude plugin validate .`
reports one expected error because names starting with `claude-` are reserved;
any other validation error is a defect. Release readiness needs
`python scripts/release.py gate` with externally signed review evidence and a
CI run ID, per `control-plane/RELEASE_REQUIREMENTS.md`.
