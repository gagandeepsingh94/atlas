---
name: atlas
description: Codebase intelligence for AI agents — generates architecture docs (.atlas/codebase-index.json + .atlas/codebase-overview.md), answers questions about code from pre-built docs, creates architecture diagrams (draw.io, Excalidraw, Mermaid), performs ML deep-dives, maps cross-repo ecosystems, and generates domain model references (docs/domain-models.md) with Mermaid diagrams. Use when asked to document a codebase, explore service architecture, answer "how does X work?", generate architecture diagrams, analyse ML pipelines, map multi-repo dependencies, or generate domain model docs.
allowed-tools: Read Glob Grep Bash Write Edit
---

# Atlas — Codebase Intelligence

Atlas gives AI agents persistent, structured understanding of codebases. Build docs once; answer questions cheaply forever. Uses git-aware incremental refresh to keep docs current without full re-exploration.

## Outputs produced

- `.atlas/codebase-index.json` — machine-readable structured map (always written)
- `.atlas/codebase-overview.md` — human-readable 11-section architecture reference
- `.atlas/architecture.drawio` + `.atlas/architecture-diagram.md` — diagrams (optional)
- `.atlas/ml-overview.md` — ML deep-dive (when ML detected or requested)
- `.atlas/ecosystem-overview.md` + diagram files — cross-repo analysis
- `docs/domain-models.md` — comprehensive domain model reference with Mermaid diagrams

---

## Step 0 — Detect Intent

Map the user's request to one of the six operations:

| If the user wants... | Run Operation |
|---|---|
| Generate/refresh codebase docs, explore architecture, document a service, index codebase | **A: Codebase Overview** |
| Ask questions about code, "how does X work?", explain a flow or component | **B: Ask Atlas** |
| Create an architecture diagram, draw.io file, Mermaid diagram, visualise system | **C: Architecture Diagram** |
| ML pipeline docs, model inventory, training/serving overview, ML-specific questions | **D: ML Overview** |
| Cross-repo dependency map, ecosystem analysis, blast radius, multi-service map | **E: Ecosystem Overview** |
| Domain model docs, service entity reference, state machines, data flows for a service | **F: Domain Models** |

---

## Operation A: Codebase Overview

Orchestrates a full or targeted exploration of the current codebase, producing `.atlas/codebase-overview.md` and `.atlas/codebase-index.json`.

### A.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas codebase-overview — Generate or refresh a comprehensive codebase overview

USAGE
  [options]

OPTIONS
  --output <path>             Override output path (default: .atlas/codebase-overview.md)
  --focus <area>              Give extra depth to one area, e.g. --focus "async pipeline"
  --fresh                     Skip git optimisation — full re-index from scratch
  --code-only                 Derive everything from codebase; skip reading existing .atlas/
  --with-diagram              Also generate .atlas/architecture.drawio + Mermaid preview
  --with-diagram=excalidraw   Also generate .atlas/architecture.excalidraw + Mermaid preview
  --with-diagram=both         Generate draw.io + Excalidraw + Mermaid preview
  --with-ml                   Also run ML Overview (forced, even if no ML detected)
  --no-ml                     Skip ML Overview even if ML artifacts are detected
```

### A.1 — Detect existing files

Use the Glob tool to check for:
- `.atlas/codebase-index.json`
- `.atlas/codebase-overview.md` (or `--output` path)

If `--fresh` was passed: skip Step A.2 entirely.

### A.2 — Run Procedure: Detect Git Changes

> Skip if `--fresh` was passed or if neither index nor overview doc exists.

Follow **Procedure 1: Detect Git Changes** in this document.

| Result | Action |
|--------|--------|
| `mode: "current"` | Update the `Last updated` date only. Print: `"No commits since <date> — doc is current. Refreshing date only."` Done. |
| `mode: "targeted"` | Proceed to A.3 in targeted mode with `changed_files` and `affected_sections`. |
| `mode: "full"` | Proceed to A.3 in full mode. |

### A.3 — Run Procedure: Index Codebase

Follow **Procedure 2: Index Codebase** in this document.

- **Full mode**: Run Mode A (index all files).
- **Targeted mode**: Run Mode B with `--files <changed_files>`.
- If `--code-only`: do not read any existing files in `.atlas/` during indexing.

### A.4 — Run Procedure: Write Overview Doc

Follow **Procedure 3: Write Overview Doc** in this document.

- **Full mode**: generate all required sections.
- **Targeted mode**: pass `--sections <affected_sections>`; carry all others verbatim.
- Pass through `--output`, `--focus`, `--fresh`.

### A.5 — Run Procedure: Generate Diagram *(only if `--with-diagram` passed)*

Follow **Procedure 4: Generate Diagram** in this document.

Map flags:
- `--with-diagram` → `--format drawio`
- `--with-diagram=excalidraw` → `--format excalidraw`
- `--with-diagram=both` → `--format both`

After diagram is written, add a cross-reference to `.atlas/architecture-diagram.md` in the companion docs block at the top of `.atlas/codebase-overview.md` using the Edit tool.

### A.6 — ML artefact detection

Scan the repo for ML signals using Glob and Grep:
- Model files: `*.pt`, `*.pth`, `*.ckpt`, `*.pb`, `*.h5`, `*.onnx`, `*.pkl`, `*.safetensors`
- ML imports: `torch`, `tensorflow`, `sklearn`, `xgboost`, `transformers`, `jax`, `lightgbm`
- Training scripts: `train.py`, `fit.py`, files with `trainer` in the name
- Experiment tracking: `mlflow`, `wandb`, `comet_ml`, `neptune`
- Notebooks: `*.ipynb`

| Condition | Action |
|-----------|--------|
| `--no-ml` passed | Skip. |
| ML artifacts found OR `--with-ml` passed | Run **Operation D: ML Overview**. Pass through `--output`, `--focus`, `--fresh`. |
| No ML artifacts and no `--with-ml` | Skip. |

### A Notes

- Always log the mode used (targeted / full / date-only) so the user knows what happened.
- `.atlas/codebase-index.json` is always written, even on a targeted refresh.
- The `<!-- Last updated: YYYY-MM-DD -->` line must always be the first line of the overview doc.
- If `.atlas/` does not exist, create it with the Write tool.

---

## Operation B: Ask Atlas

Loads pre-built Atlas docs into context and answers questions from those docs only — no source file traversal, minimal token spend.

### B.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas ask — Answer questions about this codebase using pre-built Atlas docs

USAGE
  [question] [options]

OPTIONS
  --files <path1,path2,...>   Include additional source files in context
  --fresh                     Regenerate all docs before answering
  --no-ml                     Skip ml-overview.md even if present
```

