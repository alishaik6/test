# Team Knowledge Base Pipeline — Sequential Claude Code Build Playbook

**Purpose:** Copy one prompt at a time into Claude Code after placing team source repositories and documents in a local workspace. Each prompt produces inspectable deliverables and acceptance evidence before proceeding. This is a build specification, **not** a claim that the pipeline has already been implemented.

## 0A. Mandatory execution contract — one stage, automated verification, human approval

**Two distinct phases:** (A) Build the utilities and builder skill using the sequential prompts below; (B) run the finished builder skill against team inputs. Do not confuse implementation-step completion with knowledge-processing-stage completion.

**Runtime state machine:** `PENDING -> RUNNING -> AUTO_VERIFY -> AWAITING_HUMAN_APPROVAL -> APPROVED`; on failed checks `AUTO_VERIFY -> FAILED`, with correction and re-verification before approval. A stage may advance only from `APPROVED`. Store transitions, timestamps, source fingerprints, artifact hashes, test evidence, and reviewer decisions in persistent local state (SQLite or equivalent). Never infer approval from silence or a previous conversation.

**Runtime contract for `/kb-builder next` (proposed interface, to be implemented):**
1. Confirm the previous stage is `APPROVED`, input fingerprints match, and no blocking discrepancies remain.
2. Execute exactly ONE eligible stage, writing only to staging and preserving source inputs.
3. Run deterministic automated verification for that stage; write a report with actual commands, PASS/FAIL/NOT RUN, evidence, provenance, and coverage.
4. On failure, stop, explain actionable fixes, and retain the last validated state. Do not advance.
5. On pass, STOP at `AWAITING_HUMAN_APPROVAL`; present a concise summary, changed artifacts, sample source-to-output evidence, uncertainties, and the exact local report path.
6. Wait for the developer to inspect the artifacts and explicitly approve this stage. An approval must be recorded with stage ID, artifact hashes, and source fingerprints; if any change occurs, approval expires.
7. Only a subsequent invocation of `next` may execute the next stage. No unattended advancement for MVP.

**Proposed skill commands:** `/kb-builder status`, `/kb-builder next`, `/kb-builder run <stage>`, `/kb-builder verify <stage>`, `/kb-builder approve <stage>`, `/kb-builder resume`. These are design targets, NOT existing Claude Code commands; implement and test command parsing/dispatch using the actual installed Claude Code skill mechanism. If the CLI cannot provide arguments as assumed, support equivalent natural-language invocations and document them.

**Independent validation:** automated verification is necessary but not sufficient; human approval is a separate explicit gate. For high-risk diagrams or business rules, inspect original source evidence and preserve discrepancies. Never self-approve or auto-publish.

**Stage boundaries:** document text/table/image extraction -> verify fidelity and image inventory -> image/diagram interpretation -> verify semantic relationships against originals -> curated knowledge generation -> verify citations and dependencies. Apply the same implementation/verification/approval pattern to code analysis, incremental processing, reconciliation, and publishing. Never merge extraction and interpretation into one unverified operation.

**No API requirement:** Python utilities perform deterministic extraction, hashing, diffing, persistence, and test execution. Claude Code terminal provides reasoning in bounded batches. Persist intermediate outputs and support safe resume; do not assume programmatic Anthropic API credentials or unattended model invocation.

### Copy-ready prompt — implement the stage-gated builder controller

```text
Implement the KB Builder Claude Code skill as an explicit, developer-controlled state machine, using only capabilities available in the local Claude Code terminal (do not assume Anthropic API credentials). Inspect existing code before editing. Support status, run-one-stage, verify-one-stage, explicit approve, and resume via supported skill invocation or documented natural-language equivalents. Every run executes exactly one stage, then runs its automated verification, writes a reproducible evidence report, and STOPS awaiting human approval. Block progression on FAIL/NOT RUN essential checks, unresolved critical discrepancies, or missing approval. Approval must bind to stage ID plus current artifact hashes and input fingerprints and expire on change. Save state/checkpoints durably; make retries idempotent and recoverable after interruption. Keep raw sources read-only, outputs staged, and publishing a separate explicit human-approved action. Add unit/integration tests for pass, fail, interruption, stale approval, changed input, duplicate invocation, and resume. Demonstrate a two-stage fixture run where stage two cannot execute until stage one has both verified and been explicitly approved. Do not mark success without executing the tests. Report exact files, commands, and evidence.
```

