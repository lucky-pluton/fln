# Agent & AI Architecture

How AI is used in the FLN platform: what calls a model, what a model is allowed to decide, what happens when the model is unavailable, and what is explicitly **not** an agent.

Companion documents: [ARCHITECTURE.md](ARCHITECTURE.md) (system shape), [PIPELINES.md](PIPELINES.md) (end-to-end flows), [SRS.md](SRS.md) (requirements), [CLAUDE.md](CLAUDE.md) (working rules).

> **Status banner.** There is **no agent framework in this repository** — no LangChain/LangGraph/AutoGen/CrewAI dependency, no planner, no tool registry, no autonomous loop, no multi-turn memory. "Agent" behaviour in this codebase means a small number of **single-shot, schema-validated LLM calls wrapped in deterministic TypeScript**, plus statistical inference that uses no LLM at all. Most model calls sit behind a **deterministic fallback** (the two exceptions are OCR and clustering, both of which degrade by skipping or erroring — see the table). Nothing below should be read as describing a future design unless it is marked *(planned)*.

---

## 1. The AI surface, in one table

| Capability | Module | Model | Deterministic fallback | Live route |
|---|---|---|---|---|
| Answer matching / grading | `backend/src/answerMatching.ts` | none | — (it *is* the fallback) | `students.ts` (diagnostic scoring); its `normalizeAnswer`/`asNumber` are reused by `errorClassification.ts` |
| Wrong-answer classification | `backend/src/errorClassification.ts` | none | — (pattern-based) | `students.ts` |
| Misconception fingerprinting | `backend/src/misconceptionFingerprint.ts` | none | — (statistical) | `students.ts`, `misconceptions.ts` |
| Prerequisite resolution | `backend/src/competencyPrerequisites.ts` | none | — (graph walk) | `students.ts`, `evaluation.ts`, boot validation |
| Question content per concept | `backend/src/utils/conceptQuestionGenerator.ts` | none | — (static builders; 23 of 93 concepts authored) | `QuestionService` |
| Worksheet evaluation + narrative + next level | `backend/src/gemini.ts` → `evaluateAIWorksheet()` | Gemini | yes (deterministic score + level arithmetic) | `POST /api/evaluation/submit` — the only live Gemini call on the **assessment-submission** path |
| Diagnostic evaluation | `backend/src/gemini.ts` → `evaluateAIDiagnostic()` | Gemini | yes | imported by `students.ts` and `index.ts` but **never called** |
| Misconception clustering (LLM-assisted) | `backend/src/geminiClusterMatcher.ts` → `classifyFingerprintWithGemini()` | Gemini | no — skipped if unavailable | via `studentArchetypeService.ts` (archetype assignment, called from `evaluation.ts`, `students.ts`, `misconceptions.ts`) |
| AI question / worksheet generation | `backend/src/gemini.ts` → `generateAIDiagnostic()`, `generateAIPersonalizedWorksheet()` | Gemini | yes (`generateClassSpecificDiagnostic()`, deterministic) | imported by `index.ts` but **never called** — all question content is currently deterministic (see [PIPELINES.md](PIPELINES.md) pipeline A) |
| Scanned-answer OCR | `backend/src/routes/evaluation.ts` → `runCloudOcrOnImage()` | Ollama Cloud **Gemma 4** (vision) | no — returns an error the UI surfaces | `POST /api/icr/evaluate-cloud`, `POST /api/icr/evaluate-bulk` |
| PDF → image rasterization | `ai-services/scripts/pdf_rasterize.py` | none (MuPDF) | — | both ICR routes |

**The most important line in that table is the second half of it.** The majority of the product's intelligence is *not* an LLM: answer comparison, error classification, misconception fingerprinting, prerequisite traversal, and question content are all deterministic and testable. The LLM is used where tolerance to messy input and fluent pedagogy actually pays — narrative reporting, level recommendation, clustering, and vision. This split is a deliberate design property, not an accident of implementation.

---

## 2. Model configuration

- **Client:** `@google/genai` (`GoogleGenAI`), constructed lazily in `backend/src/gemini.ts` and memoized.
- **Model list** (`GEMINI_MODELS`): `gemini-flash-latest` (default) then `gemini-3.5-flash`. The list carries a dated comment recording *why* earlier ids were dropped (`gemini-2.5-flash` / `-flash-lite` 404 for new users; `gemini-2.0-flash` 429s on free tier). Treat the list as maintained, not as a constant.
- **Retry policy** (`generateContentWithRetry()`): for each model in order, up to 3 attempts with exponential backoff, extra multiplier on `429`/`503`/`RESOURCE_EXHAUSTED`/`UNAVAILABLE`, then move to the next model. A missing key is **not** retried — it fails fast with `NO_API_KEY` so the caller takes its fallback.
- **Key:** `GEMINI_API_KEY` from the environment. Never committed, never logged, never sent to the browser.

