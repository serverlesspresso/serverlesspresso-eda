# Serverlesspresso — Event-Driven Architecture on AWS

> A production-grade, fully serverless coffee ordering system built with event-driven architecture (EDA). Originally presented at AWS re:Invent 2021 and still running live at AWS events worldwide. This repository contains the **application layer** — Lambda functions, Step Functions workflow, and React frontend. Infrastructure is provisioned separately via Terraform in [serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra).

---

## Architecture

```
                         ┌─────────────────────────────────────────────────┐
                         │                   AWS Cloud                      │
                         │                                                  │
  ┌──────────┐           │  ┌──────────────┐      ┌──────────────────────┐ │
  │  React   │──HTTPS───▶│  │ API Gateway  │─────▶│   Amazon Cognito     │ │
  │ Frontend │           │  │  REST API    │      │   (JWT auth)         │ │
  │ S3 + CDN │◀──────────│  └──────┬───────┘      └──────────────────────┘ │
  └──────────┘           │         │                                        │
                         │         ▼                                        │
                         │  ┌──────────────────────────────────────────┐   │
                         │  │            AWS Lambda Functions           │   │
                         │  │  OrderFn │ MenuFn │ StatusFn │ CountFn   │   │
                         │  └──────────────────┬─────────────────────┘    │
                         │                     │                           │
                         │         ┌───────────▼───────────┐              │
                         │         │    Amazon EventBridge  │              │
                         │         │   (Serverlesspresso    │              │
                         │         │       event bus)       │              │
                         │         └───────────┬────────────┘              │
                         │                     │                           │
                         │    ┌────────────────┼───────────────┐           │
                         │    ▼                ▼               ▼           │
                         │ ┌──────────┐ ┌──────────┐  ┌──────────────┐   │
                         │ │Step Func │ │ DynamoDB │  │  IoT Core    │   │
                         │ │Workflow  │ │  Tables  │  │(barista disp)│   │
                         │ └──────────┘ └──────────┘  └──────────────┘   │
                         └─────────────────────────────────────────────────┘
```

### Why event-driven?

Traditional request/response architectures tightly couple services — a slow downstream call blocks the entire flow. This app uses **orchestration + choreography**:

- **Orchestration** via Step Functions: the order lifecycle is a state machine with explicit retries, error handlers, and wait states. If a barista is slow, the workflow pauses — it does not block the API.
- **Choreography** via EventBridge: cross-cutting concerns (IoT display, analytics, notifications) subscribe to events on a shared bus. Adding a new consumer means adding one EventBridge rule — zero changes to existing code.

---

## AWS services used

| Service | Role |
|---|---|
| Amazon Cognito | User pool, JWT issuance, QR code-based sign-in |
| API Gateway | REST API with Cognito authorizer |
| AWS Lambda | Stateless business logic (Node.js 20.x) |
| AWS Step Functions | Order workflow orchestration with `waitForTaskToken` |
| Amazon EventBridge | Custom event bus decoupling all microservices |
| Amazon DynamoDB | Order state, menu config, per-user order counts |
| AWS IoT Core | Real-time WebSocket push to barista display |
| Amazon S3 + CloudFront | Frontend hosting with global CDN |
| AWS SAM | Application packaging and deployment |
| Amazon CloudWatch | Structured logs, metrics, X-Ray tracing |

---

## Repository structure

```
serverlesspresso-eda/
├── backend/
│   ├── src/
│   │   ├── orderManager/         # Step Functions workflow (ASL definition)
│   │   ├── orderProcessor/       # Lambda: validate and create orders
│   │   ├── menuManager/          # Lambda: read/write menu from DynamoDB
│   │   ├── countManager/         # Lambda: enforce per-user order limits
│   │   └── configManager/        # Lambda: shop open/close control
│   ├── template.yaml             # AWS SAM template
│   └── samconfig.toml            # SAM deploy config (region, stack name)
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   └── services/             # Amplify + IoT client setup
│   └── .env.example
├── docs/
│   ├── architecture.png
│   └── event-catalog.md          # All EventBridge event schemas documented
└── .github/
    └── workflows/
        ├── deploy-backend.yml    # CI: sam build + sam deploy on push to main
        └── deploy-frontend.yml   # CI: npm build + S3 sync on push to main
```

---

## Prerequisites

- AWS account (Free Tier sufficient for development)
- [Node.js](https://nodejs.org/) v20+
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) configured with your credentials
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- Infrastructure stack already deployed from [serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra)

---

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/serverlesspresso/serverlesspresso-eda
cd serverlesspresso-eda
```

### 2. Export infrastructure outputs

The infra repo provisions shared resources (DynamoDB table ARNs, Cognito pool ID, EventBridge bus ARN). Pull those values before deploying:

```bash
cd ../serverlesspresso-infra
terraform output -json > ../serverlesspresso-eda/backend/infra-outputs.json
```

### 3. Build and deploy the backend

```bash
cd backend
sam build
sam deploy --guided
```

The `--guided` flag walks you through stack name, region, and IAM confirmations on first run. Values are saved to `samconfig.toml` for all subsequent deploys.

After a successful deploy, SAM prints:

```
Outputs:
  ApiEndpoint         = https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/Prod
  UserPoolId          = us-east-1_XXXXXXXXX
  UserPoolClientId    = XXXXXXXXXXXXXXXXXXXXXXXXXX
  IoTEndpoint         = xxxxxxxxxx-ats.iot.us-east-1.amazonaws.com
