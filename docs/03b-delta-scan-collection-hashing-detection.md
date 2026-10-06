# 03b Delta Scan Collection, Hashing & Detection

## Table of Contents
1. [TL;DR](#tldr)
2. [Resource Collection Fetchers](#resource-collection-fetchers)
3. [Normalization & Hashing Algorithm](#normalization--hashing-algorithm)
4. [Change Detection Logic (New/Mod/Del)](#change-detection-logic-newmoddel)
5. [State Storage & Post-Detection Pipeline](#state-storage--post-detection-pipeline)
6. [Edge Cases & Error Handling](#edge-cases--error-handling)
7. [Worked Example](#worked-example)

## TL;DR
- Resources are collected using **service-specific Boto3 fetchers** (e.g. `get_bucket_policy`).
- Configurations are **recursively normalized** (sorting dictionary keys and lists) and hashed using **SHA-256** to ensure deterministic comparison.
- Delta is computed using set math (`current_arns - previous_arns`) on tags, augmented by AWS Config drift, and strictly validated via configuration hashing.

## Resource Collection Fetchers

The `worker.jobs.inventory.fetchers.FETCHER_MAP` dictates which fetcher handles which CloudFormation-style ARN type.

| Resource Type | Handled By | Example AWS API Calls Used |
|---------------|------------|----------------------------|
| `AWS::S3::Bucket` | `fetch_s3_config` | `s3.get_public_access_block`, `s3.get_bucket_policy`, `s3.get_bucket_versioning`, `s3.get_bucket_encryption` |
| `AWS::EC2::Instance` | `fetch_ec2_config` | `ec2.describe_instances` |
| `AWS::EC2::SecurityGroup` | `fetch_sg_config` | `ec2.describe_security_groups` |
| `AWS::IAM::Role` / `User` | `fetch_iam_config` | `iam.get_role`, `iam.get_user`, `iam.list_attached_role_policies` |
| `AWS::RDS::DBInstance` | `fetch_rds_config` | `rds.describe_db_instances` |
| `AWS::KMS::Key` | `fetch_kms_config` | `kms.describe_key`, `kms.get_key_policy` |
| `AWS::CloudTrail::Trail` | `fetch_cloudtrail_config`| `cloudtrail.describe_trails`, `cloudtrail.get_trail_status`, `cloudtrail.get_event_selectors` |

## Normalization & Hashing Algorithm

To avoid false positive drift due to API array reordering or key shuffling, configurations are normalized before hashing (`worker/jobs/inventory/hashing.py`).

**Algorithm Steps:**
1. **Recursion:** Traverse the JSON dictionary.
2. **Dicts:** Sort all dictionary keys (`{k: _normalise(v) for k, v in sorted(config.items())}`).
3. **Lists:** Sort all lists using their JSON string representation as the sort key.
4. **Hashing:** The normalized structure is converted to a string using `json.dumps(..., sort_keys=True, default=str)` and hashed via `hashlib.sha256().hexdigest()`.

**Bytes hashed:** Only the normalized configuration dictionary string representation is hashed for `config_hash`. A separate `tag_hash` hashes the `{arn, type, region, tags}` block.

## Change Detection Logic (New/Mod/Del)

Change detection happens in `compute_delta` (`worker/jobs/inventory/aws.py:L258-274`).
- **CREATED (`new_arns`):** `current_set - previous_set` (ARNs that exist in Resource Explorer but not in local PostgreSQL state).
- **DELETED (`deleted_arns`):** `previous_set - current_set` (ARNs in local state no longer returned by AWS).
- **MODIFIED (`changed_arns`):** 
  - *Phase 1 (Tags):* ARNs in both sets where `current_tag_hash != previous_tag_hash`.
  - *Phase 2 (Config Drift):* ARNs flagged by AWS Config queries (`collect_changed_arns_from_config`).
  - *Phase 3 (Strict Hash Check):* For every ARN passing Phase 1 or 2, a full API fetch is performed. The final drift decision is `new_config_hash != old_config_hash` (`has_config_drifted`). Only resources passing this strict check are processed further.

## State Storage & Post-Detection Pipeline

- **Storage:** Hash states are loaded from and saved to the PostgreSQL `api_resourcestatehash` table via `state.py`.
- **Pipeline:** If a resource is deemed CREATED or MODIFIED (has drift):
  1. **Neo4j:** Written to graph via `write_resource_to_neo4j`.
  2. **PostgreSQL:** Saved to `api_resourceconfig` via `upsert_resource_config`.
  3. **OPA Evaluation:** `evaluate_resource` assesses the new config against Rego policies and updates findings.

## Edge Cases & Error Handling

- **Missing Fetcher (`skipped_no_fetcher`):** If an ARN is discovered but no fetcher is defined in `FETCHER_MAP`, it is skipped. The previous hash is carried forward to prevent false deletions or constant retries.
- **Expected API Exceptions:** Operations like `s3.get_bucket_policy` raise `NoSuchBucketPolicy` if no policy exists. These are caught locally in the fetcher and mapped to `None` rather than failing the task.
- **Throttling:** Fetchers are wrapped in a `@with_backoff` decorator, and the global Boto3 config employs `mode: adaptive` with `max_attempts: 8` to handle rate limits gracefully.
- **S3 Discovery Fallback:** Resource Explorer sometimes lags on S3 bucket indexing; an optional fallback directly polls S3 (`_poll_s3_buckets`).

## Worked Example

### Scenario: S3 Bucket Versioning is Enabled

**Run 1 (Before):**
- **Fetched Config:** 
  ```json
  {
    "bucket_name": "prod-data-bucket",
    "versioning": "Disabled",
    "public_access_block": null,
    "policy": null
  }
  ```
- **Normalized JSON String:** `'{"bucket_name": "prod-data-bucket", "policy": null, "public_access_block": null, "versioning": "Disabled"}'`
- **Computed `config_hash` (Example):** `a1b2c3d4e5f6...`
- **Result:** Resource evaluated, hash saved.

*User enables versioning via AWS Console. AWS Config captures the change.*

**Run 2 (After):**
- **Delta Trigger:** AWS Config identifies the ARN as drifted since `last_polled_at`.
- **Fetched Config:** 
  ```json
  {
    "bucket_name": "prod-data-bucket",
    "versioning": "Enabled",
    "public_access_block": null,
    "policy": null
  }
  ```
- **Normalized JSON String:** `'{"bucket_name": "prod-data-bucket", "policy": null, "public_access_block": null, "versioning": "Enabled"}'`
- **Computed `config_hash` (Example):** `f9e8d7c6b5a4...`
- **Detection Check:** `f9e8d7c6b5a4... != a1b2c3d4e5f6...` (Drift detected = `True`).
- **Result:** New configuration is saved to PostgreSQL, written to Neo4j, and pushed to OPA for rule evaluation.
