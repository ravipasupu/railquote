# RailQuote — AWS Architecture

Diagram: `railquote-aws-architecture.html` (open in a browser).

## Principles

- Serverless only: pay per request, $0 when idle.
- One region: **us-east-1** (Bedrock + SES inbound available, lowest prices).
- **No VPC, no subnets, no NAT.** Every service is managed with a public AWS endpoint, locked down by IAM + Cognito. A NAT Gateway alone would be ~$32/mo.
- **Email stays on your existing Gmail address.** A Gmail filter forwards a copy of incoming mail to `ravi@railquote.ai` (SES). The agent sends with `Reply-To` = your Gmail, so shipper replies land in Gmail and are forwarded again. Your Gmail is never touched; only the project domain points at SES.
- New AWS account `railquote`, IAM Identity Center for login (no long-lived keys), Budgets alarm at $5 / $10.
- Target run cost at demo traffic: **$3–8 / month** (+ domain ~$12–20/yr).

## Services

| Service | Role | Why / cost |
|---|---|---|
| Route 53 | `railquote.ai` zone; its MX record points at SES; `app.` record for the UI | Needed to receive email. $0.50/mo + domain |
| SES inbound + outbound | AWS mail server. Receives the copies Gmail forwards to `ravi@railquote.ai`; sends gap questions and quotes (Reply-To = your Gmail) | Pay per message. Sandbox → request production (free) |
| S3 | `ui` (Angular build), `raw-mail` (.eml, 30-day lifecycle), `quotes` (PDFs) | Pennies |
| CloudFront + ACM | HTTPS for Angular; S3 stays private via OAC | Free cert, 1 TB/mo free |
| Cognito | User pool for ops login → JWT | Free ≤ 10k monthly users |
| API Gateway (HTTP API) | REST endpoints + JWT authorizer | ~70% cheaper than REST API |
| Lambda (Java 21, arm64, SnapStart) | `requests-api`, `agent-chat`, `email-parser`, `quote-sender`, `signals-fetcher` | Free tier covers demo; SnapStart removes Java cold start |
| DynamoDB (on-demand) | `requests`, `conversations`, `agent_steps` (trace), `history` (audit), `signals` (TTL 24h), `tariffs` | No idle cost, 25 GB free |
| Amazon Bedrock | Email extraction + agent (Converse API with tool use), Claude Haiku | Only in production; local testing uses Claude session |
| EventBridge Scheduler | Hourly trigger for `signals-fetcher` | Free tier |
| SSM Parameter Store | EIA key, config, prompt versions (`/railquote/prompts/agent`) | Free (vs Secrets Manager $0.40/secret) |
| CloudWatch + X-Ray | Structured JSON logs (7-day retention), custom metrics (`AutoQuoteRate`, `TokensPerRequest`, `BedrockCost`), 4 alarms, request traces — all via Lambda Powertools for Java | Free tier |
| IAM | One least-privilege role per Lambda | Free |

### Not used (on purpose)

VPC / subnets / NAT, RDS/Aurora, Bedrock Agents (managed), Step Functions, ECS/Fargate/EC2, Secrets Manager, WAF, Textract.

## Flows

**Email → request (grey path)**
1. Shipper emails your normal Gmail address. A Gmail filter forwards a copy to `ravi@railquote.ai`.
2. SES receipt rule stores the raw `.eml` in S3 `raw-mail`.
3. S3 event invokes `email-parser`.
4. Parser strips junk, then: if `In-Reply-To` matches a stored Message-ID or subject has `[RQ-id]` → existing request; else Bedrock classifies new vs reply (low confidence → "Needs input: possible duplicate"). Bedrock extracts fields.
5. Request + first agent message saved. If no gaps and no risk flag → `reprice_quote` → status `Auto-quoted`.

**Ops → agent (blue path)**
6. Browser loads Angular from CloudFront / S3.
7. Cognito login → JWT.
8. API Gateway (JWT authorizer) → `requests-api` / `agent-chat`.
9. `agent-chat` runs the Bedrock tool loop with `requestId` fixed; tools: `get_request`, `update_request`, `get_signals`, `reprice_quote`, `send_email`, `save_message`. Conversation appended to `conversations`.
10. `quote-sender` renders PDF → S3 `quotes` → SES sends to shipper.
11. Shipper receives (Reply-To = your Gmail); their reply lands in Gmail, is forwarded, and maps to the same request.