```

### 4. Configure and run the frontend

```bash
cd ../frontend
cp .env.example .env
# Open .env and paste in the values from the SAM outputs above
npm install
npm start
```

The app will be available at `http://localhost:3000`.

### 5. Deploy the frontend to S3 + CloudFront

```bash
npm run build

BUCKET=$(cd ../serverlesspresso-infra && terraform output -raw frontend_bucket_name)
aws s3 sync build/ s3://$BUCKET --delete

DIST_ID=$(cd ../serverlesspresso-infra && terraform output -raw cloudfront_distribution_id)
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

---

## Step Functions order workflow

The order lifecycle is a Step Functions Express Workflow. Key states:

```
PlaceOrder
    │
    ▼
CheckShopStatus ──(closed)──▶ OrderRejected
    │
  (open)
    │
    ▼
CheckOrderCount ──(limit reached)──▶ OrderRejected
    │
  (under limit)
    │
    ▼
CreateOrder ──▶ WaitForBarista ── waitForTaskToken ──┐
                                                      │
                                              (barista responds)
                                                      │
                                         ┌────────────┴────────────┐
                                      (complete)               (cancel)
                                         │                         │
                                         ▼                         ▼
                                   OrderComplete             OrderCancelled
```

The `WaitForBarista` state uses the `waitForTaskToken` integration pattern. The workflow pauses and only resumes when the barista app sends an explicit `SendTaskSuccess` or `SendTaskFailure` callback. No polling loop, no wasted Lambda compute.

---

## EventBridge event catalog

All events are published to the `Serverlesspresso` custom event bus:

| Event detail-type | Source | Triggered when |
|---|---|---|
| `OrderPlaced` | `orderProcessor` | New order created, workflow started |
| `OrderCompleted` | `orderManager` | Barista marked the drink ready |
| `OrderCancelled` | `orderManager` | Order was cancelled by barista |
| `ShopOpened` / `ShopClosed` | `configManager` | Shop status toggled |
| `MenuUpdated` | `menuManager` | Menu item added or modified |

Consumers (IoT display, notifications, analytics) attach EventBridge rules directly — they never call producers. Adding a new integration requires no changes to existing Lambda code.

---

## Environment variables

`frontend/.env` (copy from `.env.example`):

```env
REACT_APP_API_ENDPOINT=https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/Prod
REACT_APP_COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
REACT_APP_COGNITO_CLIENT_ID=XXXXXXXXXXXXXXXXXXXXXXXXXX
REACT_APP_AWS_REGION=us-east-1
REACT_APP_IOT_ENDPOINT=xxxxxxxxxx-ats.iot.us-east-1.amazonaws.com
```

---

## CI/CD

GitHub Actions run on every push to `main`:

- `deploy-backend.yml` — `sam build && sam deploy`
- `deploy-frontend.yml` — `npm run build`, S3 sync, CloudFront invalidation

Store AWS credentials as repository secrets: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

---

## Monitoring

```bash
# Tail Lambda logs in real time
sam logs -n OrderProcessorFunction --stack-name serverlesspresso --tail

# List running Step Functions executions
aws stepfunctions list-executions \
  --state-machine-arn <STATE_MACHINE_ARN> \
  --status-filter RUNNING

# View structured logs in CloudWatch Insights
# Log group: /aws/lambda/serverlesspresso-OrderProcessor
```

---

## Cost estimate

Designed to run comfortably within AWS Free Tier for development:

| Service | Free Tier limit | Typical dev usage |
|---|---|---|
| Lambda | 1M requests / month | ~10K requests |
| API Gateway | 1M API calls / month | ~10K calls |
| Step Functions (Express) | unlimited for first year, then $0.00001/state transition | ~5K transitions |
| DynamoDB | 25 GB + 25 WCU/RCU | under 1 GB |
| CloudFront | 1 TB transfer / month | minimal |

Estimated cost for 1,000 real coffee orders in production: under $1.

---

## Related repository

> This repo contains **application code only**.
> All AWS infrastructure — VPC, IAM roles, DynamoDB tables, Cognito user pool, EventBridge bus, S3 bucket, CloudFront distribution — is provisioned via Terraform in a separate repository:
>
> **[github.com/serverlesspresso/serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra)**
>
> Deploy the infrastructure first, then use its Terraform outputs to deploy this application.

---

## Cleanup

```bash
# Remove the application stack
sam delete --stack-name serverlesspresso

# Destroy infrastructure (run from serverlesspresso-infra)
terraform destroy
```

---

## Acknowledgements

Based on the official [Serverlesspresso workshop](https://catalog.workshops.aws/serverlesspresso) by the AWS Serverless Developer Advocacy team (James Beswick, Julian Wood, Eric Johnson). Extended with Terraform-managed infrastructure, GitHub Actions CI/CD, and updated to Lambda Node.js 20.x runtime.

---

## License

MIT-0 — see [LICENSE](./LICENSE)