### Env vars

| Var | Required | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | for real AI calls | Gemini. Absent ⇒ every AI path degrades to its deterministic fallback |
| `OLLAMA_API_KEY` | for OCR | Ollama Cloud key for the Gemma 4 vision model. Also read as `ICR_CLOUD_API_KEY_OLLAMA_GEMMA4` |
| `OLLAMA_API_URL` / `OLLAMA_MODEL` | no | Override endpoint / model (default `https://ollama.com/api/chat`, `gemma4:cloud`) |
| `AI_SERVICES_DIR`, `PYTHON_BIN` | for the scan path | Where to find `pdf_rasterize.py` and which interpreter to run it with |

---

## 3. What a model is allowed to decide

This is the part to read before adding an AI path.

1. **Narrative and recommendation only, never arithmetic on the record.** Scores, sublevel, level arithmetic, and the 59-level cap are computed in TypeScript. The model is asked for a narrative and a recommendation, and the recommendation is then clamped in code.
2. **Identifiers stay out of the prompt; the display name does not.** Aadhaar and Birth-Certificate numbers are tokenized by `backend/src/modules/vault/` and never reach a model, and neither does any token or vault key. What *is* sent is the child's **display name**, their level, and the question/answer pairs — a name is still personal data, so treat the prompt payload as personal data and do not widen it (no DOB, no school, no address, no raw identifier) without a reason.
3. **Structured output is validated, not trusted.** `geminiClusterMatcher.ts` parses a JSON response, validates its shape, and discards anything malformed. A malformed response is a non-event, not a crash.
4. **Teacher override always exists.** `PATCH /api/evaluation/:reportId/override` lets a Teacher correct a per-question result, and the correction is applied to the report and the student's standing. The model is never the last word.
5. **No model may extend the curriculum.** Concept IDs, levels, strands, and prerequisite edges come from `curriculumMap.ts` and `competencyPrerequisites.ts`. A model may recommend where to go next; it may not invent a level or an edge.

---

## 4. Deterministic components worth knowing

### 4.1 Answer matching — and there are three of them

This is the most consequential comparison in the product: its result feeds `recommendedLevel`, so a false negative does not just lose a mark, it places the child lower and hands them the wrong worksheets. It is also the least unified part of the codebase. Three separate implementations exist:

| Implementation | Rule | Used by |
|---|---|---|
| `answerMatching.ts::answersMatch()` | normalise (lowercase, collapse whitespace, tidy commas, drop a trailing full stop) → exact match → numeric equality when **both** sides parse as a plain number (`07` == `7`, `2.0` == `2`) → element-wise compare for comma-joined multi-blank rows. **No fuzzy matching.** | `students.ts` diagnostic scoring; `errorClassification.ts` reuses its helpers |
| `gemini.ts::answersMatch()` (module-private) | the same normalisation, plus a small **length-scaled Levenshtein tolerance for non-numeric answers only** — numeric answers are deliberately excluded, because a 1-character edit between two numbers (`5`/`6`, `12`/`13`) is a different number, not an OCR near-miss | `evaluateAIWorksheet()` — score, error tally, level derivation |
| inline in `routes/evaluation.ts` | plain `trim().toLowerCase() ===` | the per-question `questionResults` rows and the sublevel derivation on worksheet submission |

**Known inconsistency:** a child's handwritten answer can therefore be scored *correct* by the Gemini-path comparison while the per-question row persisted for the teacher records it *incorrect*. The teacher's override endpoint exists partly to absorb this, but the three-way split is a real duplication and is worth consolidating before it is load-bearing for placement accuracy.

### 4.2 Error classification (`errorClassification.ts`)

