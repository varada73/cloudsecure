# 05 Deep Scan & Graph Analytics

## Table of Contents
1. [TL;DR](#tldr)
2. [Cartography Ingestion](#cartography-ingestion)
3. [Data Dictionary (Nodes & Properties)](#data-dictionary-nodes--properties)
4. [Overlaying Findings](#overlaying-findings)
5. [Internet Exposure & Cypher Queries](#internet-exposure--cypher-queries)
6. [Graph Data Science (GDS) Analytics](#graph-data-science-gds-analytics)
7. [API Endpoints](#api-endpoints)

## TL;DR
- **Deep Scan** utilizes Lyft's Cartography to perform exhaustive asset and relationship mapping into Neo4j.
- Findings from the PostgreSQL policy engine are synced into Neo4j and attached to resource nodes via `HAS_FINDING` edges.
- Cypher queries dynamically link resources exposed to the internet.
- **Neo4j Graph Data Science (GDS)** algorithms (PageRank, Betweenness) calculate blast radius and choke points, automatically elevating the severity of findings on high-risk nodes.

## Cartography Ingestion

The Deep Scan process (`worker/jobs/deep_scan/scan.py`) orchestrates the graph build. 
For AWS, `worker/jobs/deep_scan/aws/aws.py` serves as the wrapper around `cartography.intel.aws`. 
- It configures Boto3 sessions via STS AssumeRole.
- It sequentially calls `cartography_aws.organizations.sync`, `_autodiscover_accounts`, and specific service synchronizers based on Cartography's `RESOURCE_FUNCTIONS`.
- All created nodes receive a common set of metadata: `firstseen`, `lastupdated` (matching the scan's `update_tag`), and `_module_name = 'cloudsecure:deep_scan'`.

## Data Dictionary (Nodes & Properties)

| Label | Role / Origin | Key Properties |
|-------|---------------|----------------|
| `AWSAccount` | Root node for AWS providers | `id` (Account ID), `name` |
| `AWSS3Bucket`, `EC2Instance` | Standard Cartography resource nodes | `id`, `arn`, `exposed_internet` |
| `CloudSecureFinding` | Represents a policy violation from PostgreSQL | `id` (UUID), `status`, `severity`, `check_id` |
| `InternetAccess` | Synthetic node representing the public internet | `id: 'Internet'` |

**Standard Relationship Types:**
- `CAN_ACCESS`: Represents network reachability (e.g., Internet -> Resource, or Resource -> Resource).
- `HAS_FINDING`: Links a resource node to a `CloudSecureFinding` node.

## Overlaying Findings

Because policy evaluations (Delta Scans) run independently of Cartography (Deep Scans), their outputs must be merged.
1. `worker/jobs/deep_scan/findings.py` extracts all `OPEN` findings from PostgreSQL for the provider.
2. It executes `INSERT_FINDING_TEMPLATE` (defined in `queries.py`).
3. This Cypher query finds the corresponding resource node by `arn` or `id`, creates a `CloudSecureFinding` node, and bridges them with a `HAS_FINDING` relationship.
4. Finally, `CLEANUP_FINDINGS_TEMPLATE` deletes any finding nodes in Neo4j whose `lastupdated` timestamp is older than the current `update_tag`, effectively removing resolved findings from the graph.

## Internet Exposure & Cypher Queries

To visualize and calculate attack paths from the outside in, the system synthesizes an `InternetAccess` node.
- **Node Creation:** 
  ```cypher
  MERGE (internet:InternetAccess {id: 'Internet'})
  ```
- **Edge Creation (`CREATE_INTERNET_ACCESS_EDGES_TEMPLATE`):**
  ```cypher
  MATCH (account:AWSAccount {id: $provider_uid})-->(resource)
  WHERE resource.exposed_internet = true
  MATCH (internet:InternetAccess {id: 'Internet'})
  MERGE (internet)-[r:CAN_ACCESS]->(resource)
  ```
  *(Cartography natively sets `exposed_internet = true` on certain nodes during ingestion).*

## Graph Data Science (GDS) Analytics

After nodes and edges are populated, `graph_analytics.py` executes Neo4j GDS algorithms on a temporary graph projection (`cloudsecure-gi-{provider_id}-{update_tag}`).

It computes several scores for every node:
- **`pagerank_score`:** Indicates centrality/importance.
- **`betweenness_score`:** Identifies choke points.
- **`hops_from_internet`:** Shortest path distance from the `InternetAccess` node.
- **`community_size` & `core_value`:** Identifies dense clusters/zones.

**Severity Adjustment:**
The system retrieves all `OPEN` findings and evaluates the GDS scores of the affected resource. The `compute_adjusted_severity` function elevates the finding's severity (e.g., `MEDIUM` -> `HIGH` -> `CRITICAL`) if the resource:
- Is ≤ 2 hops from the internet.
- Has high PageRank (>0.7).
- Is a choke point (Betweenness > 500).

## API Endpoints

The frontend queries this rich graph data via `app/api/routers/analytics.py`:
- `GET /api/v1/providers/{provider_id}/graph/`: Returns nodes and edges formatted for D3.js visualization.
- `GET /api/v1/providers/{provider_id}/graph/cartography/`: Raw structural graph data.
- `POST /api/v1/providers/{provider_id}/attack-engine/run/`: Executes on-demand attack path queries.