### Copy-ready verification prompt — stage controller

```text
Independently inspect the builder skill and its persisted state. Run tests proving: exactly one stage executes per invocation; a passing automated verification pauses for human approval; a failing or unrun essential test blocks progression; approvals are tied to stage and artifact/input fingerprints; changed inputs invalidate approvals; interruptions resume safely; the system never assumes API access; and publishing never happens without explicit approval. Record actual commands and evidence in work/verification/builder-controller.md, mark PASS/FAIL/NOT RUN, and recommend GO only if all critical conditions pass. Do not approve on the developer's behalf.
```

## 0. Agreed architecture and non-negotiable principles

- One isolated knowledge base per team; a designated developer runs the reusable pipeline locally.
- Developer clones the pipeline utilities and places source repositories plus Word/PDF/PowerPoint documents in an input folder. A separate team-specific Git repository publishes the validated knowledge base and one read-only Claude Code skill.
- Publish curated wiki, validated detailed extracted/enriched Markdown, relevant original images, indexes, dependency relationships, citations and freshness metadata. Do not publish bulk source code, temporary artifacts, credentials, or sensitive material without approval.
- Capture business rules, implementation behavior, APIs, events, data entities, UI flows, tests, service dependencies and documented cross-team dependencies. Preserve exact repo/path/symbol/commit and document/page/slide provenance.
- Initial full build; later developer-triggered incremental runs. Compare document hashes and normalized sections; compare code commits and changed symbols; detect diagram changes visually and semantically; regenerate affected artifacts through reverse dependencies. Handle deletions and renames.
- Discrepancies and low-confidence items are explicit review outputs. Human decisions trigger targeted reconciliation and revalidation. Unresolved critical claims are not published as verified.
- Single read-only Claude Code skill: explain systems and rules, retrieve source evidence, and perform blast-radius analysis. Adaptive detail, Mermaid diagrams when useful, confirmed vs potential vs unknown impacts, freshness/version notices.
- Never invent evidence, diagrams, source locations, dependencies, confidence scores or passing tests. Distinguish observed, inferred, disputed and unknown facts. A model's self-check is not independent verification.

## 1. Workspace preparation (human action)

Create a local folder, e.g. `team-kb-workspace/`:

```text
team-kb-workspace/
  inputs/
    repos/              # cloned team code repositories, unmodified by pipeline
    documents/          # .docx, .pdf, .pptx, images and related assets
  pipeline/             # reusable utilities repository / generated implementation
  work/                 # extracted content, cache, state, staging, logs; NOT published
  published-kb/         # separate Git repository, approved deliverables only
```

Start Claude Code at the workspace root. Review local security rules and ensure only authorized files are included. Git history may contain secrets; do not copy source code or secrets into published outputs. Back up the workspace before executing code-generation prompts. **Do not initialize or push remote repositories without explicit developer approval.**

### Common instructions to prepend to every prompt (or place in `CLAUDE.md`)

> You are implementing a local, evidence-grounded team knowledge-base pipeline. Inspect actual files before making assumptions. Never fabricate repository structure, source content, test outcomes, dependencies, extracted facts, or citations. Separate observations from proposals. Do not read outside the workspace, alter source repos, install packages, call external services, delete files, or push commits without asking first. Prefer deterministic local parsing and diffing; use an AI/vision model only where interpretation is needed. Maintain source provenance and persistent state. Make changes in small testable increments. After each task, list files changed, commands run, test results, limitations, and a precise stop/go checklist. If inputs or capabilities are missing, report the blocker and implement a safe interface or mock test rather than pretending the feature works. Never publish unverified or sensitive data.

