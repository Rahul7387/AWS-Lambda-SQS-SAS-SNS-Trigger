# AWS Lambda + SQS + SNS Order Processing Pipeline

A step-by-step guide to building a serverless order processing pipeline on AWS using IAM, SQS, SNS, Lambda, and CloudWatch.

---

## Architecture Overview

```
OrderQueue (SQS)
  └── triggers → OrderProcessor (Lambda)
                    ├── success → OrderNotifications (SNS)
                    │               ├── Email subscription
                    │               └── ProcessedOrders (SQS)
                    └── always  → OrderAlerts (SNS)
                                    └── Email subscription (optional filter: FAILED only)

OrderQueue → Dead-letter → OrderDLQ (after 3 failed attempts)
```

---

## Step 1 — IAM: Create the Lambda Execution Role

Lambda needs an IAM role to:
- Be assumed by the Lambda service
- Read messages from SQS (and delete them after processing)
- Publish messages to SNS
- Write logs to CloudWatch

### 1.1 Open IAM
- In the AWS Console, search for **IAM** and open it.
- In the left sidebar click **Roles** → **Create role**.

### 1.2 Configure the Trust Policy

| Field | Value |
|---|---|
| Trusted entity type | AWS service |
| Use case | Lambda |

Click **Next**.

### 1.3 Attach Permissions

| Policy name | Why |
|---|---|
| `AWSLambdaSQSQueueExecutionRole` | Grants `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` |
| `AmazonSNSFullAccess` | Grants `sns:Publish` so Lambda can post notifications |

> **Least-privilege note:** `AmazonSNSFullAccess` is used here for simplicity. In production, create a custom policy scoped to only `sns:Publish` on your specific topic ARN.

Click **Next**.

### 1.4 Name the Role

| Field | Value |
|---|---|
| Role name | `OrderProcessorLambdaRole` |
| Description | Execution role for OrderProcessor Lambda — SQS trigger + SNS publish |

Click **Create role**.

### 1.5 Note the Role ARN
- Open the role you just created.
- Copy the **ARN** (e.g. `arn:aws:iam::<YOUR_ACCOUNT_ID>:role/OrderProcessorLambdaRole`)

> **CloudWatch Logs:** `AWSLambdaSQSQueueExecutionRole` already includes `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents`. No extra policy needed.

---

## Step 2 — SQS: Create OrderQueue and Dead-Letter Queue

Create two queues — **OrderDLQ** first (main queue references it), then **OrderQueue**.

### 2.1 Create the Dead-Letter Queue (OrderDLQ)
- In the AWS Console, search for **SQS** and open it.
- Click **Create queue**.

| Field | Value |
|---|---|
| Type | Standard |
| Name | `OrderDLQ` |

Leave all other settings at defaults and click **Create queue**. Copy the **Queue ARN**.

### 2.2 Create the Main Queue (OrderQueue)

| Field | Value |
|---|---|
| Type | Standard |
| Name | `OrderQueue` |

- Scroll to **Dead-letter queue** → **Enabled**.
- Set queue to `OrderDLQ` and **Maximum receives** to `3`.

> **Maximum receives = 3:** If the same message fails 3 times, SQS moves it to the DLQ automatically.

Click **Create queue**. Copy the **Queue URL** and **Queue ARN**.

---

## Step 3 — SNS: Create Topics and Subscribe

| Topic | Purpose |
|---|---|
| `OrderNotifications` | Business events — published on successful processing |
| `OrderAlerts` | Operational alerts — published on success AND failure |

### 3.1 Create OrderNotifications Topic
- SNS → **Create topic** → Standard → Name: `OrderNotifications`
- Click **Create topic** and copy the **Topic ARN**.

### 3.2 Create OrderAlerts Topic
- SNS → **Create topic** → Standard → Name: `OrderAlerts`
- Click **Create topic** and copy the **Topic ARN**.

### 3.3 Subscribe Your Email to OrderAlerts
- OrderAlerts → **Create subscription** → Protocol: Email → your address.
- Confirm from your inbox.

### 3.4 (Optional) Filter Alerts by Status

To receive only failure emails, add a subscription filter policy:

```json
{
  "status": ["FAILED"]
}
```

SNS → OrderAlerts → Subscriptions → your subscription → **Edit** → paste under **Subscription filter policy**.

### 3.5 Subscribe Your Email to OrderNotifications
- OrderNotifications → **Create subscription** → Protocol: Email → your address. Confirm.

### 3.6 Create a ProcessedOrders SQS Queue

