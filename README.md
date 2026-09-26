# FLN — Foundational Literacy & Numeracy Assessment Platform

A large-scale, personalized assessment system that helps teachers measure, track, and improve every student's Foundational Literacy and Numeracy (FLN) outcomes — from automatic question paper generation to scanning answer sheets and instant, profile-driven evaluation.

---

## Table of Contents
- [What is FLN?](#what-is-fln)
- [Why FLN Matters](#why-fln-matters)
- [Initiatives](#initiatives)
- [What This Software Does](#what-this-software-does)
- [Curriculum & Level Framework](#curriculum--level-framework)
- [How It Works (Workflow)](#how-it-works-workflow)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Contribution Guidelines](#contribution-guidelines)
- [Branching & PR Convention](#branching--pr-convention)
- [License](#license)

---

## What is FLN?

**Foundational Literacy and Numeracy (FLN)** refers to the basic ability to read with comprehension and perform simple arithmetic operations — the core skills every child needs before they can meaningfully engage with the rest of their school curriculum. This platform covers the **Mathematics (numeracy) half only**, from the **Balvatika / Foundational Stage (Preschool 1–3, ages 3–6) through Class 4 (ages 9–10)**, and includes skills like number sense, counting, spatial sense, measurement, patterns, and elementary arithmetic. See [Curriculum & Level Framework](#curriculum--level-framework) for how that range is expressed as levels.

FLN is considered the "foundation" of all future learning — without it, a child cannot effectively progress through later grades, no matter how good the rest of the curriculum is.

## Why FLN Matters

India has one of the largest school-going populations in the world, but enrollment has not translated into actual learning. Large-scale assessments have repeatedly shown that a significant share of children in upper primary grades cannot read a simple grade-appropriate text or solve basic arithmetic problems. This learning gap compounds over time — children who fall behind in FLN tend to struggle increasingly with every subject built on top of it, leading to disengagement, grade repetition, and eventually dropout.

The National Education Policy (NEP) 2020 explicitly recognized this and stated that achieving universal foundational literacy and numeracy in primary school is the highest near-term priority for the Indian education system — without it, the rest of education policy has limited impact for a large portion of students.

This is the problem our project aims to help solve: giving schools and teachers a reliable, scalable, and personalized way to **assess** where each child stands on FLN, **act** on that data quickly, and **track** progress until every child clears the foundational bar.

## Initiatives

Some of the key national and state-level efforts this project aligns with:

- **NIPUN Bharat** (National Initiative for Proficiency in Reading with Understanding and Numeracy) — launched in July 2021 under the Samagra Shiksha scheme, with the goal that every child achieves grade-level FLN competencies by the end of Grade 3, by 2026–27. It uses a five-tier implementation structure (national, state, district, block, school).
- **NEP 2020** — the policy mandate that established universal FLN as the top priority for the Indian school system.
- **DIKSHA & UDISE+** — existing national digital infrastructure for teacher resources and student/school data that FLN initiatives are encouraged to build on or align with.
- **State-led missions** — several states have their own FLN programs aligned with NIPUN Bharat (e.g., Mission Buniyaad in Delhi, Mission Ankur in Madhya Pradesh), often with localized assessment tools and workbooks.

This project is built to be usable by schools, teachers, and administrators operating within this broader policy ecosystem — generating assessments aligned with grade-wise FLN expectations ("Lakshyas") rather than a generic test.

## What This Software Does

The platform is built around **personalized, student-specific assessment**, not one-size-fits-all testing. Core capabilities:

- **Student Profiling** — every student has a profile that tracks their current FLN level, assessment history, and progress over time.
- **Teacher Dashboard** — central workspace for teachers to manage classes, generate assessments, scan results, and view analytics.
- **Automatic Question Paper Generation** — question papers are generated automatically based on grade level and the student's current FLN level, not just a static template.
  - For a **new class/new school** with no prior data, the system falls back to a **standard question paper** aligned with the generic FLN benchmark expected for that grade.
  - Once a student has a profile, future papers are **personalized**, while still meeting the minimum competency bar defined for that grade under FLN criteria.
- **Print & Distribute** — teachers can print a generic class paper or individual, name-tagged worksheets per student.
- **Scan & Auto-Evaluate** — after collecting completed sheets, the teacher scans them (via phone camera or a school scanner) and the system evaluates them automatically.
- **Instant Results & Certification**
  - If a student **clears** the FLN benchmark for their grade → they receive a certificate for that grade and progress forward.
  - If a student **does not clear** it → they receive a detailed analysis of which FLN level they're actually at, along with a scheduled re-assessment date for the appropriate (lower) level.
  - Students who clear a lower-level re-assessment go on to attempt the FLN qualifier for their original grade again — every subsequent paper is generated from their updated, personalized profile.

## Curriculum & Level Framework

A child's FLN standing is a position on a **93-level curriculum**, not a grade. The taxonomy is **Balvatika-first**: it is built outward from the Balvatika (age 5–6) exit competency, and the build order runs Balvatika → Class 1 → Class 2 → Class 3 → Class 4.

| Stage | Band | Ages | Levels | Concept IDs | Testable | Oral-only |
|---|---|---|---|---|---|---|
| 1 | Preschool 1 | 3–4 | 1–7 | S1.1–S1.7 | 7 | 0 |
| 2 | Preschool 2 | 4–5 | 8–17 | S2.1–S2.10 | 10 | 0 |
| 3 | Preschool 3 / **Balvatika** | 5–6 | 18–27 | S3.1–S3.10 | 10 | 0 |
| 4 | Class 1 | 6–7 | 28–42 | S4.1–S4.15 | 15 | 1 (poems) |
| 5 | Class 2 | 7–8 | 43–61 | S5.1–S5.19 | 19 | 1 (riddles) |
| 6 | Class 3 (★ MPL) | 8–9 | 62–75 | S6.1–S6.14 | 14 | 0 |
| 7 | Class 4 | 9–10 | 76–93 | S7.1–S7.18 | 18 | 0 |
| | | | **1–93** | **S1.1–S7.18** | **93** | **2** |

*(Stage boundaries and counts are taken verbatim from [`Research/fln_level_networks.md`](Research/fln_level_networks.md) Part 1, and match `backend/src/config/curriculumMap.ts` — 93 levels, 93 unique concept IDs.)*

Five things about this framework are worth knowing before you touch it. Four are settled decisions; the fifth is an open question that is *deliberately* left open, and must stay that way:

- **The count is not fixed.** 93 is a real count from decomposing the research into ten strand-chains, not a target — the research says so explicitly. Nothing should hardcode it. Read the count from the registry.
- **One concept = one level = one graph node.** `S1.1`–`S7.18` are immutable identities stamped on every generated question. Re-ordering levels changes a level number, never a concept ID, so a failed question is looked up by concept ID with no level arithmetic or name matching.
- **Two notations exist, and they are cross-checked.** *S-notation* (`S1.1`) is the research/concept identity; *L-notation* (`L1`–`L93`) is the level identity used by the skill map. `L(n)` is the n-th S-code in stage-then-index order, computed in code, and `npm run check:level-notation-drift` reports drift if the code and the reference crosswalk disagree.
- **Prerequisite edges are typed, and only one type is a dependency.** Each edge is `prereq` (hard cognitive dependency), `sequence` (teaching order only — no inference in either direction), or `parallel` (co-equal, no dependency). Only `prereq` edges are reproduced in the code table; running inference over the other two produces false conclusions.
- **How a concept's multiple prerequisites combine is not yet decided.** Nine concepts have more than one `prereq` parent (e.g. `S2.1 ← S1.1 + S1.3`, `S6.5 ← S5.4 + S5.5 + S6.1`). The research and the code both store a plain list with no AND/OR marker and neither document commits to a reading, so **no doc should assert one**. What *is* decided: a concept with no `prereq` edge has no inferred prerequisite, and the graph is validated as a DAG (known concept IDs, no cycles) at server start-up.

### Build state: the 59 → 93 migration is in progress

The 93-level taxonomy is registered server-side, but the worksheet renderer (`backend/fln-backend/`, a standalone Puppeteer service) still implements the retired **1–59** space and throws above 59. So level recommendations are currently **capped at 59**, and levels 60–93 are specified but not yet renderable end to end. To bridge the two, each 93-space level record carries a `legacyLevel59` pointer (null when unmapped), which is what `/api/curriculum` and `/api/question-bank` use to report content status and question-bank coverage. Proposed crosswalk: [`Research/fln_59_to_93_crosswalk.PROPOSED.md`](Research/fln_59_to_93_crosswalk.PROPOSED.md).

| Concern | Source of truth |
|---|---|
| Level ⇄ concept registry (93 levels, stages, strands) | `backend/src/config/curriculumMap.ts` |
| Level ⇄ skill map, L-notation (L1–L93) | `frontend/src/data/skillProgressionMap.ts` |
| L ↔ S crosswalk (machine-checked) | `Research/fln_L_to_S_crosswalk.json`, checked by `scripts/check-level-notation-drift.ts` |
| Prerequisite edge table | `backend/src/competencyPrerequisites.ts`, generated from `Research/fln_level_networks.md` Part 2 |
| The research behind the count and the chains | [`Research/`](Research/) |
| Normative requirements | [SRS.md](SRS.md) §1.8 |
| How a level becomes an actual paper | [PIPELINES.md](PIPELINES.md) |

## How It Works (Workflow)

1. Teacher generates a question paper from the dashboard (standard paper for new classes, or personalized per student once profiles exist).
2. Paper is printed and distributed to students.
3. Students take the assessment on paper.
4. Teacher collects the answer sheets.
5. Teacher scans the sheets (phone or scanner) and uploads them into the app.
6. System auto-evaluates the sheet and updates the student's profile.
7. Teacher gets an instant result:
   - **Pass** → certificate issued, student advances.
   - **Fail** → FLN level diagnosis + scheduled re-assessment at the appropriate level.
8. Cycle repeats until the student clears the grade-level FLN qualifier.

## Tech Stack

An **npm-workspaces monorepo** — one `npm install` at the root covers every package.

- **React + Vite** (`frontend/`) — SPA. Talks to the real backend over `/api/*`; there is no in-browser mock.
- **Node + Express + TypeScript** (`backend/`) — the API, and the only place business logic lives. Modular domain routes per [ADR 001](docs/adr/001-backend-structure.md), signed-JWT auth, role scoping.
- **MongoDB *or* a local JSON store** — the native `mongodb` driver is used when `MONGODB_URI` is set; otherwise `backend/data/db.json` keeps the server zero-config. Both write paths go through one store, so treat `db.json` as dev-only: it is rewritten in full per mutation and is not concurrency-safe.
- **Puppeteer + pdf-lib** — HTML → A4 worksheet rendering, plus a standalone renderer service (`backend/fln-backend/`) that produces worksheets, answer keys and OMR coordinates.
- **Google Gemini** (`@google/genai`) — worksheet evaluation narrative and next-level recommendation, plus LLM-assisted misconception clustering. See [AGENT_ARCHITECTURE.md](AGENT_ARCHITECTURE.md).
- **Python** (`ai-services/`) — currently only `scripts/pdf_rasterize.py`, for rasterizing scanned PDFs before OCR. The older LLM pipeline in that folder is retained for reference and is not invoked by the backend; see [`ai-services/PIPELINE.md`](ai-services/PIPELINE.md).

More detail lives in the dedicated docs: system shape in [ARCHITECTURE.md](ARCHITECTURE.md), the four end-to-end flows in [PIPELINES.md](PIPELINES.md), and the AI/LLM surface in [AGENT_ARCHITECTURE.md](AGENT_ARCHITECTURE.md).

## Getting Started

```bash
git clone https://github.com/lucky-pluton/fln.git
cd fln
npm install
```

### Run against your own MongoDB (recommended for local dev)

Each contributor should point their local backend at **their own** MongoDB — either
a free [Atlas](https://www.mongodb.com/cloud/atlas/register) cluster or a local
`mongod` — instead of hardcoding data or sharing one database. This lets you seed
your own test data and iterate on features without touching anyone else's.

1. Copy the backend env template: `cp backend/.env.example backend/.env`
   (the file at the repo root, `.env.example`, is only for the AI scripts in
   `ai-services/` — it does **not** configure the database).
2. In `backend/.env`, set `MONGODB_URI` to your own connection string, e.g.
   `mongodb+srv://<user>:<pass>@<cluster>.mongodb.net/fln` (Atlas) or
   `mongodb://127.0.0.1:27017/fln` (local mongod).
3. Populate it with the full demo dataset (states/districts/schools/teachers/
   volunteers/students — matches the demo login buttons in the UI):
   ```bash
   npm run seed --workspace @fln/backend
   ```
   Optionally also run `npm run seed:question-bank` and `npm run seed:html`
   (workspace-scoped) to load the question bank / worksheet HTML collections.
4. Start the app:
   ```bash
   npm run dev:backend    # API on :3000, reads backend/.env
   npm run dev:frontend   # Vite dev server on :5173
   ```

Demo login after seeding: `superadmin@fln.org`, password `Fln@2026` (see
`backend/src/seed.ts` for the full list of generated teacher/volunteer/admin
emails, which follow a predictable `role.<state>_<district>_<block>_<school>@fln.org`
pattern).

### Aadhaar tokenization (in-process vault)

Student registration tokenizes the 12-digit Aadhaar through the in-process
vault module at [`backend/src/modules/vault/`](backend/src/modules/vault/) —
the FLN backend never stores plaintext Aadhaar and never exposes Vault
service JWTs to the browser (see
[`backend/src/aadhaarVault.ts`](backend/src/aadhaarVault.ts) and
[`backend/src/routes/students.ts`](backend/src/routes/students.ts)). The
module is wired unconditionally at boot; no feature flag, no separate
process, no service-JWT exchange.

The module needs two env vars (both required for tokenization to succeed):

- `MONGODB_URI` — the FLN backend's existing Mongo connection. The vault
  reuses the same replica set; the module fails fast with
  `VAULT_DB_REQUIRES_REPLICA_SET` (503) if pointed at a standalone
  `mongod` because `session.withTransaction(...)` is unsupported there.
- `LOCAL_DEV_MASTER_KEY` — base64; ≥ 32 decoded bytes. The
  per-record DEK wrap subkey is derived from this via
  `HKDF-SHA-256(master, salt=context, info="aadhaar-vault/dek-wrap")`.
  Production deployments are expected to swap `LocalDevKeyManager` for a
  real KMS provider; the port is stable.

Until both are set, `POST /api/students` and `POST /api/students/bulk-import`
fail with `VaultError NOT_CONFIGURED` (by design — no plaintext fallback).

End-to-end contract is enforced by the integration test suite at
[`backend/tests/aadhaar-hardening.test.ts`](backend/tests/aadhaar-hardening.test.ts)
and [`backend/tests/aadhaar-detokenize.test.ts`](backend/tests/aadhaar-detokenize.test.ts)
(run with `npm test` from `backend/`), and by the read-only at-rest audit
at [`backend/scripts/audit-aadhaar-at-rest.ts`](backend/scripts/audit-aadhaar-at-rest.ts)
(`npm run audit:aadhaar`).

## Rules
 Contributor Onboarding — Onboarding Document (Mandatory)

    Every student contributing to the FLN project is required to submit an Onboarding Document (.md) before their first pull request.
    The document is a record of your understanding of the project and your plan for contributing to it. Submissions that omit any of the sections below will be returned for revision.

    Purpose:

    The Onboarding Document exists to ensure that every contributor:

    1. Has a working understanding of what FLN is and the problem it solves.
    2. Has read the existing codebase and can describe its current state in their own words.
    3. Has independently identified weaknesses, gaps, and risks in the current implementation.
    4. Has formed opinions and proposed ideas for improving the project.
    5. Has a concrete plan for tackling at least one identified gap.
    6. Has produced a tangible contribution (code, documentation, tests, or design) that advances the project.

    Reading the code without forming a view is not enough. The document is intended to surface misunderstanding early and to surface good ideas quickly.

    File Naming and Location:

    - File name: ONBOARDING-<your-name>.md 
    - Location: you have to make the PR in the Ideas folder .
    - Format: Markdown (.md).( PDF, .docx, or plain .txt will not be accepted.)

    Required Sections:

    The document must contain the following six sections, in this order.

    1. What is FLN?

    Describe, in your own words, what FLN stands for, the domain it operates in (Foundational Literacy and Numeracy / education), the population it
    serves, and the problem it aims to solve. Do not copy the project description verbatim — paraphrase it. A reader who has never heard of FLN should be
    able to understand the project's purpose from this section alone.

    2. What do you understand by FLN (as a system)?

    Go beyond the mission statement. Describe FLN as a system: the users (students, teachers, administrators, superadmins), the main entities (schools,
    classes, assessments, worksheets, certifications), and the high-level flow of data through it. This section is about demonstrating that you
    understand how the pieces fit together, not just what the project is for.

    3. Current State of the Repository — What Has Been Done So Far:

    Walk through the repository and describe what already exists:

    - Tech stack (frontend, backend, database, auth, deployment).
    - Implemented features (authentication, role-based access, dashboards, worksheet generation, OMR, analytics, etc.).

    4. Gaps Observed in the Code:

    This is the most important section. List concrete weaknesses, bugs, missing features, or design problems you found while reading the code. You can also pick issues which are stated on FLN git repo and solve them. For each
    gap, include:

    - Where — file path and line range or component.
    - What — what is wrong or missing.
    - Why it matters — the impact on users, maintainability, performance, or correctness.

    5. Ideas for the Project:

    Propose improvements, new features, or refactors that would make FLN better. Each idea should include:

    - What — the proposed change in one or two sentences.
    - Why — the problem it solves or the value it adds.
    - How — a sketch of the implementation


    6. Your Contribution:

    Describe the actual work you have done as part of this onboarding. A contribution can be any of:

    - A bug fix.
    - A new feature or endpoint.
    - A refactor.
    - Tests (unit, integration, or end-to-end).
    - Documentation (this onboarding document counts only if it is exceptional; the document itself is mandatory, not the contribution).
    - A design document or architectural proposal.

    Review Criteria

    A reviewer will check the Onboarding Document against the following:

    - All six sections are present and in order.
    - Section 4 cites real files and real code, not vague impressions.
    - Section 5 ideas are grounded in the gaps from Section 4.
    - The document is written in the contributor's own words, not generated by an AI without understanding.

    A document that reads as if it was written without reading the codebase will be sent back.

### Explainer Video Gate (Mandatory, before Onboarding Document)

Before submitting the Onboarding Document, every new contributor must watch the FLN project explainer video (linked on Vibe) in full and pass the attention-check questions at the end. This is required *before* your first PR, not just before onboarding review — the video explains why the project is scoped the way it is (Math-only for now, no new features until Version 1 is clean, why the 93-level framework isn't a fixed lookup table) so you don't spend your first PR re-litigating decisions that are already settled.

Those settled decisions are written down, so read them before you disagree with them: the [Curriculum & Level Framework](#curriculum--level-framework) section above, the research it comes from in [`Research/fln_level_networks.md`](Research/fln_level_networks.md) and [`Research/fln_framework_evolution_log.md`](Research/fln_framework_evolution_log.md), and the open questions that research explicitly leaves unanswered in `Research/fln_level_networks.md` Part 4.

### Working Only From Predefined Issues

Until Version 1 is clean end-to-end, contributors should pick up work only from issues labeled [`intern-ready`](https://github.com/lucky-pluton/fln/issues?q=is%3Aissue+is%3Aopen+label%3Aintern-ready) — these are mechanical, well-scoped tasks (e.g. splitting a god-file, rolling out pagination) that don't require a judgment call about platform behavior. Issues without that label may touch pedagogical logic (the level framework, certification distance, diagnostic scoring) or unbuilt backend features, and need core-team review before and during the work — don't self-assign those without checking with a maintainer first. If you think something is missing from the issue list, raise it as a new issue; don't build it unscoped.


## Contribution Guidelines

This is an **open-source** project — contributions are welcome. Before contributing:

1. Check open issues or discuss the feature/fix you want to work on.
2. Fork the repo (or create a branch if you have write access).
3. Follow the branch naming and PR process below.
4. Keep PRs focused — one feature or one fix per PR.
5. Write clear commit messages describing *what* and *why*.

## Branching & PR Convention

All branches must follow this naming convention:

| Type | Branch Name Format | Example |
|------|--------------------|---------|
| Feature | `feat: <name of feature>` | `feat: auto question paper generation` |
| Fix | `fix: <name of fix>` | `fix: scanner upload crash on android` |

**Process:**
1. Create a branch using the convention above.
2. Make your changes and commit with clear messages.
3. Push the branch and **raise a Pull Request (PR)** against `main` (or the appropriate base branch).
4. PRs should reference the related issue (if any) and briefly describe the change.
5. At least one review/approval is required before merging (process may be refined as the team grows).

## License

This repository is open source. *(License file to be added — e.g., MIT/Apache 2.0. Update this section once finalized.)*