### Execution discipline

Run each implementation prompt followed immediately by its matching verification prompt. After automated verification passes, inspect the artifacts and explicitly approve that step; never auto-advance. At each gate inspect artifacts, tests, and a sample source-to-output citation. If Claude Code lacks model vision or a required parser, keep a clearly marked adapter and stop that capability at `needs_review`; do not silently skip. Keep a change log and commit locally only after reviewing diffs.

---

## Prompt 1 — Inventory sources and write an implementation plan

**Copy into Claude Code:**

```text
Inspect this workspace read-only. Inventory input repositories (names, Git HEADs, languages, build systems, sizes) and documents (relative path, format, size, page/slide count when safely detectable, whether image-heavy/scanned). Do not dump file contents or secrets into logs. Identify relevant local extraction libraries, existing pipeline utilities and tests. Propose an MVP architecture consistent with the common instructions: local execution, persistent SQLite state, staged outputs, team-specific published repo, and read-only skill. Produce pipeline/docs/architecture.md, pipeline/docs/input-inventory.md and pipeline/docs/implementation-plan.md with a module map, schemas, risk register, security boundaries, performance budget and explicit assumptions. Distinguish available functionality from planned functionality. Do not implement the pipeline yet. End with concrete acceptance criteria and any questions that block implementation.
```

**Verify:** Every discovered source has a path and format; repo commits are captured; missing capabilities are identified. No secret contents exposed. **Gate:** approve plan before coding.

### Verification prompt 1 — inventory accuracy, missing files, assumptions, and secret-safe reporting

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 1. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Compare inventory counts with filesystem listings; sample at least two repositories and three documents when available; verify paths, Git SHAs, formats, and scanned/image-heavy classification.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-01.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-01.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 2. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 2 — Scaffold CLI, configuration, safe paths and tests

```text
Implement a minimal Python 3 CLI in pipeline/ with subcommands `inventory`, `extract`, `diff`, `build`, `validate`, `review`, `publish-preview`, and `status`. Add typed configuration and a sample config with input roots, work/state/staging/output paths, allowed extensions, resource limits and model adapter settings. Make all commands default to dry-run or non-destructive behavior until explicitly approved. Prevent path traversal, symlink escapes and writes into inputs/repos. Add structured logs without document contents or credentials, deterministic IDs, unit tests, README setup, and .gitignore for caches and secrets. Stub unfinished commands with honest errors. Run tests and show results.
```

**Verify:** CLI help, safe path tests, sample config and test suite. **Gate:** no input files modified.

### Verification prompt 2 — CLI behavior and filesystem safety

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 2. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Exercise help and each command in dry-run mode; test traversal, symlinks, missing config, and read-only input roots; confirm unfinished commands fail explicitly.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-02.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-02.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 3. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 3 — Persistent state and provenance schema

```text
Implement SQLite schema and migrations for runs, sources, source_versions, extraction_units, images, code_symbols, claims, relationships, artifacts, artifact_dependencies, review_items, review_decisions and publication_snapshots. Track document file SHA-256, normalized section hash, image byte/perceptual hash, repo commit SHA, parser/model/prompt version, status, timestamps, source location, and derived artifact IDs. Add indexes and foreign keys, transaction boundaries, idempotent upserts and backup/export. Document schema in pipeline/docs/state-schema.md. Write tests for repeat runs, migrations, crash rollback, deletions and stale dependency invalidation. Never overwrite the last successful snapshot before validation/publish approval.
```

**Verify:** migration round trip, rollback and unchanged-source idempotency tests.

### Verification prompt 3 — state schema, durability, and migrations

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 3. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Create a temporary state database, test insert/read/restart, migrations, duplicate IDs, failed-run rollback, and source-to-artifact provenance links.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-03.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-03.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 4. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 4 — Deterministic document extraction with precise citations

