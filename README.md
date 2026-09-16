
# OrderFlow — Event-Driven Order Processing System

A serverless, event-driven order processing system built on AWS, demonstrating asynchronous architecture, failure handling, security best practices, and operational monitoring.

## Problem

Traditional request-response systems (like a direct API-to-database call) tightly couple every step of processing. If one part is slow or fails, the whole request fails. Real-world order systems need to accept an order instantly, then process it independently and reliably — even if processing takes time or occasionally fails.

## Solution

OrderFlow decouples order creation from order processing using an event-driven architecture. Orders are created instantly via API, then routed through an event bus and queue for asynchronous processing, with automatic retries, dead-letter handling for failures, and real-time notifications.

## Architecture

Client
→ API Gateway (POST /orders)
→ Lambda: OrderFlowCreateOrder
→ DynamoDB (order saved)
→ EventBridge (OrderCreated event published)
→ EventBridge Rule
→ SQS Queue (OrderFlowQueue)
→ Lambda: OrderFlowProcessOrder
→ DynamoDB (status updated to PROCESSED)
→ SNS (email notification sent)

Failure path:
SQS Queue → (3 failed retries) → Dead-Letter Queue (OrderFlowQueueDLQ)

Read path:
Client → API Gateway (GET /orders, GET /orders/{orderId}) → Lambda: OrderFlowGetOrders → DynamoDB

## AWS Services Used

- **API Gateway (HTTP API)** — public entry point for creating and reading orders
- **AWS Lambda** (3 functions) — order creation, asynchronous processing, and reading orders
- **Amazon DynamoDB** — order storage
- **Amazon EventBridge** — custom event bus and rule to decouple order creation from processing
- **Amazon SQS** — queue for asynchronous processing, with a dead-letter queue for failed messages
- **Amazon SNS** — email notifications on successful processing
- **Amazon CloudWatch** — alarms for Lambda errors and dead-letter queue activity
- **IAM** — custom least-privilege policies per Lambda function

## Features

- Create an order via API (`POST /orders`)
- List all orders (`GET /orders`)
- Retrieve a single order (`GET /orders/{orderId}`)
- Asynchronous order processing decoupled via EventBridge and SQS
- Automatic retry with dead-letter queue for failed processing
- Email notification on successful order processing
- CloudWatch alarms for Lambda errors and dead-letter queue depth

![Order creation via API](screenshots/02-api-create-response.png)
*Successful order creation — DynamoDB save and event publish confirmed in a single response*

![DynamoDB order storage](screenshots/01-dynamodb-orders.png)
*Orders stored in DynamoDB, showing both CREATED and PROCESSED states across multiple test runs*

![Email notification](screenshots/03-sns-email-notification.png)
*Automatic email notification sent via SNS after successful order processing*

## Security

Each Lambda function has a custom IAM policy scoped to only the specific resources and actions it needs (least privilege), rather than broad managed policies:
- `OrderFlowCreateOrder` — can only write to the `OrderFlowOrders` table and publish to the specific EventBridge bus
- `OrderFlowProcessOrder` — can only update the `OrderFlowOrders` table, consume from the specific SQS queue, and publish to the specific SNS topic
- `OrderFlowGetOrders` — read-only DynamoDB access

![IAM least-privilege policy](screenshots/04-iam-least-privilege.png)
*Custom IAM policy scoped to a single table and single event bus, replacing broad managed policies*

## Monitoring

Two CloudWatch alarms provide operational visibility without manual checking:
- Triggers on any Lambda processing error
- Triggers if any message lands in the dead-letter queue

Both alarms notify via the same SNS topic used for order notifications.

![CloudWatch alarms](screenshots/05-cloudwatch-alarms.png)
*Active CloudWatch alarms monitoring Lambda errors and dead-letter queue activity*

## Testing

- End-to-end flow tested: order creation → DynamoDB save → event publish → queue → processing → status update → notification
- **Deliberate failure testing:** introduced a controlled failure in the processing Lambda to verify the dead-letter queue correctly captures messages after 3 failed retries. Confirmed via CloudWatch Logs and the DLQ message count (receive count of 4, including manual inspection), ensuring no order is silently lost on repeated failure.
- Verified least-privilege IAM policies did not break functionality after removing broad managed policies — re-ran the full flow post-migration with no errors
- GET endpoints tested for both list and single-item retrieval

## Cost Considerations

All services used operate on a pay-per-use, serverless model (Lambda, DynamoDB on-demand, SQS, SNS, EventBridge, API Gateway HTTP API). No idle infrastructure (no EC2, no NAT Gateway). Total cost for building and testing this project was negligible (well under $1).

## Challenges & Troubleshooting

- **Double JSON parsing bug:** initially attempted to parse the EventBridge event detail twice (once as part of the SQS message body, once again assuming it was still a string), causing a `SyntaxError`. Diagnosed via CloudWatch Logs and fixed by recognizing the event detail was already a parsed object.
- **AWS Console iframe loading issue:** the Lambda code editor intermittently failed to load due to a local network/browser environment issue (confirmed via AWS Health Dashboard that no AWS-side outage was occurring in the region).

## What I Learned

- Designing and implementing event-driven, decoupled architecture using EventBridge and SQS
- Implementing and testing dead-letter queue failure handling
- Applying least-privilege IAM policies instead of default broad permissions
- Setting up operational monitoring with CloudWatch alarms
- Debugging real runtime errors using CloudWatch Logs

## Future Improvements

- API authentication (e.g., Cognito or API keys)
- Infrastructure as Code (e.g., AWS SAM or Terraform) instead of manual console configuration
- CI/CD pipeline for automated deployment
- Additional order status transitions (e.g., cancelled, shipped)

---

Built as part of an ongoing AWS learning journey, following an earlier serverless CRUD project, [CloudStarter](https://github.com/mrayanansari06/cloudstarter-serverless-task-app).
