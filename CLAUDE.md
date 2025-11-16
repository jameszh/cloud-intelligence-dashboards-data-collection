# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloud Intelligence Dashboards - Data Collection is an AWS solution that collects cost, usage, and operational data from AWS Organizations to enable comprehensive dashboards via Amazon QuickSight. The repository contains CloudFormation templates that deploy serverless data collection infrastructure using Lambda, Step Functions, S3, Glue, and Athena.

## Repository Structure

The repository is organized into five main components:

- **data-exports/**: CloudFormation templates for AWS Data Exports (CUR 2.0) replication and aggregation across Management and Linked accounts
- **data-collection/**: CloudFormation templates for collecting operational data from AWS services (Trusted Advisor, Compute Optimizer, Organizations, etc.)
- **case-summarization/**: AI-powered AWS Support Case summarization using Amazon Bedrock
- **rls/**: Row Level Security management for Cloud Intelligence Dashboards
- **security-hub/**: AWS Security Hub data collection

## Development Commands

### Linting and Quality Checks

```bash
# Lint all CloudFormation templates (runs cfn-lint, checkov, cfn_nag_scan)
./utils/lint.sh

# Lint Python Lambda functions embedded in CloudFormation templates
python3 ./utils/pylint.py
```

### Testing

```bash
# Install test dependencies first
pip3 install -U boto3 pytest cfn-flip pylint bandit cfn-lint checkov

# Run integration tests (deploys stacks, validates, tears down)
./test/run-test-from-scratch.sh

# Run tests without teardown (useful for debugging)
./test/run-test-from-scratch.sh --no-teardown

# Run specific pytest tests
pytest test/
```

### Prerequisites for Development

- Python 3.9+
- AWS credentials configured (`~/.aws/credentials`)
- cfn_nag_scan installed
- Required Python packages: `boto3 pytest cfn-flip pylint bandit cfn-lint checkov`

## Architecture Patterns

### Data Collection Architecture

All data collection modules follow a consistent pattern:

1. **EventBridge** triggers Step Functions on a schedule (default: every 14 days)
2. **Account Collector Lambda** assumes role in Management account(s) and retrieves linked accounts via Organizations API
3. **Step Functions** fan out to invoke Data Collection Lambda for each account
4. **Data Collection Lambda** assumes role in each linked account, retrieves data via AWS SDK (boto3), stores results in S3
5. **Glue Crawler** runs after collection to update table schemas in Glue Data Catalog
6. **Athena** provides SQL query access to collected data for QuickSight dashboards

### CloudFormation Module Structure

Each data collection module (e.g., `module-organization.yaml`, `module-budgets.yaml`) contains:

- **Parameters**: Configuration for database name, S3 bucket, IAM roles, schedule, resource prefix
- **Lambda Function**: Inline Python code (ZipFile) or reference to code in S3
- **IAM Roles**: LambdaRole with permissions to assume cross-account roles and write to S3
- **Step Function**: Orchestrates account collection and data gathering
- **EventBridge Schedule**: Triggers Step Function on defined schedule
- **Glue Resources**: Crawler and table definitions for Athena queries

Some Lambda source code is located in `data-collection/deploy/source/` for larger or shared functions.

### Data Exports Architecture

Data Exports templates create S3 replication rules to:
1. Replicate Cost & Usage Reports (CUR2) from Management/Source accounts to Data Collection account
2. Filter out metadata to make structure Athena-compatible
3. Run Glue Crawlers to maintain partitions
4. Optional: Secondary replication for multi-org consolidation

## Module Types

Data collection modules are categorized by where they collect data:

- **Management Account**: organization, compute-optimizer, cost-explorer-cost-anomaly, backup, health-events, license-manager
- **Linked Accounts**: budgets, trusted-advisor, support-cases, inventory, rds-usage, transit-gateway, ecs-chargeback, resilience-hub, marketplace
- **Data Collection Account**: pricing, aws-feeds, quicksight, reference

## File Naming Conventions

- CloudFormation templates: `*.yaml`
- Module templates: `module-{service-name}.yaml`
- Deployment templates: `deploy-*.yaml`
- Lambda source: `data-collection/deploy/source/*.py`
- Utilities: `utils/*.py` or `utils/*.sh`

## Important Considerations

### CloudFormation Linting Exclusions

Some templates are excluded from cfn_nag_scan because they use `Fn::ForEach` which breaks the linter:
- `module-inventory.yaml`
- `module-pricing.yaml`
- `module-backup.yaml`

### Lambda Inline Code

Most Lambda functions are embedded as ZipFile in CloudFormation templates. When modifying:
1. Edit the ZipFile content in the YAML template
2. Run `python3 ./utils/pylint.py` to validate syntax and style
3. Disable specific pylint rules as needed (see utils/pylint.py for current disabled rules)

### Cross-Account IAM Roles

All modules assume cross-account roles with naming conventions:
- Management account role: Specified via `ManagementRoleName` parameter
- Linked account roles: Deployed via StackSets from Management account permissions stack

When adding new data collection, ensure proper IAM permissions are granted in both the Lambda execution role and the cross-account assumed role.

### S3 Storage Considerations

- Do NOT use S3 Intelligent Tiering or Infrequent Access for CUR data connected to Athena
- Athena's parallel query architecture can result in massive retrieval costs (up to 75x data read multiplier)
- The KPI Dashboard scans entire CUR history, preventing effective tiering

### Testing Requirements

For comprehensive testing, the test account should have:
- EC2 instances running 14+ days (for Compute Optimizer)
- EBS volumes and snapshots
- Custom AMIs
- Enterprise Support (for Trusted Advisor module)
- RDS cluster/instance
- ECS cluster with service
- Transit Gateway with attachments
- AWS Organization with trusted access enabled

## Git Workflow

1. Work against the `main` branch
2. Run lint checks before committing: `./utils/lint.sh && python3 ./utils/pylint.py`
3. Ensure integration tests pass: `./test/run-test-from-scratch.sh`
4. Create pull requests for review