### B.1 — Parse inputs

Extract:
- **Inline question**: everything that is not a flag or flag argument
- **`--files`**: comma-separated paths; read each with the Read tool at Step B.5
- **`--fresh`**: if present, regenerate all docs before entering chat mode
- **`--no-ml`**: if present, skip `.atlas/ml-overview.md`

### B.2 — Check for existing docs

Use the Read tool to check only these exact paths (do NOT use Glob or Grep on source files):
1. `.atlas/codebase-overview.md` ← primary (required)
2. `.atlas/codebase-index.json` ← used for staleness check
3. `.atlas/ml-overview.md` ← auto-included if present, unless `--no-ml`
4. `.atlas/architecture-diagram.md` ← auto-included if present

### B.3 — Auto-generate if missing (or `--fresh`)

If `.atlas/codebase-overview.md` does not exist OR `--fresh` was passed:

Print:
```
No .atlas/codebase-overview.md found.
Generating docs now — this takes 2–4 minutes on a large repo.
Running codebase-overview...
```

Run **Operation A: Codebase Overview**. After generation, continue to B.4.

### B.4 — Staleness check

> Skip if `--fresh` was passed or if docs were just generated in B.3.

Run **Procedure 1: Detect Git Changes**.

| Result | Action |
|--------|--------|
| `mode: "current"` | Silent — proceed. |
| `mode: "targeted"` | Print: `⚠ Docs may be slightly stale — N commits since last index (DATE). Affected sections: <list>. Run codebase-overview or ask --fresh to update.` |
| `mode: "full"` | Print: `⚠ Docs are stale — N commits since last index (DATE). Run codebase-overview or ask --fresh for accurate answers.` |

### B.5 — Load context

Use the Read tool. Do NOT use Glob or Grep on source files — only read these paths:
1. `.atlas/codebase-overview.md` (always)
2. `.atlas/ml-overview.md` (if exists and `--no-ml` not passed)
3. `.atlas/architecture-diagram.md` (if exists)
4. Each file from `--files`

### B.6 — Enter chat mode

Print:
```
Atlas loaded. Context: codebase-overview.md [+ ml-overview.md] [+ architecture-diagram.md] [+ N additional files].
Ask me anything about this codebase. I'll answer from the docs above.
To include more detail: ask --files src/some/file.py
```

If an inline question was provided in B.1, answer it immediately.

**Behavioural constraints for the entire chat session:**
1. Answer ONLY from the loaded docs and `--files` content. Do not derive answers from prior knowledge unless the docs confirm it.
2. Do NOT use Glob, Grep, or Read on source files not in `--files`.
3. Stay in chat mode for all follow-up messages — do not re-read docs or re-run staleness check.
4. If a question cannot be answered from loaded context: say so and suggest `--files <relevant-file>`.
5. Never guess file paths or implementation details not present in the loaded docs.

---

## Operation C: Architecture Diagram

Orchestrates architecture diagram generation by reading the best available knowledge source.

### C.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas architecture-diagram — Generate an architecture diagram

USAGE
  [options]

OPTIONS
  --format drawio        Generate draw.io only (default)
  --format excalidraw    Generate Excalidraw only
  --format both          Generate draw.io + Excalidraw
  --type system          Full system diagram (default)
  --type data-flow       Emphasise data movement
  --type sequence        Mermaid sequence diagram for one flow
  --type infra           Cloud infrastructure resources
  --flow <name>          Flow name for --type sequence
  --output <dir>         Override output directory (default: .atlas/)
  --fresh                Ignore existing files and regenerate

TIPS
  draw.io:    After opening, press Ctrl+Shift+H (Fit Page). Use Arrange > Layout.
  Excalidraw: Select all (Ctrl+A) and click "Tidy up".
  Mermaid:    Renders immediately in GitHub — no tool needed.
```

### C.1 — Determine input source

Use the Read tool to check in priority order:

| Priority | Source | Action |
|----------|--------|--------|
| 1 | `.atlas/codebase-index.json` | Best — structured component inventory already extracted |
| 2 | `.atlas/codebase-overview.md` | Good — components in prose form |
| 3 | `.atlas/ml-overview.md` | Supplement if present |
| 4 | None | Use Glob/Grep to explore directly; print suggestion to run codebase-overview first |

### C.2 — Run Procedure: Generate Diagram

Follow **Procedure 4: Generate Diagram** in this document with all user flags passed through.

---

## Operation D: ML Overview

Explores ML components in the current repository and produces `.atlas/ml-overview.md`.

### D.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas ml-overview — Generate or refresh a comprehensive ML codebase overview

USAGE
  [options]

OPTIONS
  --output <path>    Override output path (default: .atlas/ml-overview.md)
  --focus <area>     Extra depth, e.g. --focus "training pipeline"
  --fresh            Skip merge — write completely new file
```

### D.1 — Detect existing ML overview and index

Use the Read tool to check for:
- `.atlas/ml-overview.md` (or `--output` path)
- `.atlas/codebase-index.json`

If index exists: read it. Use `architecture.frameworks` and `files[].component_type` to identify ML files.

If overview exists and `--fresh` not passed: read it in full.

### D.2 — Read existing docs

Use the Read tool to read if present: `CLAUDE.md`, `README.md`, `.atlas/codebase-overview.md`.

### D.3 — Detect ML artefacts

