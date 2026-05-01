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
| Generate/refresh codebase documentation, explore architecture, document a service, index codebase | **A: Codebase Overview** |
| Ask questions about this codebase, "how does X work?", explain a flow or component | **B: Ask Atlas** |
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

Check for:
- `.atlas/codebase-index.json`
- `.atlas/codebase-overview.md` (or `--output` path)

If `--fresh` was passed: skip Step A.2 entirely.

### A.2 — Run Procedure: Detect Git Changes

> Skip if `--fresh` was passed or if neither index nor overview doc exists.

Follow **Procedure 1: Detect Git Changes**.

| Result | Action |
|--------|--------|
| `mode: "current"` | Update the `Last updated` date only. Print: `"No commits since <date> — doc is current. Refreshing date only."` Done. |
| `mode: "targeted"` | Proceed to A.3 in targeted mode with `changed_files` and `affected_sections`. |
| `mode: "full"` | Proceed to A.3 in full mode. |

### A.3 — Run Procedure: Index Codebase

Follow **Procedure 2: Index Codebase**.

- **Full mode**: Run Mode A (index all files).
- **Targeted mode**: Run Mode B with `--files <changed_files>`.
- If `--code-only`: do not read any existing files in `.atlas/` during indexing.

### A.4 — Run Procedure: Write Overview Doc

Follow **Procedure 3: Write Overview Doc**.

- **Full mode**: generate all required sections.
- **Targeted mode**: pass `--sections <affected_sections>`; carry all other sections forward verbatim.
- Pass through `--output`, `--focus`, `--fresh`.

### A.5 — Run Procedure: Generate Diagram *(only if `--with-diagram` passed)*

Follow **Procedure 4: Generate Diagram**.

Map flags:
- `--with-diagram` → `--format drawio`
- `--with-diagram=excalidraw` → `--format excalidraw`
- `--with-diagram=both` → `--format both`

After diagram is written, add a cross-reference to `.atlas/architecture-diagram.md` in the companion docs block at the top of `.atlas/codebase-overview.md`.

### A.6 — ML artefact detection

Scan for ML signals:
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
- If `.atlas/` does not exist, create it.

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
- **`--files`**: comma-separated paths; read each in full at Step B.5
- **`--fresh`**: if present, regenerate all docs before entering chat mode
- **`--no-ml`**: if present, skip `.atlas/ml-overview.md`

### B.2 — Check for existing docs

Look for (do NOT search the source tree — only check these paths):
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
| `mode: "targeted"` | Print: `⚠ Docs may be slightly stale — N commits since last index (DATE). Affected sections: <list>.` |
| `mode: "full"` | Print: `⚠ Docs are stale — N commits since last index (DATE). Run codebase-overview or ask --fresh for accurate answers.` |

### B.5 — Load context

Read using the read file tool. Do NOT search source files — only these paths:
1. `.atlas/codebase-overview.md` (always)
2. `.atlas/ml-overview.md` (if exists and `--no-ml` not passed)
3. `.atlas/architecture-diagram.md` (if exists)
4. Each file from `--files`

### B.6 — Enter chat mode

Print:
```
Atlas loaded. Context: codebase-overview.md [+ ml-overview.md] [+ architecture-diagram.md] [+ N additional files].
Ask me anything about this codebase.
```

If an inline question was provided in B.1, answer it immediately.

**Behavioural constraints for the entire chat session:**
1. Answer ONLY from the loaded docs and `--files` content.
2. Do NOT search or read source files not in `--files`.
3. Stay in chat mode for all follow-up messages — do not re-read docs.
4. If a question cannot be answered: say so and suggest `--files <relevant-file>`.
5. Never guess file paths or implementation details not in the loaded docs.

---

## Operation C: Architecture Diagram

