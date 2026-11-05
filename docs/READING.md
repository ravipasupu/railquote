# What to read before we build (keep it light — 10–15 min each)

One page per topic. Read in this order; each one is used in the phase next to it.

| # | Topic | What it is, in one line | Read this | Used in |
|---|---|---|---|---|
| 1 | **AWS SAM** | CloudFormation with shortcuts: one `template.yaml` describes Lambdas, API, tables; `sam deploy` creates them all | https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html | Phase 1 |
| 2 | **Lambda** (Java) | A function that runs only when called; you pay per 1 ms. Java 21 + SnapStart = no cold-start pain | https://docs.aws.amazon.com/lambda/latest/dg/lambda-java.html | Phase 1 |
| 3 | **DynamoDB basics** | Key-value table: `pk` + `sk`, no joins, no idle cost. Think "a HashMap in the cloud" | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html | Phase 1 |
| 4 | **API Gateway HTTP API + Cognito JWT** | The URL in front of Lambda; Cognito issues the login token, API Gateway checks it | https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html | Phase 1 |
| 5 | **S3 + CloudFront for a SPA** | Angular build files in a private bucket, served over HTTPS by CloudFront | https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html | Phase 1 |
| 6 | **SES email receiving** | Point a domain's MX record at SES; a receipt rule drops each email in S3 and triggers Lambda | https://docs.aws.amazon.com/ses/latest/dg/receiving-email.html | Phase 2 |
| 7 | **Bedrock Converse API + tool use** | One API to call Claude; "tools" = JSON functions the model can ask you to run (this *is* the agent) | https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html | Phase 2–3 |
| 8 | **EventBridge Scheduler** | Cron in the cloud: "run this Lambda every hour" | https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html | Phase 4 |
| 9 | **Lambda Powertools for Java** | Annotations that give structured logs, X-Ray traces and custom metrics for free | https://docs.powertools.aws.dev/lambda/java/ | Phase 3–4 |
| 10 | **IAM Identity Center + AWS CLI login** | How you log in from PowerShell without storing long-lived keys (`aws sso login`) | https://docs.aws.amazon.com/cli/latest/userguide/sso-configure-profile-token.html | Phase 1 (day 1) |

## Words you'll hear a lot

- **IaC** — infrastructure as code: the `template.yaml` *is* the AWS setup; delete the stack, everything is gone.
- **Stack** — one CloudFormation deployment (ours is named `railquote`).
- **IAM role** — what a Lambda is *allowed* to do (e.g. write to one DynamoDB table and nothing else).
- **Cold start** — the first call to an idle Lambda is slower; SnapStart fixes this for Java.
- **On-demand** (DynamoDB) — pay per read/write, $0 when idle.
- **Tool use / function calling** — the model answers with "call `reprice_quote` with these args"; your code runs it and sends the result back; loop until the model gives a final answer.
- **Trace** — the list of tool calls one request caused (what the Trace button shows).

## How each session will go

1. Claude explains the step and the AWS feature in plain English (+ one link).
2. Claude writes the code and explains it block by block.
3. You read it, ask anything, and say "go".
4. You run the one PowerShell block; paste the output.
5. Claude verifies; ends with "What you should now understand".