**Fast path** (index exists): check `architecture.frameworks` for ML libraries.

**Full scan** (no index) — use Glob and Grep to find:

Model files: `*.pt`, `*.pth`, `*.ckpt`, `*.pb`, `*.h5`, `*.onnx`, `*.pkl`, `*.joblib`, `*.safetensors`

Framework imports: `torch`, `tensorflow`, `keras`, `sklearn`, `xgboost`, `lightgbm`, `catboost`, `jax`, `flax`, `transformers`, `diffusers`, `spacy`, `nltk`

Training infrastructure:
- Files: `train.py`, `fit.py`, `run_training.py`, files with `trainer` in name
- SageMaker: `estimator`, `TrainingJob`; Vertex AI: `CustomTrainingJob`

Data pipelines:
- Directories: `data/`, `datasets/`, `raw/`, `processed/`, `features/`
- DVC: `*.dvc`, `dvc.yaml`; Airflow: `dags/`, `DAG(`

Experiment tracking: `mlflow`, `wandb`, `comet_ml`, `neptune`, `clearml`, `tensorboard`

Serving: `serve.py`, `inference.py`, `predictor.py`, `torchserve`, `triton`, `bentoml`, `ray serve`, `seldon`

Notebooks: `*.ipynb` — use Read to check titles and first cells.

### D.4 — Deep exploration

Use the Read tool to read key ML files found above. Cover all 12 areas:
1. Model inventory (architecture, I/O schema, key hyperparameters, pretrained base)
2. Training pipelines (entry points, config management, hardware requirements)
3. Data sources (origin, format, volume, validation)
4. Data versioning (DVC, LFS, manifest files)
5. Feature engineering (preprocessing, feature stores, train/inference parity)
6. Experiment tracking (tool, what is logged, finding past experiments)
7. Evaluation strategy (metrics, validation approach, benchmarks)
8. Model registry & versioning (storage, promotion workflow)
9. Serving / inference (deployment target, batch vs real-time, latency SLAs)
10. Monitoring & drift detection (tools, what is monitored, alerting)
11. Retraining triggers (scheduled / performance / data-volume / manual)
12. Environment & dependencies (Python version, packages, CUDA, Docker)

### D.5 — Generate all 14 sections

Use concrete `file.py:ClassName.method()` references throughout.

1. **ML Components at a Glance** — table: model/pipeline, task, framework, status, entry point
2. **Model Inventory — Deep Dive** — per model: architecture, I/O schema, hyperparameters, pretrained base, artefact location
3. **Data Sources & Ingestion** — origin, formats, schemas, versioning, quality steps
4. **Feature Engineering Pipeline** — preprocessing, feature store, train/inference parity (critical — common source of silent bugs)
5. **Training Pipelines** — how to trigger, config management, distributed setup, hardware
6. **Experiment Tracking & Model Registry** — tool, logged items, finding experiments, promotion
7. **Evaluation & Metrics** — primary/secondary metrics, validation strategy, benchmarks
8. **Serving & Inference** — deployment target, batch vs real-time, I/O contract, loading pattern
9. **Retraining & Continuous Learning** — triggers, E2E flow with file paths, validation before promotion
10. **Monitoring & Drift Detection** — what is monitored, tool, alerting thresholds
11. **E2E ML Flows with Code Paths** — training flow, inference flow, retraining flow
12. **Tricky Parts & ML-Specific Nuances** — 8–15 items (train/serving skew, data leakage, etc.)
13. **Environment & Reproducibility** — Python version, packages, how to reproduce, CUDA requirements
14. **Key File Quick Reference** — table of most important files

### D.6 — Merge and write

Apply per-section merge rules: structurally changed → replace; additive → merge; unchanged → keep; uncertain → keep + `<!-- verify -->`. Never silently delete existing nuances.

Use the Write tool. Output file must begin with `<!-- Last updated: YYYY-MM-DD -->` then a cross-reference block.

### D Notes

- **Train/serving skew is the most common source of silent ML bugs.** Always investigate feature parity.
- For notebooks: use Read on the `.ipynb` file — markdown cells and outputs contain the best explanation of intent.
- If no experiment tracking found: note explicitly — it is a significant operational risk.

---

## Operation E: Ecosystem Overview

Orchestrates cross-repo analysis across a set of repositories.

### E.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas ecosystem-overview — Map how multiple repos interact

USAGE
  <repo1,repo2,...> [options]
  --repos-file <path> [options]

OPTIONS
  --repos-file <path>    Path to repos file (name: path-or-url per line)
  --output <dir>         Override output directory (default: .atlas/)
  --fresh                Ignore existing files and regenerate
  --no-diagram           Skip diagram files — overview doc only
  --with-excalidraw      Also generate .atlas/ecosystem.excalidraw
  --focus <area>         Extra depth: impact | data | flows | contracts

INPUT FILE FORMAT (--repos-file)
  # ecosystem-repos.txt
  service-a:  /path/to/service-a
  service-b:  https://github.com/org/service-b

REQUIRES (for GitHub URLs)
  gh CLI installed and authenticated: gh auth login
```

### E.1 — Parse inputs

**Inline repos** (comma-separated): items starting with `https://github.com/` → GitHub URL; others → local path.

**Repos file**: use Read to load it. Parse non-comment, non-blank lines as `<name>: <path-or-url>`.

Build list: `[{ name, source_type: "local"|"github", path_or_url }]`

### E.2 — Access each repo

**Local repos:** verify with Bash: `git -C <path> rev-parse --git-dir`

**GitHub repos:**
1. Check gh CLI: `gh --version`
2. Shallow-clone: `gh repo clone <url> /tmp/ecosystem-overview/<name> -- --depth=1 --single-branch`

On failure: mark FAILED, continue. If all fail, stop.

### E.3 — Run Procedure: Explore Repo Interface (per repo)

Follow **Procedure 5: Explore Repo Interface** for each accessible repo. Run in parallel where possible.

### E.4 — Build the dependency graph