```text
Implement local extractors for DOCX, PPTX and text-based PDFs using available libraries, requesting installation approval if missing. Preserve heading/section structure, paragraphs, tables (row/column coordinates), slide notes, links, captions, page/slide numbers when reliably available, and image anchors. Export normalized units as JSON plus readable Markdown under work/extracted/. Keep stable source IDs and record parser warnings. For PDFs, note when page text is absent or reading order uncertain; do not invent OCR. For DOCX page numbers that cannot be reliably determined, cite section/paragraph rather than guessing a page. Build fixture tests for text, tables, ordering, embedded images and empty/scanned pages. Do not call AI in this step.
```

**Verify:** manually compare at least one unit from each available format with its original.

### Verification prompt 4 — document fidelity and source citations

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 4. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Sample DOCX/PDF/PPTX with headings, tables, footnotes, and pages/slides; compare extracted output manually against originals; check omitted or duplicated content and exact source coordinates.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-04.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-04.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 5. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 5 — Images, diagrams and scanned pages as first-class inputs

```text
Extend extraction to export original embedded images and page/slide renderings when required, preserving anchor, dimensions, file hash, source page/slide and nearby text. Identify image-heavy or scanned PDF pages. Implement a vision-adapter interface that accepts the image plus surrounding context and returns strict JSON: visible labels, nodes, directed edges, relationship types, uncertainty, evidence anchors and a descriptive summary. Use actual available model APIs only with explicit approval for sending source material; otherwise mark `vision_pending` and create review items. Never claim that a visual interpretation is verified solely because a model generated it. Add tests using synthetic diagrams and unsupported-image fallback.
```

**Verify:** originals retained; low-confidence or unsupported visuals visibly flagged, other extraction continues.

### Verification prompt 5 — image and scanned-content coverage

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 5. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Check that image inventory matches embedded assets/pages; verify image identity and provenance, OCR/vision fallback and low-confidence routing; never count a guessed interpretation as verified.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-05.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-05.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 6. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 6 — Code inventory and evidence-grounded technical extraction

```text
Implement read-only Git/code analysis. Record each input repo's HEAD and inspect tracked source files using language-appropriate parsers or tree-sitter where supported. Extract discoverable symbols, routes, API definitions, events, configuration keys, imports, test links and call/reference relationships with exact repo/path/line/symbol/commit provenance. For unsupported languages use explicit limited fallback and mark unverified relationships. Create compact implementation summaries and business-rule candidates using an AI adapter only after source snippets are selected and permitted. Do not publish bulk code. Add fixtures for renamed symbols, deleted files, route changes and feature flags. Report analysis coverage and blind spots.
```

**Verify:** sample claims can be traced to exact source lines and commit; no invented relationships.

### Verification prompt 6 — code extraction correctness

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 6. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Cross-check sampled functions, API endpoints, events, config flags and code citations against real files and Git HEAD; test unsupported languages and missing repos.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-06.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-06.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 7. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 7 — Structured knowledge model and reconciliation

```text
Create a normalized evidence/claim/relationship schema linking business concepts, rules, actors, services, APIs, events, data stores, UI, tests, documents, images and external teams. Each claim must have source IDs, evidence locations, extraction method, verified/inferred/disputed/unknown status and last-seen version. Reconcile document claims against code behavior without assuming code equals intended requirements. Generate work/reviews/discrepancies.md and structured JSON for conflicting or unsupported claims. Distinguish contradictory evidence from mere missing evidence. Preserve both sides with citations. Add deterministic tests for duplicate facts, conflicting thresholds and external dependencies.
```

**Verify:** conflicts show both sources and no automatic unsupported resolution.

### Verification prompt 7 — knowledge schema and contradiction handling

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 7. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Validate schema, trace every sampled claim to evidence, test conflicting doc/code facts and external-team uncertainty; reject invented relationships.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-07.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-07.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 8. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 8 — Initial full knowledge-base generation