**Signals**
12. EventBridge hourly → `signals-fetcher` → EIA (diesel), NWS (alerts), GDELT (events) → `signals` table.

## API (HTTP API → Lambda)

| Method · path | Does |
|---|---|
| `GET /requests` | Queue rows + counts |
| `GET /requests/{id}` | Request, quote breakdown, conversation |
| `POST /requests/{id}/chat` | User message → agent → updated request + reply |
| `POST /requests/{id}/ask-shipper` | Agent emails the gap questions |
| `POST /requests/{id}/send` | Render PDF, email quote, status → Sent |
| `GET /signals` | Current diesel / weather / geo values + sources |

## DynamoDB

| Table | Key | Notes |
|---|---|---|
| `requests` | `pk=REQ#<id>` | fields, missing[], status, quote, messageIds[] |
| `conversations` | `pk=REQ#<id>`, `sk=<timestamp>` | who (agent/user/email), text, decision, reason, promptVersion |
| `agent_steps` | `pk=REQ#<id>`, `sk=<timestamp>#<n>` | tool, input summary, output summary, ms, tokens, cost — one row per tool call |
| `history` | `pk=REQ#<id>`, `sk=<timestamp>` | field, old, new, by (agent / user sub / shipper email) |
| `signals` | `pk=SIGNAL#<type>#<region>` | value, source, fetchedAt, TTL 24h |
| `tariffs` | `pk=TARIFF#<carType>` | rate per car-mile, surcharge rules |

## Audit & observability (what the demo shows)

| In the UI | Backed by | How |
|---|---|---|
| Trace under each agent message (tool, ms, tokens, $) + decision chip + reason | `agent_steps` | Written inside the Bedrock tool loop; final answer must return `decision` + `reason` |
| History per request (old → new, by whom) | `history` | `update_request` writes a diff row; approvals logged with the Cognito user |
| Cost per request / day | `agent_steps.tokens` × Bedrock price | Also emitted as `BedrockCost` metric |
| Agent activity page (auto-quote %, time to quote, tokens/request, alarms) | CloudWatch custom metrics | Powertools `@Metrics`; alarms: cost > $1/day, tokens > 8k, auto-quote < 40%, parse failures |
| Policy checks (geo needs human; never guess missing fields; margin > 3% needs approval) | Code in the agent Lambda | Checked before `send_email`; each check logs a trace step PASSED / BLOCKED |
| Prompt version on every trace | SSM Parameter Store | Trace records which prompt version ran |
| Request timeline across services | X-Ray | Powertools `@Tracing` |

## External APIs

| Source | Endpoint | Key |
|---|---|---|
| EIA | `https://api.eia.gov/v2/petroleum/pri/gnd/data` (weekly diesel by PADD) | free key → SSM |
| NWS | `https://api.weather.gov/alerts/active?point=lat,lon` | none (User-Agent header) |
| GDELT | `https://api.gdeltproject.org/api/v2/doc/doc?query=...&mode=artlist` | none |
| Nominatim | city → lat/lon (cached in DynamoDB) | none, 1 req/s |

## Repo + deploy

```
railquote/
  ui/      Angular
  api/     Java 21 Lambdas (Maven, one module per function)
  infra/   template.yaml (AWS SAM)
  docs/    ui mock, architecture
```

- GitHub private repo: `feature/x` → PR → `main`. No GitHub Actions for now.
- Deploy from PC: `sam build` + `sam deploy` (infra + Lambdas); `ng build` + `aws s3 sync` + CloudFront invalidation (UI).
- Local POC / prompt work: Claude session, not Bedrock.

## Build phases

1. Account, Budgets, Identity Center, SAM skeleton, DynamoDB, `requests-api`, Angular queue on CloudFront + Cognito.
2. Domain + SES inbound → S3 → `email-parser` → Bedrock extraction.
3. `agent-chat` tool loop + conversations; auto-quote rule.
4. `signals-fetcher` (EIA / NWS / GDELT) + source badges in UI.
5. Ask-shipper email, reply matching, quote PDF + SES out.