From interface profiles, resolve dependencies between repos:

Matching strategy (try in order):
1. Exact URL match — outbound HTTP call matches another repo's known host
2. Topic name match
3. Queue name match
4. Endpoint path match
5. Named reference in config or README

Record: direction, mechanism, criticality (SYNC/ASYNC), label. Unresolved → `EXTERNAL: <host/topic>` nodes.

### E.5 — Impact analysis

For each repo:

**Blast radius**: SYNC dependents fail immediately; ASYNC dependents stop receiving updates; transitive (A→B→C: C affected if B down).

**Severity:**
- `P0` — sync, no fallback, on core path
- `P1` — sync with circuit breaker / fallback
- `P2` — async, consumer has DLQ / retry
- `P3` — async, best-effort

### E.6 — Identify cross-repo E2E flows

Trace significant journeys crossing ≥2 repos. Notation:
```
[Actor] → [Repo A]: mechanism — action
         → [Repo B]: mechanism — action  (async)
                   → [Repo C]: mechanism — action
         ← [Repo A]: response to actor
```

### E.7 — Write `.atlas/ecosystem-overview.md`

Use the Write tool. Structure:
```
<!-- Last updated: YYYY-MM-DD -->
# Ecosystem Overview
> Repos analysed: ...

## 1. Ecosystem at a Glance
## 2. Dependency Graph (ASCII)
## 3. Shared Infrastructure
## 4. API & Event Contract Map
## 5. Cross-Repo E2E Flows
## 6. Impact Analysis (Blast Radius + SPOFs + Cascade Scenarios)
## 7. Data Ownership Map
## 8. Coupling & Cohesion Assessment
## 9. Per-Repo Interface Profiles (Raw Data)
## 10. Excluded Repos & Next Steps
```

### E.8 — Run Procedure: Generate Diagram *(unless `--no-diagram` passed)*

Follow **Procedure 4: Generate Diagram** with ecosystem conventions:
- Each repo = a swim lane group
- REST edges: solid; Kafka/event: dashed; SQS: dotted
- EXTERNAL nodes: grey fill, dashed border

For Mermaid: produce **two diagrams** in `.atlas/ecosystem-diagram.md`:
1. Full component diagram (`graph LR`) — every component named, never abbreviated
2. Critical path summary — one node per repo, P0/P1 edges highlighted

If `--with-excalidraw`: also generate `.atlas/ecosystem.excalidraw`.

### E.9 — Cleanup

```bash
rm -rf /tmp/ecosystem-overview/
```

### E Notes

- Partial success is better than total failure — analyse available repos, document failures.
- SYNC vs ASYNC is the most important field to get right.
- Never abbreviate components in diagrams.

---

## Procedure 1: Detect Git Changes

Determine what changed since the last `.atlas/codebase-overview.md` update.

### Output

```
mode:             "current" | "targeted" | "full"
changed_files:    [list of file paths]
affected_sections:[list of section names]
commit_messages:  [list of commit subject lines]
reason:           human-readable explanation
```

### Step 1 — Extract last-updated date

**Prefer the index**: use Read on `.atlas/codebase-index.json`, read `meta.generated_at`.

**Fallback**: use Read on `.atlas/codebase-overview.md`, parse `<!-- Last updated: YYYY-MM-DD -->` from line 1.

If neither → `mode: "full"`, reason: "Could not determine last-updated date".

### Step 2 — Get commits since that date

```bash
git log --since="LAST_UPDATED_DATE" --oneline
```

| Outcome | Result |
|---------|--------|
| `git` not available | `mode: "full"` |
| Not a git repo | `mode: "full"` |
| Zero commits | `mode: "current"` |
| One or more commits | Continue to Step 3 |

### Step 3 — Get changed files and commit messages

```bash
git log --since="LAST_UPDATED_DATE" --name-only --pretty=format:""
git log --since="LAST_UPDATED_DATE" --pretty=format:"%s"
```

Deduplicate file paths, strip blank lines → `changed_files`. Collect commit subjects → `commit_messages`.

### Step 4 — Decide mode

Count distinct packages (unique parent directories of changed files).

| Condition | Mode |
|-----------|------|
| Hub file changed: `go.mod`, `package.json`, `requirements.txt`, primary domain model, `main.go`, `main.py`, `index.ts` | `full` |
| >20 files changed | `full` |
| >5 distinct packages changed | `full` |
| ≤20 files AND ≤5 packages AND no hub files | `targeted` |

### Step 5 — Map to affected sections (targeted mode only)

**Prefer index**: for each file in `changed_files`, read its `feeds_sections` from `.atlas/codebase-index.json`. Union all → `affected_sections`.

**Fallback heuristics** (no index):

| File pattern | Sections affected |
|---|---|
| `domain/`, `models/`, domain structs | `Domain Models`, `State Machine Reference`, `ID / Key System` |
| `handlers/`, `routes/`, `controllers/` | `E2E Flows`, `Component Map`, `Key File Quick Reference` |
| `*repository*`, `*_store*`, db config | `Data Layer`, `E2E Flows` |
| `lambdas/`, `functions/` | `Component Map`, `Repository Layout`, `E2E Flows` |
| `service/`, `usecases/` | `E2E Flows`, `Architecture Patterns` |
| `config/`, `.env*` | `Data Layer`, `Architecture Patterns` |
| `*_test.*`, `*.spec.*` | `Testing Approach` |

---

## Procedure 2: Index Codebase

Build or update `.atlas/codebase-index.json`.

### Index Schema

