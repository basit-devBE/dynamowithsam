# Orders DynamoDB Table — AWS SAM + SAM Pipelines + GitHub Actions

Deploys a DynamoDB `Orders` table to two independent environments (**dev** and **prod**),
each via its own GitHub Actions workflow and its own SAM Pipeline bootstrap resources.

## Table design (`template.yaml`)

| Element | Value |
|---|---|
| Partition key | `orderId` (String) |
| Additional attributes | `customerId` (String), `status` (String) |
| GSI 1 | `CustomerIdIndex` on `customerId` |
| GSI 2 | `StatusIndex` on `status` |
| Billing mode | `PAY_PER_REQUEST` (on-demand) |
| Table class | `STANDARD_INFREQUENT_ACCESS` (non-default) |

The table name is `Orders-<Environment>`, where `Environment` is a stack parameter
(`dev` or `prod`).

## Environments and pipelines

Each environment has its own:
- Dedicated S3 artifacts bucket (created by `sam pipeline bootstrap`, not shared)
- Dedicated CloudFormation execution role
- Dedicated pipeline execution role, trusted only for its own branch via GitHub OIDC
- Dedicated GitHub Actions workflow (no shared/auto-generated multi-stage workflow)

| Environment | Branch | Workflow | Stack name |
|---|---|---|---|
| dev | `develop` | [`deploy-dev.yml`](.github/workflows/deploy-dev.yml) | `orders-table-dev` |
| prod | `main` | [`deploy-prod.yml`](.github/workflows/deploy-prod.yml) | `orders-table-prod` |

Pushing to `develop` deploys only the dev stack; pushing to `main` deploys only the prod
stack. No IAM role, S3 bucket, or workflow is shared between the two.

## Bootstrapping (already done for this repo)

```bash
sam pipeline bootstrap --stage dev  --no-interactive --region eu-central-1 \
  --permissions-provider oidc --cicd-provider github-actions \
  --oidc-provider-url https://token.actions.githubusercontent.com \
  --oidc-client-id sts.amazonaws.com \
  --github-org basit-devBE --github-repo dynamowithsam --deployment-branch develop

sam pipeline bootstrap --stage prod --no-interactive --region eu-central-1 \
  --permissions-provider oidc --cicd-provider github-actions \
  --oidc-provider-url https://token.actions.githubusercontent.com \
  --oidc-client-id sts.amazonaws.com \
  --github-org basit-devBE --github-repo dynamowithsam --deployment-branch main
```

Both bootstraps reuse the account's existing GitHub OIDC provider
(`token.actions.githubusercontent.com`) instead of creating a duplicate.

## Verifying in the AWS Console

1. Open **DynamoDB → Tables** in `eu-central-1` and select `Orders-dev` or `Orders-prod`.
2. **Insert an item**: Explore table items → Create item → fill in `orderId`,
   `customerId`, `status` (e.g. `orderId=ord-001`, `customerId=cust-42`, `status=PLACED`) → Create.
3. **Query the base table**: Explore table items → Query → partition key `orderId`.
4. **Query a GSI**: Explore table items → Indexes tab → choose `CustomerIdIndex` or
   `StatusIndex` → Query on `customerId` / `status`.

## Local development

```bash
sam validate --lint
sam build
sam deploy --guided   # first-time manual deploy, if needed outside the pipeline
```
