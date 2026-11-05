# RailQuote

AI agent that turns messy rail-freight quote request emails into priced, delivery-dated quotes.

- Email in → agent extracts the order, drops junk, checks live signals (diesel, weather, geo risk)
- No gaps, no risk → auto-quotes
- Gaps → agent asks (in-app chat or email to shipper) and updates the **same** request from the answer
- Every request has its own saved conversation with the agent

## Folders

| Folder | What |
|---|---|
| `docs/ui-mock/railquote-mock.html` | Working UI mock — open in a browser, click a row, chat with the agent |
| `docs/architecture/` | AWS architecture diagram + ARCHITECTURE.md |
| `ui/` | Angular app (to be created) |
| `api/` | Java Lambdas (to be created) |
| `infra/` | AWS SAM template (to be created) |

## Stack (planned)

Angular · API Gateway (HTTP) · Lambda (Java 21) · DynamoDB · S3 · CloudFront · SES · EventBridge Scheduler · Bedrock (Claude) · Cognito · AWS SAM
