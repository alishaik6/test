# Lean Knowledge Base Builder — POC Playbook

This is a new, independent POC plan. Do not modify the existing IMPLEMENTATION_PLAYBOOK.md. Use Claude Code terminal, not a separate model API. Goal: demonstrate an end-to-end result quickly; avoid elaborate orchestration and test frameworks.

## Operating principles
- One copy/paste master prompt per stage; run stages in order and review outputs between stages.
- Scan a user-specified project folder recursively, including Git repositories, and process supported files one at a time.
- Ignore .git, .venv, node_modules, build/output directories, and binaries or secrets. Report unsupported files and failures rather than silently dropping them.
- Keep extracted evidence separate from model interpretations; retain source paths and page/slide/sheet/line references where feasible.
- Install dependencies once into a project-local .venv via requirements.txt and a simple setup.sh (provide Windows instructions if relevant).
- POC quality check: demonstrate one representative file, check that outputs exist and are readable, and report limitations. No exhaustive testing, approval controller, databases, or autonomous background processing.

## Stage 1 — Discover and extract

**Copy this single prompt into Claude Code:**

> Read this POC playbook. Implement Stage 1 only as a reusable Claude Code skill at .claude/skills/kb-builder/SKILL.md with lightweight Python scripts, requirements.txt, and setup.sh. Accept a source directory parameter; recursively inventory Git/source files, PDF, DOCX, PPTX, XLSX, CSV, and common images. Process one file at a time and generate extracted Markdown plus separate image artifacts under a dedicated output folder. Preserve source location metadata. Use PyMuPDF, python-docx, python-pptx, openpyxl, and Python csv where suitable; preserve Excel formulas and sheet/cell references. For source code, capture relevant file text with paths and line numbers, without executing it. Extract embedded images where practical; if scanned content or diagrams cannot be reliably extracted, mark it for Stage 2. Skip generated/dependency directories and avoid exposing secrets. Keep code minimal and readable. Install dependencies into .venv only after showing the commands. Run one small sample, show the output paths and any errors, then stop. Do not implement Stages 2 or 3.

**Expected output:** inventory, per-file Markdown, images, simple error report.

## Stage 2 — Interpret images and enrich Markdown

**Copy this single prompt into Claude Code:**

> Read this POC playbook and the Stage 1 artifacts. Implement and run Stage 2 only. Using Claude Code's available image-reading capability, inspect each extracted image or rendered diagram/page in manageable batches. Explain architecture, labels, arrows, business processes, charts, and relationships that are actually visible. Append clearly marked interpretation sections with image paths and source page/slide references to enriched Markdown, without overwriting raw extraction. Flag illegible, ambiguous, or unavailable visuals instead of guessing. Show one enriched file and the remaining limitations, then stop. Do not implement Stage 3 or assume separate Claude API access.

**Expected output:** enriched Markdown linked to source images, with uncertainty marked.

## Stage 3 — Curate hybrid-discovery LLM Wiki

**Copy this single prompt into Claude Code:**

> Read this POC playbook and the enriched Markdown from Stages 1–2. Implement and run Stage 3 only. Synthesize a navigable Markdown LLM Wiki. Always evaluate these fixed categories: (1) domain overview and capabilities, (2) architecture and components, (3) business rules and validations, (4) workflows, (5) APIs and integrations, (6) data models and flows, (7) dependencies and change impacts, (8) glossary. Populate categories only where evidence exists; mark gaps instead of inventing content. Also automatically discover additional domains, subdomains, and relationships from the sources and create useful pages for them. Keep the eight-category checklist configurable in a simple Markdown or YAML file. Cite source file locations for important claims, note contradictions, and generate wiki/index.md linking all pages. Use bounded batches when the corpus is too large for one context. Show the wiki index and a sample domain page, then stop.

**Expected output:** wiki/index.md, grounded domain pages, gaps and contradictions.

## POC completion
Demonstrate at least one end-to-end example covering extraction, visual interpretation where applicable, and a source-grounded wiki page. Review the results manually. Defer automated mode, change detection, sophisticated checkpoints, production security audits, vector/graph databases, and a separate consumer skill until the POC proves useful.