Orchestrates architecture diagram generation by reading the best available knowledge source and delegating to the diagram procedure.

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
```

### C.1 — Determine input source

Check in priority order:

| Priority | Source | Action |
|----------|--------|--------|
| 1 | `.atlas/codebase-index.json` | Best — use it |
| 2 | `.atlas/codebase-overview.md` | Good — components in prose |
| 3 | `.atlas/ml-overview.md` | Supplement if present |
| 4 | None | Explore codebase directly; print suggestion to run codebase-overview first |

### C.2 — Run Procedure: Generate Diagram

Follow **Procedure 4: Generate Diagram** with all user flags passed through.

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

Check for:
- `.atlas/ml-overview.md` (or `--output` path) — existing doc to merge into
- `.atlas/codebase-index.json` — fast-path for ML artifact detection

If index exists: read it. Use `architecture.frameworks` and `files[].component_type` to identify likely ML files.

If overview exists and `--fresh` not passed: read it in full.

### D.2 — Read existing docs

If present, read: `CLAUDE.md`, `README.md`, `.atlas/codebase-overview.md`, and other files in `.atlas/`.

### D.3 — Detect ML artefacts

**Fast path** (index exists): check `architecture.frameworks` for ML libraries and `files[]` for ML symbols.

**Full scan** (no index):

Model files: `*.pt`, `*.pth`, `*.ckpt`, `*.pb`, `*.h5`, `*.onnx`, `*.pkl`, `*.joblib`, `*.safetensors`

Framework imports: `torch`, `tensorflow`, `keras`, `sklearn`, `xgboost`, `lightgbm`, `catboost`, `jax`, `flax`, `transformers`, `diffusers`, `spacy`, `nltk`

Training infrastructure:
- `train.py`, `fit.py`, `run_training.py`, files with `trainer` in name
- SageMaker: `estimator`, `TrainingJob`; Vertex AI: `CustomTrainingJob`

Data pipelines:
- `data/`, `datasets/`, `raw/`, `processed/`, `features/`
- DVC: `*.dvc`, `dvc.yaml`; Airflow: `dags/`, `DAG(`; Prefect/Luigi/Metaflow/Kedro

Experiment tracking: `mlflow`, `wandb`, `comet_ml`, `neptune`, `clearml`, `tensorboard`

Feature stores: `feast`, `tecton`, `hopsworks`

Serving: `serve.py`, `inference.py`, `predictor.py`, `torchserve`, `triton`, `bentoml`, `ray serve`, `seldon`

Notebooks: `*.ipynb` — read titles and first few cells.

### D.4 — Deep exploration

Read key files found above. Cover all 12 areas:
1. Model inventory (architecture, input/output schema, key hyperparameters, pretrained base)
2. Training pipelines (entry points, config/hyperparameter management, hardware requirements)
3. Data sources (origin, format, volume, validation)
4. Data versioning (DVC, LFS, manifest files)
5. Feature engineering (preprocessing, feature stores, train/inference parity)
6. Experiment tracking (tool, what is logged, how to find past experiments)
7. Evaluation strategy (metrics, validation approach, benchmarks)
8. Model registry & versioning (storage, promotion workflow)
9. Serving / inference (deployment target, batch vs real-time, latency SLAs)
10. Monitoring & drift detection (tools, what is monitored, alerting)
11. Retraining triggers (scheduled / performance / data-volume / manual)
12. Environment & dependencies (Python version, packages, CUDA, Docker)

### D.5 — Generate content

Produce all 14 required sections with concrete file path references:

1. **ML Components at a Glance** — table: model/pipeline, task, framework, status, entry point
2. **Model Inventory — Deep Dive** — per model: architecture, I/O schema, hyperparameters, pretrained base, artefact location
3. **Data Sources & Ingestion** — origin, formats, schemas, versioning, quality steps
4. **Feature Engineering Pipeline** — preprocessing, feature store, train/inference parity
5. **Training Pipelines** — how to trigger, config management, distributed setup, hardware
6. **Experiment Tracking & Model Registry** — tool, logged items, finding experiments, promotion
7. **Evaluation & Metrics** — primary/secondary metrics, validation strategy, benchmarks
8. **Serving & Inference** — deployment target, batch vs real-time, I/O contract, loading pattern
9. **Retraining & Continuous Learning** — triggers, E2E flow with paths, validation before promotion
10. **Monitoring & Drift Detection** — what is monitored, tool, alerting thresholds
11. **E2E ML Flows with Code Paths** — training flow, inference flow, retraining flow
12. **Tricky Parts & ML-Specific Nuances** — 8–15 items (train/serving skew, data leakage, class imbalance, etc.)
13. **Environment & Reproducibility** — Python version, key packages, how to reproduce, CUDA requirements
14. **Key File Quick Reference** — table of most important files

### D.6 — Merge and write

Apply per-section merge rules (structurally changed → replace; additive → merge; unchanged → keep; uncertain → keep + `<!-- verify -->`).

Never silently delete existing nuances.

Output file must begin with `<!-- Last updated: YYYY-MM-DD -->` then a cross-reference block.

### D Notes

- **Train/serving skew is the most common source of silent ML bugs.** Always investigate feature parity.
- For notebooks: read markdown cells and outputs — they contain the best explanation of intent.
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
```

### E.1 — Parse inputs

**Inline repos** (comma-separated): items starting with `https://github.com/` → GitHub URL; others → local path. Derive display name from directory name.

**Repos file** (`--repos-file <path>`): parse non-comment, non-blank lines as `<name>: <path-or-url>`.

Build list: `[{ name, source_type: "local"|"github", path_or_url }]`

### E.2 — Access each repo

**Local repos:** verify path exists and is a git repo.

**GitHub repos:**
1. Check `gh` CLI is available.
2. Shallow-clone: `gh repo clone <url> /tmp/ecosystem-overview/<name> -- --depth=1 --single-branch`

On failure: mark FAILED, continue with remaining repos. If all fail, stop.

### E.3 — Run Procedure: Explore Repo Interface (per repo)

Follow **Procedure 5: Explore Repo Interface** for each accessible repo. Run in parallel where possible.

### E.4 — Build the dependency graph

From interface profiles, resolve dependencies between repos:

**Matching strategy** (try in order):
1. Exact URL match
2. Topic name match
3. Queue name match
4. Endpoint path match
5. Named reference in config or README

For each resolved dependency record: direction, mechanism (REST/gRPC/Kafka/SQS/etc.), criticality (SYNC/ASYNC), label.

Unresolved → record as `EXTERNAL: <host/topic>` nodes.

### E.5 — Impact analysis

For each repo:

**Blast radius** (what breaks if this repo is down):
- SYNC dependents: fail immediately
- ASYNC dependents: stop receiving updates
- Transitive: A → B → C means C affected if B is down

**Severity classification:**
- `P0` — sync dependency, no fallback, on core path
- `P1` — sync dependency with circuit breaker / fallback
- `P2` — async dependency, consumer has DLQ / retry
- `P3` — async dependency, best-effort

### E.6 — Identify cross-repo E2E flows

Trace significant journeys crossing ≥2 repos. Notation:
```
[Actor] → [Repo A]: mechanism — action
         → [Repo B]: mechanism — action  (async)
                   → [Repo C]: mechanism — action
         ← [Repo A]: response to actor
```

### E.7 — Write `.atlas/ecosystem-overview.md`

Structure:
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
- REST edges: solid; Kafka/event edges: dashed; SQS: dotted
- EXTERNAL nodes: grey fill, dashed border

For Mermaid: produce **two diagrams** — full component diagram (`graph LR`) and critical path summary (one node per repo, P0/P1 edges highlighted).

If `--with-excalidraw`: also generate `.atlas/ecosystem.excalidraw`.

### E.9 — Cleanup

Remove temp directories: `rm -rf /tmp/ecosystem-overview/`

---

## Procedure 1: Detect Git Changes

Determine what changed since the last `.atlas/codebase-overview.md` update.

### Output

```
mode:             "current" | "targeted" | "full"
changed_files:    [list of file paths]  — empty if mode=current
affected_sections:[list of section names] — empty if mode=current or mode=full
commit_messages:  [list of commit subject lines]
reason:           human-readable explanation
```

### Step 1 — Extract last-updated date

**Prefer the index** (`.atlas/codebase-index.json`): read `meta.generated_at`.

**Fallback** to `.atlas/codebase-overview.md`: parse `<!-- Last updated: YYYY-MM-DD -->` from line 1.

If neither yields a parseable date → `mode: "full"`, reason: "Could not determine last-updated date".

### Step 2 — Get commits since that date

```bash
git log --since="LAST_UPDATED_DATE" --oneline
```

| Outcome | Result |
|---------|--------|
| `git` not available | `mode: "full"`, reason: "git not available" |
| Not a git repo | `mode: "full"`, reason: "not a git repo" |
| Command fails | `mode: "full"`, reason: log the raw error |
| Zero commits | `mode: "current"`, reason: "No commits since DATE" |
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

### Step 5 — Map changed files to affected sections (targeted mode only)

**Prefer index**: for each file in `changed_files`, read its `feeds_sections` from `.atlas/codebase-index.json`. Collect union of all feeds_sections → `affected_sections`.

**Fallback heuristics** (no index):

| File pattern | Sections affected |
|---|---|
| `domain/`, `models/`, `entities/`, domain structs | `Domain Models`, `State Machine Reference`, `ID / Key System` |
| `handlers/`, `routes/`, `controllers/`, `openapi/*.yml` | `E2E Flows`, `Component Map`, `Key File Quick Reference` |
| `*repository*.go`, `*_store.py`, db config | `Data Layer`, `E2E Flows` |
| `lambdas/`, `functions/` | `Component Map`, `Repository Layout`, `E2E Flows` |
| `service/`, `usecases/` | `E2E Flows`, `Architecture Patterns` |
| `config/`, `.env*` | `Data Layer`, `Architecture Patterns` |
| `cmd/services/`, `cmd/workers/` | `Component Map`, `Repository Layout` |
| `*_test.*`, `*.spec.*` | `Testing Approach` |

### Notes

- When in doubt, use `full` mode — a false positive is far less harmful than a false negative.
- The index's `feeds_sections` is always preferred over heuristics.

---

## Procedure 2: Index Codebase

Build or update `.atlas/codebase-index.json` — the structured map of the codebase.

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
      "feeds_sections": ["Domain Models", "State Machine Reference"],
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

**`component_type` values:**

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

**`component.layer` values (diagram layout):**

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

| File pattern | `feeds_sections` to assign |
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
| Top-level directory (inferred from structure) | `Repository Layout`, `Component Map` |

### Mode A — Full index

**1. Get current git SHA:** `git rev-parse HEAD`. Record in `meta.git_sha`. Use `null` if not a git repo.

**2. Discover all significant files.** Skip: `vendor/`, `node_modules/`, `.git/`, `dist/`, `build/`, `_generated.go`, `*.pb.go`, `mock_*.go`, `__pycache__/`, binaries.

Group by type: source files, config (`*.yaml`, `*.yml`, `*.toml`), infra (`*.tf`, `docker-compose.yml`).

**3. Classify each file:**
- `component_type`: infer from path and content
- `symbols`: read the file, extract exported names (Go: `type Foo struct`, `func Foo(`; Python: `class Foo`, `def foo`; TypeScript: `export const/function/class/interface`)
- `feeds_sections`: apply the mapping table above
- `last_modified_sha`: `git log -1 --format="%H" -- <filepath>` (null if not in git)

**4. Build components:** group files into logical deployable units (REST API server, consumer lambda, data store, etc.). Each component: `id` (URL-slug), `name`, `type`, `layer`, `files[]`, `deps[]`.

**5. Extract entities:** for each `domain_model` file, extract entity `name`, `id_field`, `states[]`.

**6. Extract ID types:** find identifier fields (`*ID`, `*Id`, `*_id`, `*Key`). Record `name`, `format`, `creator`, `used_in[]`.

**7. Extract flows:** for each handler/entry-point, trace the call chain 2–3 levels deep. Record `name`, `entry`, `steps[]`.

**8. Build `section_file_map`:** invert per-file `feeds_sections` into section → file list map. Deduplicate.

**9. Detect architecture:** from imports/config/manifests, detect languages, frameworks, databases, queues, external services.

**10. Write `.atlas/codebase-index.json`.** Create `.atlas/` if needed.

### Mode B — Targeted update

Input: comma-separated list of changed file paths.

1. Read existing `.atlas/codebase-index.json`.
2. For each changed file:
   - **Deleted**: remove from `files[]`, remove from `component.files[]`, rebuild deps.
   - **New**: classify and add to `files[]`, assign to or create a component.
   - **Modified**: re-read, update `symbols`, `feeds_sections`, `last_modified_sha`, update affected entities/flows.
3. Rebuild `section_file_map` from full updated `files[]`.
4. Update `meta.generated_at` and `meta.git_sha`.
5. Write the updated index.

Do not re-read files not in the changed list.

### Notes

- Correctness over completeness — 80% accurate beats 100% guessed.
- Do not deep-index test files: mark `component_type: "test"`, skip symbol extraction.
- `section_file_map` is the highest-value output — invest care in `feeds_sections` accuracy.

---

## Procedure 3: Write Overview Doc

Read `.atlas/codebase-index.json` and produce `.atlas/codebase-overview.md`.

**Prerequisite:** `.atlas/codebase-index.json` must exist. If not, stop: "Run Procedure 2: Index Codebase first."

### Step 1 — Load the index

Read `.atlas/codebase-index.json`. Determines: `section_file_map`, component inventory, entities, ID types, flows, language.

### Step 2 — Load existing overview (for merge)

If `.atlas/codebase-overview.md` exists AND `--fresh` not passed: read it in full, parse `Last updated` date. In targeted mode, sections NOT in `--sections` will be kept verbatim.

If `--fresh`: ignore existing file.

### Step 3 — Read source files per section

Use `section_file_map` to know which files to read for each section being generated. **Do not read files not listed** in the relevant section's map entry.

For sections with empty map entries (Repository Layout, Tricky Parts, Key File Quick Reference): derive from the index structure itself.

### Step 4 — Generate content

**Full mode**: produce all 11 required sections (see below).

**Targeted mode** (`--sections` provided): produce only those sections; carry all others verbatim.

Always use concrete `file.go:FunctionName()` references.

### Step 5 — Merge

| Situation | Action |
|---|---|
| Structurally changed | Replace with new content |
| Additive only | Merge in; keep existing accurate items |
| Unchanged | Keep existing wording |
| Uncertain | Keep content, append `<!-- verify -->` |

Never silently delete existing nuances or gotchas.

### Step 6 — Write the final file

File must begin with `<!-- Last updated: YYYY-MM-DD -->` (today's date), then a cross-reference block linking to companion docs, then the full content.

### Required Sections (11)

**1. Repository Layout** — annotated directory tree from index `files[]` and structure.

**2. Component Map** — table: component, type, purpose, entry point. Derive from `components[]`.

| Component | Type | Purpose | Entry Point |
|---|---|---|---|

**3. Domain Models Deep Dive** — field-level breakdown: key fields, status enums, lifecycle, constraints. Include all states and their behavioural meaning.

**4. ID / Key System** — comparison table of every identifier type. Most valuable section for new engineers.

| ID Type | Format | Who Creates It | Where Used |
|---|---|---|---|

**5. E2E Flows with Code Paths** — every significant journey traced with `file.go:Fn()` references. Always trace async continuations. Format:
```
1. Partner → POST /v1/deliveries
2. handler.go:CreateDelivery → validates input
3. service.go:Create → applies business rules
4. (async) stream → lambda.go:ProcessNew → publishes event
```

**6. Architecture Patterns** — recurring patterns (event-driven, CQRS, saga, etc.). For each: name it, explain why it's used, reference the file.

**7. Data Layer** — tables/collections, indexes, consistency model, caching, access patterns.

**8. Tricky Parts & Nuances** — 8–15 non-obvious behaviours. Always explain the *why*, not just the *what*.

**9. State Machine Reference** — ASCII state transition diagrams for key entities. List all valid transitions and triggers.

**10. Testing Approach** — unit vs integration vs E2E, how to run each, test patterns (doubles, fakes, fixtures).

**11. Key File Quick Reference** — table of most important files with single-line purpose.

### Notes

- E2E flows must include async continuations.
- The ID/Key System section is highest-value for new engineers and where subtle bugs hide.
- The doc serves two audiences: someone new to the codebase AND someone debugging a production incident.
- In targeted mode: do NOT regenerate sections not in `--sections`.

---

## Procedure 4: Generate Diagram

Produce architecture diagram files from available codebase knowledge.

### Input Priority

Read in order, stop at first that exists:
1. `.atlas/codebase-index.json` — best
2. `.atlas/codebase-overview.md` — components in prose
3. `.atlas/ml-overview.md` — supplement if present
4. Direct exploration — entry points, clients, DB config, infra

### Step 1 — Build Component Inventory

**Nodes:**
- Internal services / lambdas / workers
- Databases / data stores
- Message queues / topics
- External third-party services
- Actors / clients

**Edges:**
- Direction: A → B
- Protocol: REST, gRPC, Kafka, SQS, DynamoDB stream, WebSocket, etc.
- Label: short description

When reading from index: nodes ← `components[]`, edges ← `components[].deps`, groups ← `layer` field, external ← `architecture.external_services`.

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

Output to `.atlas/architecture.drawio`.

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
<mxCell id="EDGE_ID" value="label" style="edgeStyle=orthogonalEdgeStyle;rounded=0;" edge="1" source="SOURCE_ID" target="TARGET_ID" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

### Step 4 — Generate Excalidraw JSON *(only if `--format excalidraw` or `--format both`)*

Output to `.atlas/architecture.excalidraw`. Root structure (v2):
```json
{
  "type": "excalidraw", "version": 2, "source": "atlas",
  "elements": [], "appState": { "viewBackgroundColor": "#ffffff", "gridSize": 20 }, "files": {}
}
```

Fill colours: API `#a5d8ff`, Lambda/worker `#b2f2bb`, Database `#ffc9c9`, Queue `#ffec99`, External `#e9ecef`, Actor `#dee2e6`.

### Step 5 — Generate Mermaid Preview

Always generate regardless of format flags. Embed in `.atlas/architecture-diagram.md`.

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
> Auto-generated. Open `.atlas/architecture.drawio` in diagrams.net.
> Run atlas codebase-overview to regenerate source docs.

```mermaid
graph LR
    ...
```
```

### Step 6 — Merge with existing files

If output file exists: detect new components/edges and add them. Preserve hand-tuned coordinates (if existing node coords differ >50px from default grid, they were manually adjusted — keep them). Update `Last updated` date.

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

- Always generate the Mermaid preview — it's the fastest win for GitHub rendering.
- Diagrams show components and connections — not internal functions or classes.
- If >30 nodes: produce two diagrams — (1) full component view, (2) simplified critical-path view.

---

## Procedure 5: Explore Repo Interface

Extract a repository's cross-repo interface profile: what it exposes, depends on, and owns.

### Output Format

```
REPO: <display-name>
PATH: <local path>

EXPOSES:
  REST/GraphQL/gRPC endpoints:
    - METHOD /path — brief purpose
  Kafka / Pub-Sub topics PUBLISHED:
    - topic-name (message type) — what triggers publication
  SQS / event queues PUBLISHED:
    - queue-name — trigger
  Webhooks sent:
    - target type, payload shape, trigger
  Shared databases (if accessible to others):
    - table/collection name, access pattern

DEPENDS ON:
  HTTP/gRPC calls TO other services:
    - Target: <service name or URL>, endpoint, purpose, SYNC/ASYNC
  Kafka / Pub-Sub topics CONSUMED:
    - topic-name — what it does with messages
  SQS / event queues CONSUMED:
    - queue-name — processing logic
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

Check for `.atlas/codebase-index.json`:
- `EXPOSES` ← components with `type: "api"` and `layer: 1`
- `DEPENDS ON` ← components with `type: "external"` or `layer: 4`; `architecture.external_services`
- `OWNS` ← `architecture.databases` and `architecture.queues`
- `TECH PROFILE` ← `architecture.languages`, `architecture.frameworks`, `architecture.deployment`

At most 2–3 targeted file reads to fill gaps.

If no index but `.atlas/codebase-overview.md` exists: read it — Component Map, Data Layer, Architecture Patterns sections answer most questions.

### Step 2 — Deep exploration (no index, no overview)

Find endpoints: route registration files (`router.go`, `routes.py`, `app.ts`), handler files, OpenAPI specs, `.proto` files.

Find outbound calls: HTTP client instantiation (`http.NewRequest`, `axios`, `fetch`, `requests.get`, `grpc.Dial`), client wrapper files, config with base URLs.

Find Kafka/SQS topics: publishing (`Publish(`, `SendMessage(`, `producer.send(`), consuming (`Subscribe(`, `ReceiveMessage(`, consumer group config).

Find data stores owned: infrastructure files (`*.tf`, `serverless.yml`), database migration files, README.

Find auth: middleware files (`auth.go`, `middleware/`), JWT validation, OAuth2, API key headers, mTLS config.

### Step 3 — Classify dependencies

- **SYNC** if the caller blocks waiting for a response
- **ASYNC** if the caller does not wait (Kafka, SQS, DynamoDB stream)

### Step 4 — Return the profile

Output the structured profile. Use `<!-- unconfirmed -->` for relationships that seem likely but aren't explicitly visible.

### Notes

- Prefer existing docs over re-exploration.
- **SYNC vs ASYNC is the most critical field** — a SYNC dependency going down takes callers down.
- Scope is cross-repo surface only — internal architecture is out of scope here.
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

If an argument was provided, treat it as either:
- An absolute path (use as-is)
- A repo name — search for it as a subdirectory of the current working directory

If no argument was provided, use the current working directory as `TARGET_REPO`.

### F.2 — Discover sibling transport/proto repos

From the parent directory of `TARGET_REPO`, list all sibling directories. Identify those likely to contain shared transport models or protobuf definitions:
- Directory names containing: `transport`, `proto`, `grpc`, `models`, `schema`, `contracts`, `events`
- Files matching: `**/*.proto`, `**/transport_models*`, `**/grpc_models*`
- Go modules or package names that are imported by `TARGET_REPO`

Scan the top-level `go.mod`, `package.json`, `Cargo.toml`, `Gemfile`, or `requirements.txt` of `TARGET_REPO` for shared model imports and cross-reference with sibling directories.

Call the discovered list `TRANSPORT_REPOS`.

### F.3 — Explore TARGET_REPO deeply

For each of the following, read actual source files (not just directory listings):

**Language & structure:** detect language(s), read `README.md` and any existing architecture docs, map the top-level directory layout.

**Domain models** — look in: `internal/domain/`, `app/models/`, `src/models/`, `lib/`, `domain/`, `pkg/`, `crates/`

For each model/struct/class found:
- All fields with types
- Relationships to other models (foreign keys, embedded structs, associations)
- Status enums or state machine fields
- Key methods or business logic on the model
- DRN/UUID/ID fields that cross service boundaries

**State machines:** find status enums and allowed transitions via `switch/case` blocks or state machine libraries. Note any special override transitions (e.g., `COMPLETED → FAILED` race-condition patterns).

**Data persistence:** what databases (DynamoDB, Postgres, Redis, MySQL, MongoDB), table/collection schemas and key structures (partition keys, sort keys, GSIs), caching patterns.

**Event/message flows:** Kafka topics consumed and produced, SQS queues, SNS topics, webhooks, DynamoDB streams, gRPC services.

**External integrations:** other services called, auth patterns.

**API surface:** REST endpoints (path, method, request/response shape), gRPC methods, WebSocket endpoints.

### F.4 — Explore TRANSPORT_REPOS

For each repo in `TRANSPORT_REPOS`:
- Read `.proto` files, Go structs, or schema files relevant to `TARGET_REPO`
- Focus on message types that `TARGET_REPO` produces or consumes
- Note the field names and types that cross the service boundary

### F.5 — Write docs/domain-models.md

Create the file at `TARGET_REPO/docs/domain-models.md`. Include only sections that have content — skip empty sections rather than writing placeholders.

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

Do not write placeholder sections. If a section has no content, omit it entirely.