```json
{
  "meta": {
    "schema_version": "1",
    "generated_at": "YYYY-MM-DD",
    "git_sha": "abc123",
    "repo_name": "derived from root directory name"
  },
  "files": [
    {
      "path": "internal/domain/delivery.go",
      "component_type": "domain_model",
      "symbols": ["Delivery", "DeliveryStatus", "StatusPending"],
      "feeds_sections": ["Domain Models", "State Machine Reference", "ID / Key System"],
      "last_modified_sha": "def456"
    }
  ],
  "components": [
    {
      "id": "delivery-api",
      "name": "Delivery API",
      "type": "api",
      "layer": 1,
      "files": ["internal/rest/handler.go"],
      "deps": ["delivery-service", "dynamo-deliveries"]
    }
  ],
  "entities": [
    {
      "name": "Delivery",
      "file": "internal/domain/delivery.go",
      "id_field": "delivery_id",
      "states": ["pending", "in_progress", "completed", "cancelled"]
    }
  ],
  "id_types": [
    {
      "name": "delivery_id",
      "format": "UUID v4",
      "creator": "Delivery API on POST /deliveries",
      "used_in": ["deliveries table", "pickup service"]
    }
  ],
  "flows": [
    {
      "name": "Create Delivery",
      "entry": "internal/rest/handler.go:CreateDelivery",
      "steps": [
        "internal/service/delivery.go:Create",
        "internal/repo/delivery_repo.go:Put"
      ]
    }
  ],
  "section_file_map": {
    "Repository Layout": [],
    "Component Map": ["cmd/", "internal/service/"],
    "Domain Models": ["internal/domain/delivery.go"],
    "ID / Key System": ["internal/domain/delivery.go"],
    "E2E Flows": ["internal/rest/handler.go"],
    "Architecture Patterns": ["go.mod"],
    "Data Layer": ["internal/repo/delivery_repo.go"],
    "Tricky Parts": [],
    "State Machine Reference": ["internal/domain/delivery.go"],
    "Testing Approach": ["internal/service/delivery_test.go"],
    "Key File Quick Reference": []
  },
  "architecture": {
    "languages": ["Go"],
    "frameworks": ["gin", "aws-sdk-go-v2"],
    "deployment": ["lambda", "ecs"],
    "databases": [{ "type": "dynamodb", "tables": ["deliveries"] }],
    "queues": [{ "type": "sqs", "names": ["delivery-events-queue"] }],
    "external_services": ["stripe", "twilio"]
  }
}
```

### Enum Reference

**`component_type`:**

| Value | When to use |
|-------|-------------|
| `domain_model` | Core entity / struct / enum |
| `handler` | HTTP handler, route, controller |
| `service` | Business logic layer |
| `repo` | Data access / repository layer |
| `lambda` | Serverless function entry point |
| `consumer` | Message queue / event consumer |
| `worker` | Background worker / cron job |
| `config` | Configuration / environment setup |
| `infra` | Infrastructure definitions (Terraform, CDK, k8s) |
| `test` | Test file — do not deep-index |
| `other` | Does not fit above |

**`component.layer`:**

| Value | Meaning |
|-------|---------|
| `0` | Actor / external client |
| `1` | API / gateway / public-facing entry point |
| `2` | Internal service / worker / lambda / consumer |
| `3` | Data store / message queue / cache |
| `4` | External third-party service |

**`feeds_sections` allowed values** (use exact strings):
`"Repository Layout"`, `"Component Map"`, `"Domain Models"`, `"ID / Key System"`, `"E2E Flows"`, `"Architecture Patterns"`, `"Data Layer"`, `"Tricky Parts"`, `"State Machine Reference"`, `"Testing Approach"`, `"Key File Quick Reference"`

### File → Section Mapping

| File pattern | `feeds_sections` |
|---|---|
| Domain entity / struct / status enum | `Domain Models`, `State Machine Reference`, `ID / Key System` |
| HTTP handler / route / controller | `E2E Flows`, `Component Map`, `Key File Quick Reference` |
| Service / business logic | `E2E Flows`, `Architecture Patterns` |
| Repository / data access | `Data Layer`, `E2E Flows` |
| Lambda / serverless entry point | `Component Map`, `Repository Layout`, `E2E Flows` |
| Consumer / async worker | `Component Map`, `Repository Layout`, `E2E Flows` |
| Configuration / env setup | `Data Layer`, `Architecture Patterns` |
| Dependency manifest (`go.mod`, `package.json`, `requirements.txt`) | `Architecture Patterns` |
| Infrastructure files (`*.tf`, `docker-compose.yml`, `k8s/`) | `Architecture Patterns`, `Repository Layout` |
| Test files | `Testing Approach` |

### Mode A — Full index

**1.** Run `git rev-parse HEAD` with Bash. Record in `meta.git_sha`.

**2.** Use Glob to discover all significant files. Skip: `vendor/`, `node_modules/`, `.git/`, `dist/`, `build/`, `*_generated.go`, `*.pb.go`, `mock_*.go`, `__pycache__/`, binaries (`*.so`, `*.dylib`, `*.exe`, `*.wasm`).

**3.** For each file, use Read to classify:
- `component_type`: infer from path and content
- `symbols`: extract exported names — Go: `type Foo struct`, `func Foo(`; Python: `class Foo`, `def foo`; TypeScript: `export const/function/class/interface`
- `feeds_sections`: apply mapping table above
- `last_modified_sha`: Bash `git log -1 --format="%H" -- <filepath>` (null if not in git)

Do not deep-index test files. Skip generated code.

**4.** Group files into logical components: `id` (URL-slug), `name`, `type`, `layer`, `files[]`, `deps[]` (from imports and config).

**5.** For each `domain_model` file: extract entity `name`, `id_field`, `states[]`.

**6.** Find identifier fields (`*ID`, `*Id`, `*_id`, `*Key`). Record `name`, `format`, `creator`, `used_in[]`.

**7.** For each handler/entry-point, use Read to trace the call chain 2–3 levels deep. Record `name`, `entry`, `steps[]`.

**8.** Build `section_file_map` by inverting per-file `feeds_sections`. Deduplicate.

**9.** From imports/config/manifests, detect languages, frameworks, databases, queues, external services.

**10.** Use Write to save `.atlas/codebase-index.json`. Create `.atlas/` first if needed.

### Mode B — Targeted update

Input: comma-separated changed file paths.

