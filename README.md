# AWS Security Misconfiguration Scanner

A Python security auditing tool for identifying common misconfigurations across AWS IAM, S3, and EC2 security groups.

The scanner combines three audit modules into a single workflow, assigns severity ratings, and generates a JSON report with findings and remediation recommendations. Security group checks run across enabled AWS regions.

## Capabilities

| Area | Implemented checks |
| --- | --- |
| IAM | Users without configured MFA devices, access keys older than 90 days, and console passwords last used more than 90 days ago |
| S3 | Missing Block Public Access settings, missing default encryption configuration, disabled versioning, disabled access logging, and allow statements with wildcard principals |
| EC2 security groups | Inbound rules allowing `0.0.0.0/0` or `::/0`, including SSH, RDP, database ports, large port ranges, and all-traffic rules |
| Reporting | Consolidated JSON findings, severity counts, resource identifiers, and remediation recommendations |
| Validation | Automated tests and a GitHub Actions workflow for linting, syntax checks, and testing |

The scanner reads AWS configuration through Boto3. It does not automatically modify resources or remediate findings.

## Project Structure

| Path | Purpose |
| --- | --- |
| [scripts/iam_audit.py](scripts/iam_audit.py) | IAM user and credential checks |
| [scripts/s3_audit.py](scripts/s3_audit.py) | S3 bucket configuration and policy checks |
| [scripts/security_groups_audit.py](scripts/security_groups_audit.py) | Security group rule analysis across enabled regions |
| [scripts/full_audit.py](scripts/full_audit.py) | Audit orchestration, console summary, and JSON export |
| [tests/](tests/) | Unit tests and an orchestrator integration test |
| [.github/workflows/ci.yml](.github/workflows/ci.yml) | Continuous integration workflow |
| [docs/sample_audit_report.json](docs/sample_audit_report.json) | Example report structure |
| [CIS_MAPPING.md](CIS_MAPPING.md) | Project benchmark mapping notes |

## Getting Started

### Prerequisites

- Python 3.12, matching the version used in CI.
- AWS credentials available through the standard Boto3 credential provider chain, such as an authenticated AWS profile.
- Permission to read the IAM, S3, and EC2 configuration being audited.

The audit uses IAM user, MFA, and access-key read operations; S3 bucket listing and configuration reads; and EC2 region and security group discovery. Use a scoped read-only identity appropriate to the target account. No credentials belong in the repository.

### Installation

Clone the repository and install the dependencies in a virtual environment.

```bash
git clone https://github.com/vchoudhary480/AWS-Security-Misconfiguration-Scanner.git
cd AWS-Security-Misconfiguration-Scanner
python -m venv .venv
```

Activate the environment on Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

On macOS or Linux:

```bash
source .venv/bin/activate
```

Install the project dependencies:

```bash
python -m pip install -r requirements.txt
```

### Run the Full Audit

From the repository root, using your configured AWS identity:

```bash
python scripts/full_audit.py
```

The audit prints findings and a summary to the console, then writes a timestamped report to:

```text
docs/audit_report_YYYYMMDD_HHMMSS.json
```

Individual modules can also run independently:

```bash
python scripts/iam_audit.py
python scripts/s3_audit.py
python scripts/security_groups_audit.py
```

## Reporting & Exit Codes

Each finding includes a severity, category, affected resource, issue description, and remediation recommendation. The combined report includes a timestamp, total finding count, and severity breakdown.

| Exit code | Meaning |
| --- | --- |
| `0` | The full audit completed without recorded findings |
| `1` | Findings were recorded, or an unhandled failure occurred |

The full audit uses exit code `1` when findings exist. Python can also return this code for an unhandled error, so inspect the console output before treating it as a findings-only signal.

Some AWS API errors are printed without becoming findings. An empty report or exit code `0` therefore does not prove that every check completed successfully.

## Testing & Continuous Integration

The repository contains 18 test functions covering IAM checks, S3 checks, security group rule analysis, and combined reporting. AWS-dependent tests use Moto to mock service responses; rule-analysis tests also use direct input dictionaries.

Run the suite from the repository root:

```bash
python -m pytest tests/ -v --cov=scripts --cov-report=term-missing
```

The GitHub Actions workflow runs on pushes and pull requests targeting `main`. It installs the pinned dependencies, runs Ruff, checks script syntax, and executes the tests.

```bash
ruff check scripts/ tests/
```

Tests validate the covered scenarios. They do not establish complete detection coverage or a zero false-positive rate.

## Sample Output

### Security Group Findings

![Console findings for intentionally misconfigured security groups](screenshots/01_vulnerable_sg_findings.png)

### Full Audit Summary

![Consolidated AWS audit summary with severity counts](screenshots/02_full_audit_summary.png)

### JSON Report

![Example JSON audit report](screenshots/03_audit_report_json.png)

## Scope & Limitations

- **Public CIDRs:** Security group checks recognize `0.0.0.0/0` and `::/0`. Other broad or publicly routable CIDRs are not classified as public by the current implementation.
- **IAM coverage:** Checks target IAM users and credentials. They do not audit root-account security, effective permissions, or privilege-escalation paths such as `iam:PassRole`.
- **S3 policy evaluation:** Wildcard-principal allow statements are flagged without a complete evaluation of conditions, explicit denies, or effective access. Review findings in context.
- **Console inactivity:** The inactivity check relies on an available `PasswordLastUsed` value. Users without that value are not flagged by this check.
- **Audit completeness:** Coverage depends on API permissions, available responses, and implemented checks. The tool is a focused learning project, not a complete account-security assessment.

## Benchmark Mapping & Lab Context

[CIS_MAPPING.md](CIS_MAPPING.md) records the project's AWS Foundations Benchmark mapping notes, including manual configuration work and out-of-scope controls. Some mappings require review against the implementation. The document does not establish full benchmark coverage or compliance.

The original account-hardening lab also documented CloudTrail and AWS Config setup. The automated scanner described here covers IAM, S3, and EC2 security groups; it does not audit those logging services.

## Intended Use

Use the scanner to learn AWS security APIs, investigate selected configuration risks, and develop custom audit checks. Review findings before applying remediation and use broader assessment tools when comprehensive coverage is required.