```text
Implement a staged full build that produces under work/staging/: detailed validated source Markdown, curated domain wiki, business-rule catalogs, service summaries, cross-service workflows, relationship indexes, source catalog, freshness metadata, and Mermaid diagrams generated from structured relationships. Every factual assertion must carry traceable source IDs; unknowns must be labeled. Generate a machine-readable manifest mapping each published artifact to source units and upstream dependencies. Do not publish staging automatically. Provide example queries and show how wiki pages link to detailed evidence. Add schema, link, citation and Mermaid syntax tests.
```

**Verify:** sample business rule, diagram edge and service dependency trace back to evidence.

### Verification prompt 8 — full build completeness and reproducibility

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 8. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Run twice on unchanged inputs; compare output hashes excluding documented volatile metadata; audit coverage, citations, links, and unverified claims.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-08.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-08.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 9. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 9 — Document and image delta detection

```text
Implement incremental document diffing. First compare SHA-256 to skip unchanged files. For changed files re-extract locally, normalize text and compare stable section/table units so formatting-only changes avoid AI work. Use matching based on heading identity plus content similarity, not positional page number alone; classify added/modified/deleted/moved units. For images compare exact hashes, then perceptual similarity as a heuristic; a perceptual match must not override semantic differences. If an image changes, invoke approved vision interpretation and compare structured nodes/edges, or mark review required. Track source renames, changed parser versions, removed pages and ambiguous matches. Produce work/reports/document-delta.json with reasons and impacted unit IDs. Test a slide insertion, section move, one changed arrow, changed caption and formatting-only edit.
```

**Verify:** only changed semantic units are queued; uncertain image diffs go to review.

### Verification prompt 9 — document and image incremental changes

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 9. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Use fixture edits for text-only, formatting-only, changed image, moved section, deletion, and renamed file; verify changed units and unchanged cache reuse.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-09.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-09.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 10. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 10 — Git/code delta detection

```text
Implement incremental code diffing from the last successfully processed commit to the current HEAD using Git, including added/deleted/renamed files. Parse changed files and compare symbols, API contracts, events, config and tests; record affected business-rule candidates. Handle rewritten history, missing old commits, uncommitted modifications, parser changes and branch switches explicitly: require a safe full rescan or human decision rather than assuming the diff is complete. Produce work/reports/code-delta.json with exact evidence and impact candidates. Test commits that change only comments, a rule threshold, an API signature, a deleted symbol and a renamed file.
```

**Verify:** code change reports include base/head commits and clear fallbacks.

### Verification prompt 10 — Git incremental changes

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 10. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Test changed function, rename, deleted file, merge history and changed config; verify commit ranges and that untouched symbols are not reanalyzed.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-10.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-10.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 11. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 11 — Dependency-aware regeneration and deletions

```text
Build reverse dependency traversal from changed extraction units, code symbols, image relationships and claims to affected wiki pages, diagrams, indexes and skill retrieval metadata. Recompute only impacted artifacts and necessary aggregate indexes. Supply relevant unchanged context to AI prompts when interpreting changed units; do not interpret isolated snippets without dependencies. Track deletions so stale rules and edges are removed or flagged; preserve historical provenance. Cache results using source hash plus parser/model/prompt/schema version. Add tests showing one changed rule updates its wiki, workflow diagram and impact index but leaves unrelated pages byte-identical. Generate an explainable rebuild plan before execution.
```

**Verify:** targeted rebuild, unchanged artifacts retained, deleted facts not silently preserved.

### Verification prompt 11 — dependency invalidation and stale knowledge removal

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 11. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Change a shared rule and verify all dependent pages/diagrams are rebuilt; test cycles, deleted evidence, unaffected pages, and rollback.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-11.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-11.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 12. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 12 — Risk-based validation and diagram verification

