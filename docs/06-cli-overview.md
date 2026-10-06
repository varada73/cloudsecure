# 06 CloudSecure CLI

## Table of Contents
1. [TL;DR](#tldr)
2. [CLI Architecture & Engine](#cli-architecture--engine)
3. [Scan Execution Flow](#scan-execution-flow)
4. [Reporting & Exports](#reporting--exports)
5. [Usage Example](#usage-example)

## TL;DR
- **CloudSecure CLI** is a local Python-based tool for auditing Infrastructure as Code (IaC) before deployment.
- It acts as an intelligent wrapper around **tfsec**, filtering and formatting its output into rich terminal tables, PDFs, and CSVs.
- Uses `click` for command routing and `rich` for terminal UI (progress bars, ASCII art, colored tables).

## CLI Architecture & Engine

**Location:** `Cloudsecure_cli/`
**Primary Entrypoint:** `cli.py`

The CLI is built using Python's `click` library and leverages `rich` to provide a premium developer experience (colored output, spinners, aligned tables, and a custom ASCII banner).

Instead of parsing Terraform HCL natively, the CLI delegates scanning to an external binary: **tfsec**.
- `tfsec` must be installed on the user's `$PATH`.
- The CLI invokes it via `subprocess.run(["tfsec", exec_path, "--format", "json", "--soft-fail", "--include-passed"])`.
- The `--soft-fail` flag guarantees that `tfsec` returns JSON output and a 0 exit code even if vulnerabilities are found, allowing the Python wrapper to gracefully parse the results.

## Scan Execution Flow

Located in `scanners/terraform.py:run_local_scan`:

1. **Path Resolution:** The CLI checks if the provided `scan_path` is a directory or a specific file.
2. **Execution:** It runs `tfsec` against the directory containing the IaC files.
3. **Parsing:** The JSON stdout from `tfsec` is parsed into a Python dictionary.
4. **Filtering (File-Level):** If the user specified a single file (e.g., `main.tf`), the CLI filters the `tfsec` JSON output to only include results where `location.filename` matches the target file.
5. **UI Rendering:** The results are mapped to a `rich.table.Table`. Severities are color-coded (CRITICAL = Red/White, HIGH = Red, MEDIUM = Yellow, LOW = Blue).

## Reporting & Exports

Located in `utils/reporting.py`:

The CLI supports exporting the parsed scan results for CI/CD pipelines or compliance documentation.
- **CSV Export:** Extracts `rule_id`, `severity`, `resource`, `description`, and `location` using Python's built-in `csv.DictWriter`.
- **PDF Export:** Uses `reportlab` to generate a styled PDF document with a title, timestamp, and a data table summarizing the vulnerabilities (descriptions are truncated to fit the page width).

## Usage Example

```bash
# Run a scan and print the Rich table to the console
cloudsecure iac scan ./terraform

# Run a scan and export results to PDF
cloudsecure iac scan ./terraform --pdf-export scan_report.pdf

# Check CLI version
cloudsecure version
```
