# Discovery Fact Sheet

## Services, Queues, Tables, and Topics
- **PostgreSQL**: Stores relational data (`users`, `providers`, `inventory_runs`, `findings`, `resource_configs`, `config_cursors`).
- **Neo4j**: Stores graph topology of cloud resources and relationships.
- **Valkey / Redis**: Acts as Celery broker and result backend. Stores delta scan state (hashes) under `cloudsecure:inventory:scheduled` and per-account hashes.
- **Celery Worker**: Queues: `inventory` (Delta Scans), `deep_scan` (Cartography), `anomaly` (CloudTrail anomaly detection).
- **Celery Beat**: Schedules periodic tasks.
- **OPA**: Open Policy Agent server.

## Dependencies (Key)
- `fastapi==0.115.6`, `pydantic==2.10.6`, `sqlalchemy==2.0.36`, `psycopg==3.2.3`, `celery==5.4.0`, `redis==5.2.1`, `boto3==1.36.1`, `neo4j==6.0.3` (from `requirements-core.txt`)
- `cartography` (from `requirements-cartography-aws.txt`)
- `react^18.2.0`, `vite^5.0.0`, `tailwindcss^3.4.0` (from `frontend/package.json`)

## Scheduled Jobs
- `periodic-inventory-pulls`: Runs `worker.tasks.periodic_inventory_pulls` every 30 minutes (schedules `perform_inventory_pull_task`).

## Environment Variables / Config Keys
- `postgres_db`, `postgres_user`, `postgres_password`, `postgres_host`, `postgres_port`
- `valkey_url`
- `neo4j_uri`, `neo4j_user`, `neo4j_password` (and shared/tenant variations)
- `opa_url`, `rules_dir`
- `aws_default_region`, `enable_aws_config_drift`, `aws_config_region`, `aws_config_initial_lookback_minutes`
- `enable_s3_fallback_discovery`, `consolidated_s3_rules_enabled`, `suppress_fragmented_s3_public_rules`
- `cartography_permission_relationships_file`
- `secret_key`, `debug`, `cors_origins`

## Cloud API Calls (Key)
- **AWS Resource Explorer 2**: `GetIndex`, `GetDefaultView`, `ListViews`, `Search`
- **AWS Config**: `SelectResourceConfig`
- **S3 (Fallback)**: `ListBuckets`, `GetBucketLocation`, `GetBucketTagging`
- **Cartography Deep Scan**: `Describe*`, `List*`, `Get*` APIs across IAM, EC2, S3, RDS, KMS, Lambda, ECR, Organizations.

## Rego Packages
- `cloudsecure.rules.unified_contextual`
- `cloudsecure.rules.india_aws_s3`, `india_aws_rds`, `india_aws_kms`, `india_aws_iam`, `india_aws_ec2`, `india_aws_cloudtrail`
- `cloudsecure.rules.cis_aws_s3`, `cis_aws_rds`, `cis_aws_kms`, `cis_aws_iam`, `cis_aws_ec2`, `cis_aws_cloudtrail`
- `cloudsecure.rules.consolidated_s3`

## Cypher Queries (Graph Analytics)
- GDS Projections: `CALL gds.graph.project.cypher(...)`
- Algorithms: `gds.pageRank.write`, `gds.betweenness.write`, `gds.louvain.write`, `gds.kcore.write`, `gds.wcc.write`, `gds.shortestPath.dijkstra.stream`
- Shadow Scoring: `SET n.violation_similarity_score = ...` based on PageRank and Internet Hops.