```text
Implement validation gates: schema integrity, link/citation resolution, source coverage, duplicate claims, Mermaid render/syntax checks, semantic edge comparison against structured relationships, freshness metadata, sensitive-data screening, and contradiction detection. For changed diagrams compare old/new structured nodes and edges, validate against available code/contracts when possible, and flag uncertain interpretations for manual review. Syntax-valid Mermaid is not proof of semantic correctness. Classify severity as blocking, review-required or informational with documented rules. Continue processing independent sources, but never mark affected unresolved critical claims verified. Produce work/reports/validation.json and review-required.md. Add negative tests for fabricated citations, missing edges, stale source references and an unparseable image.
```

**Verify:** critical false claims block verified publication; unrelated valid content remains eligible.

### Verification prompt 12 — validation severity and diagram semantics

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 12. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Check Mermaid renders, then independently compare graph edges/nodes with structured evidence; test false-confidence and critical issue blocking.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-12.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-12.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 13. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 13 — Human review and targeted correction loop

```text
Implement structured review decisions in work/reviews/review-decisions.yaml with item ID, source evidence, reviewer, timestamp, disposition (accept/reject/correct/investigate), rationale and approved corrected claim. Validate decision schema and forbid unsupported decisions from silently overriding source evidence. On review rerun, apply decisions to the claim graph, regenerate only affected artifacts, revalidate, and preserve an audit trail. Invalidate prior decisions when underlying evidence materially changes. Keep unresolved items explicit. Build tests for correction, rejection, reopen on source change, and idempotent rerun. Provide a human-readable review guide.
```

**Verify:** correction updates the right wiki and diagram without full rebuild.

### Verification prompt 13 — review decisions and targeted corrections

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 13. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Test accept/reject/investigate decisions, audit trails, changed evidence reopening a decision, revalidation and no unrelated rewrites.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-13.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-13.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 14. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 14 — Secure publication and versioned snapshot

```text
Implement `publish-preview` to show exactly which staged files would be copied into published-kb/, with additions/deletions/changes and security/validation status. Publish only on explicit developer approval, only validated approved artifacts, and never raw source repos, secrets, logs, temporary caches or unresolved claims represented as verified. Include detailed extracted Markdown, approved original images, curated wiki, indexes, relationship data, manifest, freshness dates and source commit/document versions. Preserve previous good publication on failure using atomic staging/swap or rollback. Do not run git push automatically. Add tests for sensitive-file exclusion, rollback, partial validation and stale artifact deletion.
```

**Verify:** inspect exact Git diff and publication manifest before manual commit.

### Verification prompt 14 — publish boundary and sensitive-content exclusion

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 14. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Preview Git diff; scan outputs for secrets, raw code, temporary files and unapproved images; verify version metadata and no remote push.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-14.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-14.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 15. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 15 — One read-only Claude Code skill

```text
Create one Claude Code skill in published-kb/.claude/skills/team-knowledge/SKILL.md following the installed Claude Code skill conventions verified from available local documentation; if conventions are unavailable, mark that assumption and create a clearly documented template. The skill must answer business-rule and system-behavior questions, retrieve detailed extracted Markdown through indexes, explain current implementation, and provide blast-radius analysis across documented internal/external dependencies. Use adaptive depth: concise for simple questions; Mermaid sequence/flow diagrams, examples or decision tables for complex questions. Cite exact evidence, show knowledge snapshot date and relevant source commit/document version, distinguish documented vs implemented behavior, confirmed vs potential vs unknown impacts, and call out discrepancies. No writes, corrections or autonomous code modifications. Add realistic evaluation questions and expected evidence criteria; never assert evaluation passed unless executed.
```

**Verify:** skill is read-only; sample impact answer includes downstream services, UI, tests, external-team verification needs and freshness warning.

