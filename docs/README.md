# CloudSecure Documentation Index

## Overview
Welcome to the internal documentation for CloudSecure, an open-source Cloud Security Posture Management (CSPM) platform for AWS.

## Files
1. [01-architecture-overview.md](./01-architecture-overview.md): System context, container, and infrastructure diagrams. Explains core actors and technologies.
2. [02-data-flow-diagrams.md](./02-data-flow-diagrams.md): Sequence diagrams and flowcharts for Delta Scan and Deep Scan operations.
3. [03-delta-scan-deep-dive.md](./03-delta-scan-deep-dive.md): In-depth look at how the inventory pull works, including AWS polling, SHA-256 hashing, and change detection.
4. [04-policy-engine-and-opa.md](./04-policy-engine-and-opa.md): Explains the use of Open Policy Agent (OPA) and Rego for evaluating cloud configurations against frameworks like CIS.
5. [05-deep-scan-neo4j-cartography-gds.md](./05-deep-scan-neo4j-cartography-gds.md): Covers the full Cartography ingestion pipeline and the usage of Neo4j Graph Data Science (GDS) algorithms to dynamically adjust finding severities.

## Glossary
- **CSPM**: Cloud Security Posture Management.
- **Cartography**: An open-source Python tool that consolidates infrastructure assets and relationships in Neo4j.
- **Cypher**: The query language used to interact with Neo4j.
- **Deep Scan**: A comprehensive scan that builds the relationship graph using Cartography.
- **Delta Scan / Inventory Pull**: A fast scan that uses hashing to identify configuration drift without pulling the entire environment.
- **Drift**: Changes in resource configuration detected between scans.
- **Finding**: A security violation resulting from policy evaluation.
- **GDS**: Graph Data Science, a Neo4j plugin providing network algorithms like PageRank and Betweenness.
- **OPA**: Open Policy Agent, a general-purpose policy engine.
- **Rego**: The declarative language used by OPA to write policies.

## Not Found in Codebase
The following technologies/features were evaluated but are **not found in the codebase**:
- **Event-driven scan triggers**: CloudWatch Events, EventBridge, CloudTrail triggers, SNS/SQS are not used to trigger scans. Scans rely entirely on scheduled Celery Beat polling (`periodic_inventory_pulls`). 
  - *Where this could fit*: Wiring AWS EventBridge to a webhook in FastAPI could significantly reduce scan latency.
- **Embedded OPA library / WASM**: OPA is strictly deployed as a standalone REST server, not embedded.

## Open Questions / Ambiguities
- **MFA Delete Field Fetching**: As noted in `cis_aws_s3.rego`, the `mfa_delete_enabled` field is not fully populated by the fetcher yet. This needs alignment between the Rego policy and the Python fetcher logic.
- **Missing Fetchers**: `worker/jobs/inventory/aws.py` gracefully skips resources with no fetcher defined. The current list of supported resources is limited to a subset of AWS (`S3`, `EC2`, `IAM`, `RDS`, `KMS`, `CloudTrail`).