1. Use Read to load existing `.atlas/codebase-index.json`.
2. For each changed file:
   - **Deleted**: remove from `files[]`, remove from component `files[]`, rebuild deps.
   - **New**: classify and add to `files[]`, assign to or create a component.
   - **Modified**: use Read to re-read file, update `symbols`, `feeds_sections`, `last_modified_sha`, update affected entities/flows.
3. Rebuild `section_file_map`.
4. Update `meta.generated_at` and `meta.git_sha`.
5. Use Write to save updated index.

Do not re-read files not in the changed list.

### Notes

- Correctness over completeness — 80% accurate beats 100% guessed.
- `section_file_map` is the highest-value output — invest care in `feeds_sections` accuracy.

---

## Procedure 3: Write Overview Doc

Read `.atlas/codebase-index.json` and produce `.atlas/codebase-overview.md`.

**Prerequisite:** `.atlas/codebase-index.json` must exist. If not, stop and say "Run Procedure 2: Index Codebase first, or run Operation A: Codebase Overview."

### Step 1 — Load the index

Use Read on `.atlas/codebase-index.json`. This gives: `section_file_map`, component inventory, entities, ID types, flows, language.

### Step 2 — Load existing overview (for merge)

If `.atlas/codebase-overview.md` exists AND `--fresh` not passed: use Read to load it in full, parse `Last updated` date. In targeted mode, sections NOT in `--sections` will be kept verbatim.

If `--fresh`: ignore existing file.

### Step 3 — Read source files per section

Use the Read tool. Use `section_file_map` to know exactly which files to read for each section being generated. **Do not read files not listed** in the relevant section's map entry.

For sections with empty map entries (Repository Layout, Tricky Parts, Key File Quick Reference): derive content from the index structure itself.

### Step 4 — Generate content

**Full mode**: produce all 11 required sections.

**Targeted mode** (`--sections` provided): produce only those sections; carry all others verbatim.

Always use concrete `file.go:FunctionName()` references.

### Step 5 — Merge (if existing file present)

| Situation | Action |
|---|---|
| Structurally changed | Replace with new content |
| Additive only | Merge in; keep existing accurate items |
| Unchanged | Keep existing wording |
| Uncertain | Keep content, append `<!-- verify -->` |

Never silently delete existing nuances or gotchas.

### Step 6 — Write the final file

