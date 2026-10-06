# Documentation Progress

## Steps Done
- [x] Step 0: Initial Discovery & Progress File Creation
- [x] Step 1: `docs/01-architecture-overview.md`
- [x] Step 2: `docs/02-data-flow-diagrams.md`, part 1
- [x] Step 3: `docs/02-data-flow-diagrams.md`, part 2
- [x] Step 4: `docs/03a-delta-scan-triggers-and-cursor.md`
- [x] Step 5: `docs/03b-delta-scan-collection-hashing-detection.md`
- [x] Step 6: `docs/04-policy-engine-and-opa.md`
- [x] Step 7: `docs/05-deep-scan-and-graph.md`

## Next Step
- ALL TASKS COMPLETE.

## Key Facts Found (Map)
* **Top-level Structure:** `backend-fastapi` (core service), `frontend`, `Cloudsecure_cli`, `IaC_backend`, `scripts`, `docs`
* **Languages:** Python (FastAPI backend), likely JS/TS (frontend)
* **Dependencies:** Python (`requirements-core.txt`, `requirements-deep.txt`, `requirements-cartography-aws.txt`)
* **Services/Queues:** Celery (`backend-fastapi/worker/celery_app.py`), FastAPI (`backend-fastapi/app/`), Neo4j DB
* **Delta Scan:** `backend-fastapi/worker/jobs/inventory/` (contains `aws.py`, `config_drift.py`)
* **Deep Scan:** `backend-fastapi/worker/jobs/deep_scan/` (contains `scan.py`, Cartography references)
* **OPA/Rego:** `backend-fastapi/worker/jobs/rule_engine/`
* **Neo4j / DB Utils:** `backend-fastapi/worker/jobs/inventory/neo4j_writer.py`, `backend-fastapi/worker/jobs/deep_scan/db_utils.py`
* **Hashing:** `backend-fastapi/worker/jobs/inventory/hashing.py`
* **Cursor/Checkpoint:** Present in `backend-fastapi/worker/jobs/inventory/aws.py`, `config_drift.py`, and `deep_scan` modules.
* **DB Schemas:** `backend-fastapi/app/models/orm.py`, `backend-fastapi/app/schemas/`

## Open Questions
- What specific graph data science (GDS) features are active in Neo4j?
- Are there specific AWS services used for event triggering outside of Celery (e.g. EventBridge)?