Assigns an `errorType` from the answer pair alone: unanswered, digit reversal, decimal place shift, off-by-one. Anything that does not match a known pattern stays `unclassified` **rather than being guessed at**. Added because `errorType` was previously always `unclassified` (FLN #459). Has a focused test: `npm run test:error-classification`.

### 4.3 Misconception fingerprinting (`misconceptionFingerprint.ts`)

The largest single analysis module, and it uses **no LLM**. It derives per-skill fingerprints from a child's response history against the level/skill graph. The LLM-assisted part is only the *clustering* step in `geminiClusterMatcher.ts`, which proposes groupings that are then validated.

### 4.4 Prerequisites (`competencyPrerequisites.ts`)

Pure graph logic over a generated edge table. Validated at boot from `index.ts` — unknown concept ids and cycles are logged and treated as a startup-class defect. See [SRS.md](SRS.md) §1.8.3 for the typing rules: only `prereq` is load-bearing, and how a concept's *multiple* `prereq` parents combine is an **open decision** — the data is a plain list with no AND/OR marker, so do not read it as conjunctive.

### 4.5 Question content (`conceptQuestionGenerator.ts`)

`generateQuestionsByConcept(conceptId, subLevel)` emits **four questions per concept**, keyed by the concept id, with `topic` (strand), `subtopic` (the sub-level label — `Mastery` / `Easier` / `Remedial`), `difficulty`, `source_level` and `conceptId` stamped on each `Question`. Explicit builders exist for **23 of the 93 concepts** (`S1.1`–`S1.7`, `S2.1`, `S2.4`, `S2.10`, `S3.1`, `S3.3`, `S3.6`, `S4.1`, `S4.6`, `S4.7`, `S5.4`, `S5.5`, `S6.6`, `S6.7`, `S7.14`, `S7.15`, `S7.18`); every other concept hits a `default:` branch that emits a **placeholder** (`[Concept Sx.y] Level N Practice Question #k`, answer `k*10`). So the deterministic content path is real for a quarter of the curriculum and stubbed for the rest — plan content work accordingly, and do not assume a level has authored questions just because generation succeeds.

This is the current content mechanism — **not** the question-template system, and not the Gemini generation helpers. `backend/src/routes/questionTemplates.ts` is a real intent-based authoring surface (full CRUD, CSV import, a param catalogue, a level map, and an `intent` classifier backfilled by `backend/src/migrations/questionTemplateIntentV1.ts`), but the generation pipeline does not consume it. That is the open gap tracked as **issue #486**, "Generation pipeline must consume questionTemplates": `getQuestionTemplatesByVariantKey()` has exactly two callers, both inside that same route file. Templates can be authored today and no child will see them until something calls it.

---

## 5. Vision & OCR

- **Rasterization:** the backend shells out with `execFileSync` to `ai-services/scripts/pdf_rasterize.py` (MuPDF) — page 1 only, or all pages for a multi-page PDF — and base64-encodes the JPEGs. **This is the only Python script the current backend invokes.** It is synchronous and blocking; a large PDF holds the event loop for the duration.
- **OCR:** `ollama-gemma4` is the single configurable provider (`POST /api/icr/cloud-config`, admin/superadmin only; key stored in config, not in the prompt path). One vision call per page, with a cumulative base64 size cap. PDFs must be rasterized locally first because the vision endpoint accepts image MIME types only.
- **Dead branches:** `runCloudOcrOnImage()` still contains Google Cloud Vision, AWS, Azure, MiniMax and OCR.space branches. They are unreachable — the config route rejects any provider other than `ollama-gemma4` with a 400. Read them as historical residue, not as supported providers.
- **Legacy Python pipeline:** `run_pipeline.py`, `personalized_evaluation_pipeline.py`, `scripts/0_auto_classify_questions.py` … `3_generate_report.py`, `prompts/`, `syllabus/`, `questions/`, and `personalized_evaluation/` all operate on a **legacy `class_N` × `phrase_N` data model** unrelated to the current student model. The backend's last write/read bridge to them was deleted as dead code in commit `9c919895`; the replacement is tracked as FLN #458. `ai-services/PIPELINE.md` documents that pipeline accurately *as a legacy pipeline* — it is not a description of the running system.

---

## 6. Guardrails and known limits

- **No key, no AI:** with `GEMINI_API_KEY` unset the server still boots and serves every route; AI-backed responses are visibly lower fidelity (no narrative) but scores, levels, and analytics are unaffected. This is the intended degradation.
- **Latency is unbounded on the LLM paths** apart from the retry budget. Assessment submission waits on Gemini; there is no queue and no timeout contract.
- **Cost is unmetered** per assessment; there is no per-class or per-child cap.
- **Single provider:** Gemini only. The SRS lists multi-LLM provider switching as out of scope, so there is no abstraction seam to add one through today.
- **Prompt text is inline in TypeScript**, except for the legacy `ai-services/prompts/*.txt`, which nothing reads. There is no prompt registry, versioning, or review step for prompt changes.

---

## 7. Adding an AI capability — the rules

1. Put it in `backend/src/**` (or a deeper `backend/src/modules/<domain>/`), never in a React component. Business logic on the server is the architecture's hard rule, not a preference.
2. Route every model call through `generateContentWithRetry()`; do not construct a client per call site.
3. Ship a **deterministic fallback** in the same change, and make the degraded path observable.
4. Compute scores, levels, locks, and certification **in code**. Ask the model for prose, classification, or a suggestion; clamp what comes back.
5. Validate structured output against a shape and reject on mismatch.
6. Keep identifiers out of the payload. The child's display name is already sent today (§3.2) — do not widen beyond name, level, and question/answer pairs.
7. Add a test if the logic is deterministic. Narrow per-package test commands already exist (e.g. `npm run test:answer-matching`, `npm run test:error-classification` from `backend/`).
8. If the capability changes what a child is told, it must be reachable by, and overridable by, a Teacher.

*(Planned, not implemented: a prompt registry, per-path timeouts, cost metering, and a real replacement for the legacy Python pipeline — FLN #458.)*