Use the Write tool (or Edit if merging). File must begin with `<!-- Last updated: YYYY-MM-DD -->` (today's date), then a cross-reference block linking to companion docs, then the full content.

### Required Sections (11)

**1. Repository Layout** — annotated directory tree from index `files[]` and directory structure.

**2. Component Map** — table from `components[]`:

| Component | Type | Purpose | Entry Point |
|---|---|---|---|

**3. Domain Models Deep Dive** — field-level breakdown: key fields, status enums, lifecycle, constraints. Include all states and behavioural meaning.

**4. ID / Key System** — highest-value section for new engineers:

| ID Type | Format | Who Creates It | Where Used |
|---|---|---|---|

**5. E2E Flows with Code Paths** — every significant journey with `file.go:Fn()` references. **Always trace async continuations.** Format:
```
1. Partner → POST /v1/deliveries
2. handler.go:CreateDelivery → validates input
3. service.go:Create → applies business rules
4. repo.go:Put → writes to DynamoDB
5. HTTP 201 returned
6. (async) DynamoDB stream → lambda.go:ProcessNew → publishes event
```

**6. Architecture Patterns** — recurring patterns (event-driven, CQRS, saga, etc.). For each: name, why it's used here, file reference.

**7. Data Layer** — tables/collections, indexes, consistency model, caching, access patterns. Include key schema (partition key, sort key, indexes).

**8. Tricky Parts & Nuances** — 8–15 non-obvious behaviours. Always explain the *why*, not just the *what*.

**9. State Machine Reference** — ASCII state transition diagrams. List all valid transitions and triggers.
```
pending → in_progress → completed
       ↘              ↗
        cancelled
```

**10. Testing Approach** — unit vs integration vs E2E, how to run each, test patterns.

**11. Key File Quick Reference** — table of most important files:

| File | Purpose |
|---|---|

### Notes

- E2E flows must include async continuations.
- The ID/Key System section is highest-value; where subtle bugs hide.
- The doc serves two audiences: someone new AND someone debugging a production incident at 2 AM.
- In targeted mode: do NOT regenerate sections not in `--sections`.

---

## Procedure 4: Generate Diagram

Produce architecture diagram files from available codebase knowledge.

### Input Priority

Use Read to check in order:
1. `.atlas/codebase-index.json` — best
2. `.atlas/codebase-overview.md` — components in prose
3. `.atlas/ml-overview.md` — supplement if present
4. Direct exploration with Glob/Grep — fallback

### Step 1 — Build Component Inventory

**Nodes:**
- Internal services / lambdas / workers
- Databases / data stores
- Message queues / topics
- External third-party services
- Actors / clients

**Edges:** direction (A → B), protocol (REST/gRPC/Kafka/SQS/stream/WebSocket), label.

**When reading from index:** nodes ← `components[]`, edges ← `components[].deps`, groups ← `layer` field, external ← `architecture.external_services`.

**MANDATORY: Never abbreviate.** Every Lambda, consumer, worker, data store, and queue must be its own named node.

### Step 2 — Assign Layout Grid

```
Row 0 (y=80):   Clients / Actors
Row 1 (y=220):  Public-facing API / Gateway
Row 2 (y=380):  Internal Services / Workers / Lambdas
Row 3 (y=540):  Data Stores (databases, caches, queues)
Row 4 (y=700):  External Services / Third-party
```

Horizontal: 180px wide nodes, 40px gap, start x=80px. Assign each node a slug ID and `(x, y)`.

### Step 3 — Generate draw.io XML

Use Write to output `.atlas/architecture.drawio`.

Root template:
```xml
<mxGraphModel dx="1422" dy="762" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1654" pageHeight="1169" math="0" shadow="0">
  <root>
    <mxCell id="0" />
    <mxCell id="1" parent="0" />
    <!-- NODES and EDGES here, IDs from 2 -->
  </root>
</mxGraphModel>
```

Node styles:
| Type | `style` |
|------|---------|
| HTTP API / service | `rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;` |
| Lambda / worker | `rounded=1;whiteSpace=wrap;html=1;fillColor=#d5e8d4;strokeColor=#82b366;` |
| Database | `shape=mxgraph.flowchart.database;fillColor=#f8cecc;strokeColor=#b85450;` |
| Kafka / SQS | `shape=hexagon;fillColor=#fff2cc;strokeColor=#d6b656;` |
| External service | `rounded=1;whiteSpace=wrap;html=1;fillColor=#f5f5f5;strokeColor=#666666;fontColor=#333333;` |
| Actor / client | `ellipse;fillColor=#f0f0f0;strokeColor=#666666;` |

Node template:
```xml
<mxCell id="NODE_ID" value="Display Name" style="STYLE" vertex="1" parent="1">
  <mxGeometry x="X" y="Y" width="W" height="H" as="geometry" />
</mxCell>
```

Edge template:
```xml
<mxCell id="EDGE_ID" value="label" style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;" edge="1" source="SOURCE_ID" target="TARGET_ID" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

### Step 4 — Generate Excalidraw JSON *(only if `--format excalidraw` or `--format both`)*

Use Write to output `.atlas/architecture.excalidraw`. Root structure (v2):
```json
{
  "type": "excalidraw", "version": 2, "source": "atlas",
  "elements": [],
  "appState": { "viewBackgroundColor": "#ffffff", "gridSize": 20 },
  "files": {}
}
```

Fill colours: API `#a5d8ff`, Lambda/worker `#b2f2bb`, Database `#ffc9c9`, Queue `#ffec99`, External `#e9ecef`, Actor `#dee2e6`.

### Step 5 — Generate Mermaid Preview

Always generate regardless of format flags. Use Write for `.atlas/architecture-diagram.md`.

Use `graph LR` for multi-layer systems; `graph TD` only for simple linear pipelines (≤5 rows).

Apply classDefs:
```
classDef api fill:#dae8fc,stroke:#6c8ebf
classDef worker fill:#d5e8d4,stroke:#82b366
classDef db fill:#f8cecc,stroke:#b85450
classDef queue fill:#fff2cc,stroke:#d6b656
classDef external fill:#f5f5f5,stroke:#666666
```

File format:
```markdown
<!-- Last updated: YYYY-MM-DD -->

# Architecture Diagram

> Auto-generated. Open `.atlas/architecture.drawio` in [diagrams.net](https://app.diagrams.net).
> **draw.io tip:** Press `Ctrl+Shift+H` (Fit Page) then `Arrange > Layout` to auto-arrange.
> Run atlas codebase-overview to regenerate source docs.

```mermaid
graph LR
    ...
```
```

### Step 6 — Merge with existing files

Use Read on existing files first. Detect new components/edges and add them. Preserve hand-tuned coordinates (if existing node coords differ >50px from default grid — they were manually adjusted, keep them). Update `Last updated` date.

If `--fresh`: overwrite without merging.

### Parameters

- `--format drawio` *(default)*: draw.io XML only
- `--format excalidraw`: Excalidraw JSON only
- `--format both`: both draw.io and Excalidraw
- `--type system` *(default)*: full system, all components
- `--type data-flow`: emphasise data movement
- `--type sequence`: Mermaid sequenceDiagram for one flow (use with `--flow`)
- `--type infra`: cloud infrastructure resources
- `--flow <name>`: flow name for `--type sequence`
- `--output <dir>`: override output directory
- `--fresh`: ignore existing files and regenerate

### Notes

- Always generate the Mermaid preview — fastest win for GitHub rendering.
- If >30 nodes: produce two diagrams — (1) full component view, (2) simplified critical-path view.
- For sequence diagrams: use `sequenceDiagram` syntax with `participant` declarations.

---

## Procedure 5: Explore Repo Interface

Extract a repository's cross-repo interface profile.

### Output Format

```
REPO: <display-name>
PATH: <local path>

EXPOSES:
  REST/GraphQL/gRPC endpoints:
    - METHOD /path — brief purpose
  Kafka / Pub-Sub topics PUBLISHED:
    - topic-name (message type) — trigger
  SQS / event queues PUBLISHED:
    - queue-name — trigger
  Webhooks sent:
    - target type, payload shape, trigger

DEPENDS ON:
  HTTP/gRPC calls TO other services:
    - Target: <service or URL>, endpoint, purpose, SYNC/ASYNC
  Kafka / Pub-Sub topics CONSUMED:
    - topic-name — what it does
  SQS / event queues CONSUMED:
    - queue-name — processing summary
  Databases READ from other repos:
    - table/bucket name, owner service

OWNS:
  Databases / data stores:
    - name, type, what data lives there
  Message queues / topics:
    - name, type

TECH PROFILE:
  Language + framework
  Deployment model
  Auth mechanism
```

### Step 1 — Fast path

Use Read to check for `.atlas/codebase-index.json`:
- `EXPOSES` ← components with `type: "api"` and `layer: 1`
- `DEPENDS ON` ← `architecture.external_services` and `layer: 4` components
- `OWNS` ← `architecture.databases` and `architecture.queues`
- `TECH PROFILE` ← `architecture.languages`, `architecture.frameworks`, `architecture.deployment`

At most 2–3 targeted Read calls to fill gaps.

If no index but `.atlas/codebase-overview.md` exists: use Read — Component Map, Data Layer, Architecture Patterns sections answer most questions.

### Step 2 — Deep exploration (no index, no overview)

Use Glob and Grep to find:

**Endpoints exposed:** route registration files (`router.go`, `routes.py`, `app.ts`), handler files (`*handler*`, `*controller*`), OpenAPI specs (`openapi.yml`, `swagger.yml`), `.proto` files.

**Outbound HTTP/gRPC:** HTTP client instantiation (`http.NewRequest`, `axios`, `fetch`, `requests.get`, `grpc.Dial`), client wrapper files (`*_client.go`, `clients/`), config files with base URLs.

**Kafka/SQS topics:** publishing (`Publish(`, `SendMessage(`, `producer.send(`), consuming (`Subscribe(`, `ReceiveMessage(`, consumer group config, `@KafkaListener`).

**Data stores owned:** infrastructure files (`*.tf`, `serverless.yml`), migration files, README.

**Auth:** middleware files (`auth.go`, `middleware/`), JWT validation, OAuth2, API key headers, mTLS config.

### Step 3 — Classify dependencies

- **SYNC** if the caller blocks waiting for a response (HTTP request, gRPC call)
- **ASYNC** if the caller does not wait (Kafka, SQS, DynamoDB stream)

### Step 4 — Return the profile

Output the structured profile. Use `<!-- unconfirmed -->` for relationships that seem likely but aren't explicitly visible.

### Notes

- Prefer existing docs over re-exploration.
- **SYNC vs ASYNC is the most critical field.**
- Scope is cross-repo surface only — internal architecture is out of scope.
- Config files are gold: they often explicitly name external URLs, topic names, queue ARNs.

---

## Operation F: Domain Models

Deeply explores a service repo and generates `docs/domain-models.md` — a comprehensive human-readable domain model reference with Mermaid diagrams. Automatically discovers related transport/proto repos in sibling directories.

### F.0 — Handle `--help`

If `--help` or `-h` was passed, print:

```
atlas domain-models — Generate a domain model reference for a service

USAGE
  [repo-path-or-name] [options]

OPTIONS
  <repo-path-or-name>    Target repo (default: current working directory)
  --help, -h             Show help
```

### F.1 — Resolve the target repo

If `$ARGUMENTS` was provided, treat it as either:
- An absolute path (use as-is)
- A repo name — search for it as a subdirectory of the current working directory

If no argument was provided, use the current working directory as `TARGET_REPO`.

### F.2 — Discover sibling transport/proto repos

From the parent directory of `TARGET_REPO`, use Glob and Bash to list all sibling directories. Identify those likely to contain shared transport models or protobuf definitions:
- Directory names containing: `transport`, `proto`, `grpc`, `models`, `schema`, `contracts`, `events`
- Files matching: `**/*.proto`, `**/transport_models*`, `**/grpc_models*`
- Go modules or packages imported by `TARGET_REPO`

Use Read on the top-level `go.mod`, `package.json`, `Cargo.toml`, `Gemfile`, or `requirements.txt` of `TARGET_REPO` for shared model imports and cross-reference with sibling directories.

Call the discovered list `TRANSPORT_REPOS`.

### F.3 — Explore TARGET_REPO deeply

Use Read, Glob, and Grep to investigate:

**Language & structure:** detect language(s), read `README.md` and any architecture docs, map top-level directory layout.

**Domain models** — look in: `internal/domain/`, `app/models/`, `src/models/`, `lib/`, `domain/`, `pkg/`, `crates/`

For each model/struct/class found:
- All fields with types
- Relationships to other models (foreign keys, embedded structs, associations)
- Status enums or state machine fields
- Key methods or business logic on the model
- DRN/UUID/ID fields that cross service boundaries

**State machines:** find status enums and transitions via `switch/case` blocks or state machine libraries. Note any special override transitions.

**Data persistence:** databases (DynamoDB, Postgres, Redis, MySQL, MongoDB), table/collection schemas and key structures (partition keys, sort keys, GSIs), caching patterns.

**Event/message flows:** Kafka topics consumed and produced, SQS queues, SNS topics, webhooks, DynamoDB streams, gRPC services.

**External integrations:** other services called, auth patterns.

**API surface:** REST endpoints (path, method, request/response shape), gRPC methods, WebSocket endpoints.

### F.4 — Explore TRANSPORT_REPOS

For each repo in `TRANSPORT_REPOS`, use Read to:
- Read `.proto` files, Go structs, or schema files relevant to `TARGET_REPO`
- Focus on message types `TARGET_REPO` produces or consumes
- Note field names and types that cross the service boundary

### F.5 — Write docs/domain-models.md

Use Write to create `TARGET_REPO/docs/domain-models.md`. Include only sections with content — skip empty sections rather than writing placeholders.

Structure:
- **Overview** — language/framework, key responsibilities (3-5 bullets), top-level repo structure table
- **High-Level Architecture** — Mermaid `graph TD` showing external inputs, service components, data stores, outputs
- **Core Domain Models** — Mermaid `classDiagram` per entity + prose description, lifecycle, and relationships
- **[Entity] vs [Entity]** — side-by-side comparison when two similar models need distinguishing
- **Status Lifecycles** — Mermaid `stateDiagram-v2` per entity with a state machine; note non-obvious transitions
- **Key Flows** — Mermaid `sequenceDiagram` per major operation (creation, update, cancellation)
- **External Integrations** — table: service name | purpose | protocol | direction
- **Kafka / Event Integration** — consumed topics table + produced topics table
- **Data Persistence** — table per store: store | purpose | key schema | notable patterns
- **API Surface** — endpoint table with method, path, auth, description
- **Cross-Service ID Glossary** — table of ID types + Mermaid `graph LR` showing cross-boundary ID relationships

**Mermaid rules (follow strictly to avoid parse errors):**
1. No `[]` in relationship targets — use cardinality notation: `Entity "1" --> "*" OtherEntity`
2. No spaces or dots in quoted node IDs: use `Assignment -->|1..*| DR[DeliveryRequest]`
3. No parentheses in bare node IDs: use `DeliveryDrn["delivery_drn - Roogo"] --> X`
4. Pipe characters only inside `|label|` edge syntax
5. classDiagram array fields: use `Item list` not `[]Item`

**Quality bar:** A new engineer can understand the service in 5 minutes (Overview + Architecture), find any domain model (Core Domain Models), trace a request end-to-end (Key Flows), and know which IDs to use across services (ID Glossary).
