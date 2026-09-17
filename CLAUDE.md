# Working rules for AI coding assistants

This file is read automatically by Claude Code. It holds only generic, public working rules.
Project-specific context (AOIs, storage paths, run ids, decisions, next steps) lives in
`CLAUDE.local.md`, which is gitignored and never committed. Read it first if it exists.

## Talking to the user

- Reply in Roman Urdu: simple, structured, nothing hidden.
- Areas always in **acres**, never m² or hectares. 1 pixel (10 m) = 0.025 acre; the 5×5 window = 0.62 acre.
- Give critical feedback: point out flaws, offer alternatives with their tradeoffs, ask clarifying questions.
- Keep the user updated during long work.

## Before acting

- Confirm first: Earth Engine exports, bulk downloads, cloud-storage uploads, `git push`, deleting or
  overwriting data.
- KISS / YAGNI: build the simplest thing that meets today's need; ask before adding complexity.
- Time every long step, log per-stage timings, find the slow stage and optimise it proactively.
- Never hard-code CPU, RAM or threads; use `sar_pipeline.resources` (container-aware).

## Git

- Commit only with the repo-local identity (check `git config user.name` before committing).
- No AI attribution in commits or PRs: no `Co-Authored-By`, no session links.
- This repository is **public**: no client, region, bucket, project or track names, no coordinates,
  no emails in tracked files. `tests/test_repo_hygiene.py` checks this against `secrets/private_terms.txt`.
- Never commit `secrets/`, `data/`, `processed/`, `reference/`, non-example configs, or local changes
  to `notebooks/04_pixel_explorer.ipynb`.
- Code and docs in English, written so a junior can follow from the basics.

## Where things are

- Pipeline CLI: `python -m sar_pipeline --config <yaml> <step>`; analysis CLI:
  `python -m sar_pipeline.analysis <step> --config <yaml>`.
- Docs: `docs/04_runbook.md` (pipeline), `docs/08_analysis.md` (ground truth, model, field labels).
- Tests: `pytest` (no network).