SQS → **Create queue** → Standard → Name: `ProcessedOrders`. Copy the **Queue ARN**.

### 3.7 Subscribe ProcessedOrders Queue to OrderNotifications

SNS → OrderNotifications → **Create subscription** → Protocol: Amazon SQS → Endpoint: ARN of `ProcessedOrders`.

### 3.8 Add SQS Access Policy for SNS Delivery

SQS → ProcessedOrders → **Access policy** tab → **Edit**. Replace with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "<ProcessedOrders-Queue-ARN>",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "<OrderNotifications-Topic-ARN>"
        }
      }
    }
  ]
}
```

---

## Step 4 — Lambda: Create and Deploy the Function

### 4.1 Basic Configuration

Lambda → **Create function**:

| Field | Value |
|---|---|
| Author from scratch | selected |
| Function name | `OrderProcessor` |
| Runtime | Python 3.12 |
| Architecture | x86_64 |

### 4.2 Attach the Execution Role

Permissions → **Use an existing role** → `OrderProcessorLambdaRole`.

Click **Create function**.

### 4.3 Upload the Code

See [`lambda/handler.py`](./lambda/handler.py) in this repo.

**Option A — Inline editor:**
- Code source → paste `handler.py` contents → **Deploy**.

**Option B — ZIP upload:**
```bash
cd lambda
zip function.zip handler.py
```
Upload from: function page → **Upload from** → **.zip file**.

### 4.4 Set the Handler

If you uploaded `handler.py`: Configuration → General configuration → **Edit** → set Handler to `handler.lambda_handler`.

### 4.5 Set Environment Variables

Configuration → **Environment variables** → **Edit**:

| Key | Value | Required |
|---|---|---|
| `SNS_TOPIC_ARN` | ARN of `OrderNotifications` | Yes |
| `ALERT_SNS_TOPIC_ARN` | ARN of `OrderAlerts` | Optional |

---

## Step 5 — Event Source Mapping: Connect SQS to Lambda

### 5.1 Add the Trigger

OrderProcessor → **+ Add trigger**:

| Field | Value |
|---|---|
| Trigger | SQS |
| SQS queue | `OrderQueue` |
| Batch size | 5 |
| Batch window | 0 seconds |
| Report batch item failures | Enabled |

> **Report batch item failures** must be enabled — otherwise a single failed message causes the entire batch to be retried.

### 5.2 How It Works Internally

```
SQS polls (AWS-managed) → Lambda invocation
  ├── success  → SQS deletes those messages automatically
  └── failure  → messageIds in batchItemFailures
                    └── SQS retries (up to maxReceiveCount=3)
                           └── moves to OrderDLQ
```

---

## Step 6 — Testing

### 6.1 Send a Test Message via Console

SQS → OrderQueue → **Send and receive messages** → paste:

```json
{
  "orderId": "ORD-001",
  "customer": "Jane Doe",
  "amount": 49.99,
  "items": ["Widget A", "Widget B"]
}
```

### 6.2 Send via AWS CLI

```bash
aws sqs send-message \
  --queue-url <OrderQueue-URL> \
  --message-body '{"orderId":"ORD-002","customer":"John Smith","amount":129.00,"items":["Gadget X"]}'
```

### 6.3 Verify in CloudWatch

Lambda → OrderProcessor → **Monitor** → **View CloudWatch logs**. Look for:

```
Published SNS notification for order ORD-001
Processed: 1, Failed: 0
```

### 6.4 Test the DLQ Path

```bash
aws sqs send-message \
  --queue-url <OrderQueue-URL> \
  --message-body 'this is not valid json'
```

After ~2 minutes (3 retries), check SQS → OrderDLQ → **Poll for messages**.

### 6.5 Lambda Test Event (Console)

OrderProcessor → **Test** tab → Template: `SQS` → replace body with:

```json
"{\"orderId\":\"ORD-TEST\",\"customer\":\"Test User\",\"amount\":9.99}"
```

---

## Resources Created

| Resource | Type | Purpose |
|---|---|---|
| `OrderProcessorLambdaRole` | IAM Role | Lambda execution permissions |
| `OrderDLQ` | SQS Queue | Dead-letter queue |
| `OrderQueue` | SQS Queue | Main inbound queue |
| `ProcessedOrders` | SQS Queue | Downstream consumer simulation |
| `OrderNotifications` | SNS Topic | Business events |
| `OrderAlerts` | SNS Topic | Operational alerts |
| `OrderProcessor` | Lambda Function | Order processing logic |
