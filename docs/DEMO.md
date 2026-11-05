# RailQuote — 5-minute demo script

Audience: someone who knows nothing about rail, AWS or AI agents.

**0:00 — The problem (30 s)**
"Freight customers email quote requests in every shape — PDFs, forwarded chains, one-liners. Someone reads each one, checks fuel prices, weather, route risk, prices it, and emails back. RailQuote does that with an AI agent — and a human stays in control."

**0:30 — Queue (45 s)** · open the app
- "Every email becomes a row. Five came in today: two priced automatically, two needed a person, one already sent."
- Point at *Agent decision · why*: "The agent always says what it decided and why."
- Point at the cost tile: "And what it cost — two cents for the whole morning."

**1:15 — A clean request (60 s)** · click Midwest Grain
- Left: "what the agent pulled out of the email, junk removed."
- Right: "the quote, line by line, with the *source* of each number — our tariff, the government diesel price, the weather service, the conflict-events database."
- Click **Trace**: "this is every tool call the agent made — what, how long, how many tokens. Nothing is a black box."

**2:15 — A messy request (75 s)** · click Delta Chem
- "Two things were missing, so the agent refused to guess — that's a policy, not luck."
- Click **Ask shipper by email** → "it emails the customer…" (wait 3 s) "…the reply comes back, is matched to the *same* request by the email headers, and the agent fills the gaps and prices it."
- Point at **History**: "every change, by whom — agent, me, or the customer's email."

**3:30 — Human control (30 s)** · type `add 5% fuel` → `send`
- "I can override anything in plain English; the agent re-prices, logs it, and only sends when I say so."

**4:00 — Agent activity (45 s)** · top nav
- "The operations view: auto-quote rate, time to quote, tokens, cost, alarms, and the policies that blocked auto-quotes today."

**4:45 — Architecture (15 s)** · open the architecture page
- "All serverless on AWS — Lambda, DynamoDB, SES, Bedrock. No servers, about $5 a month. Every box here maps to something you just saw."

## Highlights to say out loud
- Agent with tool use, not a chatbot: it *does* things (reprice, email, update) and every action is traced.
- Human in the loop by design: policies decide what needs a person.
- Live external data drives the price: EIA diesel, NWS weather, GDELT geo risk.
- Full audit: decision + reason, trace, history, cost — per request and per day.
- Cost-engineered: serverless, no VPC, ~$5/month, one `sam deploy`.
