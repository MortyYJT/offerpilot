# OfferPilot

An application-planning tool for master's degrees at Australian universities, built as the lead-capture
and first-screening front end of an education agency run by MortyYJT. Students explore programs, check
themselves against official requirements, and learn the application process on their own; those who
leave contact details are handed to the agency for human service. See `note/product/vision.md`.

The core surface is a **visual application roadmap**: open any phase and see what to prepare, when to
prepare it, and which official page the requirement came from.

## Current stage status

Status as of 2026-10-11. The roadmap of what comes next is `note/product/roadmap.md`.

On `main`:

- Onboarding flow, portfolio picker, roadmap interface and advisor chat skeleton in `web/`.
- A FastAPI service with PostgreSQL in `api/` persists the profile, the portfolio (as application rows)
  and roadmap task rows (stage 2, M1-M2). Only the onboarding position stays in `localStorage`.
- Material library (M3): upload, versions, archive, send for review. Review criteria must cite an
  official source; review verdicts are written by the maintainer CLI (`api/review_cli.py`), and the HTTP
  API is read-only for them. Specs live in `openspec/specs/`.

In progress, not on `main`:

- A static clickable prototype under `/prototype` (PR #16) that fixes the feature set before the backend
  is extended: school library, program library from CRICOS open data, background assessment,
  comparison, application board, advisor and hand-off to a human consultant.

Not implemented:

- There is no agent and no model call. The advisor chat appends the message and answers with a fixed
  notice rather than pretending an assistant replied.
- No official deadline is recorded. Every program carries `dataStatus: "待核验"` and the interface says
  so.
- Admission thresholds come from a first-pass manual seed set that mirrors `web/lib/programs.ts`. A guard
  fails the test run if the seed and that file diverge, but neither has been checked against a university
  page. This is placeholder data pending the annotation stage.
- No file content parsing (OCR or PDF extraction), and no automated review.
- The backend has no authentication. A profile is scoped to an anonymous cookie, so clearing browser data
  loses access to it.

## Local setup

~~~bash
make install     # install dependencies
make dev         # http://localhost:3000
make verify      # build plus end-to-end walkthrough screenshots
make help        # list every command
~~~

If `npm install` fails with `EPERM`, the shared npm cache contains root-owned files. `make install`
already points npm at a repository-local cache directory to avoid that.

## Product flow

```
Onboarding (ten steps, green progress bar with an x/y counter)
  -> "background captured, generating a plan"
  -> Portfolio selection (reach / steady / safe)
  -> Main interface: home | roadmap | profile | advisor
```

The **roadmap** is the main surface: six phases stacked vertically, each showing preparation materials,
a suggested date, the official deadline, and the source. Ticking a material updates the phase status.

## Data honesty rules

These exist because the previous implementation of this product generated a plausible-looking
verification date and an official-looking threshold with no source. Do not undo them.

1. Admission data is marked `待核验` (pending verification) in `web/lib/programs.ts`.
2. `SourceCitation.verifiedAt` is `null`, not a hardcoded date.
3. Every date is a **system suggestion** derived backwards from the intake term. Official deadline
   fields are left empty and rendered as "pending verification".
4. When the system cannot decide, it returns "needs manual review" instead of guessing a tier.

## Interface language

Interface copy is Chinese because the product targets Chinese applicants. Source comments,
documentation, and commit messages are English. See CONTRIBUTING.md.

## Checks

~~~bash
make build        # type check and production build
make screenshots  # full walkthrough in a real browser, writes docs/screenshots/
~~~

The walkthrough uses the `@playwright/test` devDependency in `web/` and runs locally only. It needs a
Chromium build: run `npx playwright install chromium` in `web/` once if it is missing.

## How this differs from a support-chat agent

Both projects use a deterministic workflow with an agent layer, but the object under management is
different. A support agent aims to **answer a question correctly**. OfferPilot aims to **keep a
multi-month application state correct**, where every fact has to trace back to an official page.

That pushes the architecture toward source versioning and human review, freshness gates, refusing to
answer when evidence is missing, and confirmation before irreversible writes.

## Project records

- [Contribution and commit rules](CONTRIBUTING.md)
- [Repository instructions for agents](AGENTS.md) and [agent guidelines](docs/engineering/agent-guidelines.md)
- [Current-truth specs](openspec/specs/) and [proposed changes](openspec/changes/)
- [English verification reports](docs/verification/)
- [Chinese notes index](note/README.md): vision, roadmap, decisions, handoff, design history, interview review
