# Serverlesspresso — Event-Driven Architecture on AWS

> A production-grade, fully serverless coffee ordering system built with event-driven architecture (EDA). Originally presented at AWS re:Invent 2021 and still running live at AWS events worldwide. This repository contains the **application layer** — Lambda functions, Step Functions workflows, and API Gateway. Infrastructure is provisioned separately via Terraform in [serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra).

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
                         │  │  Validator │ Config │ Counter │ Publisher │   │
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
                         │ │Workflows │ │  Tables  │  │(barista disp)│   │
                         │ └──────────┘ └──────────┘  └──────────────┘   │
                         └─────────────────────────────────────────────────┘
```

### Why event-driven?

Traditional request/response architectures tightly couple services — a slow downstream call blocks the entire flow. This app uses **orchestration + choreography**:

- **Orchestration** via Step Functions: the order lifecycle is a state machine with explicit retries, error handlers, and wait states. If a barista is slow, the workflow pauses using `waitForTaskToken` — it does not block the API or consume Lambda compute.
- **Choreography** via EventBridge: cross-cutting concerns (IoT display, notifications) subscribe to events on a shared bus. Adding a new consumer means adding one EventBridge rule — zero changes to existing code.

---

## AWS services used

| Service | Role |
|---|---|
| Amazon Cognito | User pool, JWT issuance, QR code-based sign-in |
| API Gateway | REST API with Cognito authorizer |
| AWS Lambda | Stateless business logic (Node.js 20.x) |
| AWS Step Functions | Order workflow orchestration with `waitForTaskToken` |
| Amazon EventBridge | Custom event bus decoupling all microservices |
| Amazon DynamoDB | Order state, menu config, validator tokens, order counts |
| AWS IoT Core | Real-time WebSocket push to barista and customer displays |
| Amazon S3 + CloudFront | Frontend hosting with global CDN |
| AWS SAM | Application packaging and deployment per service |
| Amazon CloudWatch | Structured logs, metrics, alarms |

---

## Repository structure

```
serverlesspresso-eda/
├── src/                               # SAM templates and Lambda code (authored here)
│   ├── config-service/                # DynamoDB stream → EventBridge on config change
│   │   ├── configChanged.js
│   │   ├── template.yaml
│   │   └── samconfig.toml
│   ├── counting-service/              # Order ID increment and reset
│   │   ├── getOrderId.js
│   │   ├── resetOrderId.js
│   │   ├── template.yaml
│   │   └── samconfig.toml
│   ├── publisher-service/             # EventBridge → IoT Core WebSocket push
│   │   ├── publishToIOT.js
│   │   ├── publishToIOTuserTopic.js
│   │   ├── publishToIOTconfig.js
│   │   ├── template.yaml
│   │   └── samconfig.toml
│   ├── order-manager/                 # Order lifecycle Step Functions + Lambdas
│   │   ├── functions/
│   │   │   ├── sanitize/
│   │   │   ├── workflowStarted/
│   │   │   ├── waitingCompletion/
│   │   │   └── OrderTimedOut/
│   │   ├── statemachine/om.asl.json
│   │   ├── template.yaml
│   │   └── samconfig.toml
│   ├── order-processing/              # Main order processor Step Functions
│   │   ├── functions/resumeWF/
│   │   ├── statemachine/op.asl.json
│   │   ├── template.yaml
│   │   └── samconfig.toml
│   └── validator-service/             # QR code generation and verification
│       ├── getCode.js
│       ├── verifyCode.js
│       ├── ddb.js
│       ├── template.yaml
│       └── samconfig.toml
├── backend/                           # AWS reference code (read-only reference)
├── baseCore/                          # AWS reference core stack
└── appCore/                           # AWS reference app core
```

---

## Prerequisites

- AWS account (Free Tier sufficient for development)
- [Node.js](https://nodejs.org/) v20+
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) configured
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- Infrastructure stack deployed from [serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra)

---

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/serverlesspresso/serverlesspresso-eda
cd serverlesspresso-eda
```

### 2. Deploy each service

Each service is deployed independently. Run the following in order:

```bash
# 1. Config service
cd src/config-service && sam build && sam deploy --guided

# 2. Counting service
cd ../counting-service && sam build && sam deploy --guided

# 3. Publisher service
cd ../publisher-service && sam build && sam deploy --guided

# 4. Order manager
cd ../order-manager && sam build && sam deploy --guided

# 5. Order processing
cd ../order-processing && sam build && sam deploy --guided

# 6. Validator service
cd ../validator-service && sam build && sam deploy --guided
```

Each `--guided` deploy prompts for stack name and parameter values. After the first run, settings are saved to `samconfig.toml` — subsequent deploys just need `sam deploy`.

---

## Services

### config-service

Listens to the DynamoDB config table stream. When the shop opens or closes, this Lambda publishes a `ConfigService.ConfigChanged` event to EventBridge. Other services react to this event without being directly called.

**CloudFormation stack:** `serverlesspresso-config-service`

---

### counting-service

Two Lambda functions managing the per-session order number counter stored in DynamoDB:

- `GetOrderId` — increments the counter by 1, returns the new order number
- `ResetOrderId` — resets the counter to 0 when a new QR code session starts

**CloudFormation stack:** `serverlesspresso-counting-service`

---

### publisher-service

Three Lambda functions that bridge EventBridge and IoT Core. When any service emits an event, the appropriate publisher forwards it to the correct IoT topic via WebSocket:

- `PublisherAdmin` — broadcasts to the display screen topic
- `PublisherUser` — sends to the individual customer's topic
- `PublisherConfig` — forwards config change events

**CloudFormation stack:** `serverlesspresso-publisher-service`

---

### order-manager

A Step Functions state machine plus four Lambda functions managing the barista-side of the order lifecycle:

- `WorkflowStarted` — saves the new order to DynamoDB when a workflow begins
- `WaitingCompletion` — stores the Step Functions task token so the barista can resume the workflow
- `OrderTimedOut` — marks the order as cancelled if neither party acts in time
- `SanitizeOrder` — validates that the drink and modifiers exist on the current menu

**CloudFormation stack:** `serverlesspresso-order-manager`

---

### order-processing

The main order workflow. A Step Functions state machine that:

1. Checks if the shop is open
2. Checks if capacity is available (max 20 concurrent orders)
3. Emits `WorkflowStarted` and waits for the customer to submit their order (`waitForTaskToken`)
4. Generates an order number from the counting table
5. Emits `WaitingCompletion` and waits for the barista to act (`waitForTaskToken`)
6. Completes or times out based on the barista's response

**CloudFormation stack:** `serverlesspresso-order-processing`

---

### validator-service

Two Lambda functions behind API Gateway with Cognito authorizer:

- `GetCode` — admin-only endpoint that generates a 5-minute QR code stored in DynamoDB
- `VerifyCode` — validates the scanned token, decrements the available count, and emits `Validator.NewOrder` to start the order workflow

**CloudFormation stack:** `serverlesspresso-validator-service`

---

## EventBridge event catalog

All events flow through the `serverlesspresso-dev` custom event bus:

| Event | Source | Triggered when |
|---|---|---|
| `Validator.NewOrder` | `validator-service` | Customer scans valid QR code |
| `OrderProcessor.WorkflowStarted` | `order-processing` | Workflow begins, awaiting order submission |
| `OrderProcessor.WaitingCompletion` | `order-processing` | Order submitted, awaiting barista |
| `OrderProcessor.OrderTimeOut` | `order-processing` | Customer or barista timed out |
| `OrderProcessor.orderFinished` | `order-processing` | Order lifecycle complete |
| `OrderManager.WaitingCompletion` | `order-manager` | Task token stored, barista notified |
| `OrderManager.OrderCOMPLETED` | `order-manager` | Barista completed the order |
| `OrderManager.OrderCANCELLED` | `order-manager` | Barista cancelled the order |
| `OrderManager.MakeOrder` | `order-manager` | Barista claimed the order |
| `ConfigService.ConfigChanged` | `config-service` | Shop opened or closed |

---

## Monitoring

```bash
# Tail Lambda logs
sam logs -n ConfigChangedFunction --stack-name serverlesspresso-config-service --tail
sam logs -n VerifyCodeFunction --stack-name serverlesspresso-validator-service --tail

# List running Step Functions executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:645804193376:stateMachine:serverlesspresso-order-processor \
  --status-filter RUNNING
```

---

## Cost estimate

| Service | Free Tier limit | Typical dev usage |
|---|---|---|
| Lambda | 1M requests / month | ~10K requests |
| API Gateway | 1M API calls / month | ~10K calls |
| Step Functions (Express) | 4K state transitions / month free | ~5K transitions |
| DynamoDB | 25 GB + 25 WCU/RCU | under 1 GB |
| IoT Core | 500K messages / month free | minimal |

---

## Related repository

> This repo contains **application code only**.
> All AWS infrastructure — VPC, IAM roles, DynamoDB tables, Cognito user pool, EventBridge bus, S3 bucket, CloudFront distribution — is provisioned via Terraform in:
>
> **[github.com/serverlesspresso/serverlesspresso-infra](https://github.com/serverlesspresso/serverlesspresso-infra)**
>
> Deploy infrastructure first, then deploy each service in this repo using the Terraform outputs.

---

## Cleanup

```bash
# Delete all service stacks
sam delete --stack-name serverlesspresso-validator-service
sam delete --stack-name serverlesspresso-order-processing
sam delete --stack-name serverlesspresso-order-manager
sam delete --stack-name serverlesspresso-publisher-service
sam delete --stack-name serverlesspresso-counting-service
sam delete --stack-name serverlesspresso-config-service

# Destroy infrastructure
cd ../serverlesspresso-infra/environments/dev
terraform destroy
```

---

## Acknowledgements

Based on the official [Serverlesspresso workshop](https://catalog.workshops.aws/serverlesspresso) by the AWS Serverless Developer Advocacy team. Extended with Terraform-managed infrastructure and independent per-service SAM deployments.

---

## License

MIT-0 — see [LICENSE](./LICENSE)