### Verification prompt 15 — read-only skill answers and blast radius

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 15. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Ask grounded and adversarial questions; verify citations, freshness, uncertainty, external dependencies, no write operations and no unsupported conclusions.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-15.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-15.md`. Fix failures, rerun this verification prompt, and only then proceed to Step 16. A model's self-review is not independent proof: inspect critical evidence yourself.

## Prompt 16 — End-to-end test, performance baseline and release checklist

```text
Create small synthetic fixture repositories and documents containing text, tables, diagrams, contradictions and external dependencies. Run the full pipeline, capture artifacts and validation results, then simulate incremental changes: edit a rule, add a slide, change a diagram edge, rename an API, delete a document, resolve a discrepancy. Verify correct delta scope, accurate citations, diagram semantics, auditability, rollback and safe publication. Measure extraction time, AI calls/tokens where available, changed units, rebuilt artifacts, and total run time. Write pipeline/docs/e2e-results.md, performance-baseline.md, operator-runbook.md, limitations.md and release-checklist.md. Clearly separate tests actually run from tests requiring user credentials or model access. Do not claim production readiness if required gates remain untested.
```

**Verify:** independent sample checks and documented blockers; then manually commit the validated team KB to its separate repository.

---

### Verification prompt 16 — end-to-end regressions and performance

**Run in a fresh Claude Code turn after implementation, before proceeding:**

```text
Act as a skeptical QA reviewer for Step 16. Do not implement the next step. Read its stated requirements and acceptance criteria, inspect the actual files and changes, and execute relevant deterministic tests using available tools. Specifically: Run clean build and incremental build with representative fixtures; test interrupted runs, rollback, deleted sources, diagram change, review loop and runtime metrics.

Include positive, negative, boundary, and regression cases where applicable. Verify at least one source-to-output example against the original evidence; do not accept model-generated explanations as proof. Record exact commands, actual results, expected results, file paths, and reproducible failures. Do not claim tests passed if they were not executed. Do not modify source inputs, publish, or push. If a test cannot run, mark it NOT RUN with the reason; do not substitute a hypothetical pass.

Write work/verification/step-16.md with: requirement-to-test matrix, PASS/FAIL/NOT RUN for each check, severity of defects, risks to later steps, evidence and a clear GO/NO-GO recommendation. If you discover defects, propose minimal fixes but ask before making changes. Treat any critical failure, missing essential evidence, or unexecuted essential test as NO-GO.
```

**Decision gate:** Review `work/verification/step-16.md`. Fix failures, rerun this verification prompt, and only then proceed to Step release. A model's self-review is not independent proof: inspect critical evidence yourself.

## Operator workflow after initial build

1. Update documents and input repositories in the original workspace; keep a known previous successful snapshot.
2. Run inventory/diff; inspect proposed document, image and Git deltas.
3. Run incremental build into staging; inspect impacted artifact list.
4. Run validation; review `discrepancies.md` and `review-required.md`.
5. Enter structured review decisions, rerun targeted reconciliation and validation.
6. Review publication diff, sensitive-data exclusions, freshness/version markers and unresolved warnings.
7. Explicitly approve publication; commit the published team KB and read-only skill to the separate Git repo. Consumers clone that repo and ask questions using the skill.

## Definition of done

- Repeat runs with no source changes do not trigger unnecessary AI interpretation or artifact rewrites.
- Changed sections, code symbols and diagram relationships update all and only relevant dependent artifacts, with safe conservative fallback on uncertainty.
- Every material claim/edge is source-traceable; missing or disputed evidence is labeled.
- Critical unresolved items are never published as verified facts; independent valid knowledge may proceed under policy.
- Original source material remains private by default; approved detailed extracted Markdown and selected images provide depth without bulk code duplication.
- One skill supports explanation, rules, workflows and blast radius with current snapshot provenance.
- Last successful publication is recoverable after any interrupted build.

## Important caveats

- PDF page rendering/OCR, PowerPoint shape interpretation and model vision availability vary by local environment. Treat these as capability checks, not guaranteed features.
- Image hashing detects visual changes; it cannot by itself prove semantic equivalence. Diagram semantic validation needs evidence and sometimes human judgment.
- The full pipeline is substantial. The prompts are staged so each result can be inspected and corrected before continuing; do not paste all prompts at once.

## 0B. File-by-file approval and expanded input formats (mandatory)

- Supported inputs: local Git repositories; .pdf (digital and scanned); .docx; .pptx; .xlsx/.xlsm (read-only, no macro execution); .csv/.tsv; and standalone/embedded images and diagrams. Inventory all inputs and flag unsupported/corrupt/encrypted formats rather than silently skipping them. For legacy .doc/.ppt/.xls, use an explicitly tested safe conversion path or report unsupported.
- Use format-specific Python utilities: e.g. pypdf/pdfplumber and scanned-page rendering, python-docx, python-pptx, openpyxl read-only (preserve formulas, cached values where available, hidden sheets, named ranges, merged cells, comments, provenance), CSV dialect/encoding detection, image extraction and hashing, and Git-aware source analysis. Libraries are examples; verify installed capabilities and fidelity. Do not execute document macros or untrusted repository code.
- Preserve provenance: repository/commit/path/symbol; PDF page/region; Word section/table; PowerPoint slide/shape; spreadsheet workbook/sheet/cell range/formula; CSV filename/row/column; embedded image parent and position.
- **Default runtime mode: interactive per-file.** Select a deterministic next file (initially prioritize representative examples of each supported format). Process ONE file end-to-end through extraction, extraction verification, image interpretation, interpretation verification, knowledge generation, and knowledge verification. Show extracted content, knowledge, citations, diagrams, discrepancies and test evidence. STOP for explicit file-level human approval. Do not select the next file without approval.
- Allow revise/retry on the current file; re-run only invalidated operations, preserve previous verified outputs, and invalidate approvals on input/output changes. Track per-file hashes, status, checkpoints, reports, and reviewer decisions. Treat a Git repository as a source collection: process bounded file/symbol units, with repository-level reconciliation.
- After all individual files are approved, run a separately verified cross-file reconciliation stage for relationships, contradictions, cross-repo impacts, and source-grounded confidence. File approval does not mean cross-file claims are validated.
- **Automatic mode is opt-in only** after representative coverage, including scanned PDFs, diagram-heavy slides, formula-heavy spreadsheets, malformed CSVs, and negative tests. Automatic mode must retain verification, checkpoints, exception queues and human escalation for uncertain/high-risk items. No automatic publishing without explicit approval.

### Copy-ready implementation prompt — format support and file-level gates

```text
Extend the KB Builder skill and its Python utilities to inventory Git repositories, PDF/scanned PDF, DOCX, PPTX, XLSX/XLSM, CSV/TSV, and standalone/embedded images. Implement tested format-specific extraction with exact provenance, unsupported-file reporting, and incremental fingerprints. Implement interactive per-file execution as the default: fully process exactly one selected file through extraction, interpretation, knowledge generation, and separate automated verification gates; present all evidence and STOP awaiting explicit human approval. Permit targeted correction and retry without losing prior verified checkpoints. Never process the next file until approval bound to its current hashes. Once all files are approved, run cross-file reconciliation as its own verified, approval-gated stage. Make automatic mode an explicit opt-in, never the default. Write tests for each format, damaged/unsupported files, formula/cell provenance, scanned pages, embedded diagrams, stale approvals, and stop/resume behavior. Report commands, evidence, and outstanding limitations.
```

### Copy-ready verification prompt — formats and file-level gates

```text
Independently run representative fixtures for each supported input format and negative cases. Confirm every file is inventoried or explicitly reported unsupported; check source-level fidelity and precise citations; verify only one file is processed per interactive invocation; test failures, retries, changed hashes, approval gates, cross-file reconciliation, and opt-in automatic mode. Examine original documents and output samples, not just logs. Report PASS/FAIL/NOT RUN and actual evidence. Block GO if any essential file type is untested, data is silently omitted, or progression occurs without human approval.
```
