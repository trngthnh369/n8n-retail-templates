# Daily Sales Brief Agent

Every morning at 07:00 it reads yesterday's orders, computes the numbers in code, and posts a short manager brief to Telegram or Slack.
The LLM only writes the wording.
It does not calculate anything.

## Who it is for

Retail and e-commerce managers who want one message with yesterday's result instead of opening several dashboards.
Typical users: store and regional managers, e-commerce leads, founders.

## What the brief contains

- Revenue, orders, AOV and units for yesterday
- Change versus the day before and versus the same weekday last week (absolute and percent)
- Top 5 products by revenue
- Split by channel (Retail POS vs Online) and the top 5 stores or channels
- Warnings, for example when a comparison period has no orders

Definitions are in the sticky note `Note - KPIs` and in the header of `Compute KPIs`.
A day is a calendar date in `Config.timezone`.
Cancelled and test orders are excluded.
A percent change is `n/a` when the base period has no orders.

## Facts-only rule for the LLM

The system prompt in `Build Prompt` is:

```
You write the daily sales brief for a retail manager.
Rules:
1. Use only the numbers and names present in FACTS_JSON. Never calculate, estimate, round differently, abbreviate (no k or M) or invent a number, product, store or cause.
2. Write numbers exactly as they appear in FACTS_JSON. Put the currency code before money amounts.
3. If a value is null or n/a, write n/a. If warnings is not empty, mention each warning in one line.
4. State what changed and by how much. Do not explain causes and do not give advice.
5. Plain text, 5 to 8 short lines, no markdown, no emoji, at most 120 words.
```

The user message is `FACTS_JSON:` followed by the computed JSON.
`Finalize Brief` then checks the reply: 3 to 12 lines, and every number must appear in the facts (within its own rounding).
If a check fails, the LLM is off, or the call errors, the template brief is used and `summary_note` explains why.
The check covers numbers, not wording.
It cannot tell "up" from "down".

## Demo vs live

**Demo (default).**
Click "Test workflow" on `Manual Trigger (demo run)`.
No credentials and no API key are needed.
`Demo Orders (DEFAULT)` generates about 1,000 synthetic orders for the last 9 days (12 "Store A01" to "Store A12" stores plus online).
`useLlm` is false, so the brief comes from the template.
The output is JSON with the brief and the computed facts.

**Live.**
1. In `Config` set `mode` to `live`, `timezone`, `currency`, `deliveryChannel` (`telegram` or `slack`) and `telegramChatId` or `slackChannelId`.
2. Enable `Shopify: Get Orders (LIVE - disabled)` and select a Shopify Access Token credential.
   It reads the last 9 days of orders.
3. Enable the Telegram or Slack node and select a credential.
4. Optional LLM: set `useLlm` to true, enable `Anthropic: Write Brief`, select a Header Auth credential (header name `x-api-key`) and set `llmModel`.
5. Set the workflow timezone (Settings) to the same value as `Config.timezone`, then activate.

## Credentials needed (live only)

- Shopify Access Token
- Telegram Bot API or Slack
- Optional: Header Auth for the LLM API (Anthropic Messages API as configured)

## Node map

```
Schedule (07:00) --+
Manual (demo) -----+--> Config --> Mode is live?
                                    |live: Shopify: Get Orders --+
                                    |demo: Demo Orders ----------+--> Compute KPIs --> Build Prompt --> Use LLM?
   --yes--> Anthropic: Write Brief --+
   --no -----------------------------+--> Finalize Brief --> Mode is live? (delivery)
                                                              |live: Deliver to Slack? --yes--> Slack: Send Brief
                                                              |                         --no ---> Telegram: Send Brief
                                                              |demo: Demo Output
```

## Customize

- Metrics, definitions and channel mapping: `Compute KPIs`.
- Template wording: `Build Prompt`.
  LLM rules: the `system` string in the same node.
- Different order source (WooCommerce, ERP, database): replace the Shopify node with any node that returns orders with `created_at`, `total_price`, `line_items`, `source_name`.
- Different model or provider: change the HTTP node URL, headers and body, and the reply parsing in `Finalize Brief`.
- Schedule: edit the cron expression in `Schedule Trigger (daily 07:00)`.

## Limitations

- Revenue is the order total as Shopify reports it.
  Refunds issued later are not subtracted.
- The workflow fetches the last 9 days of orders on every run.
  For very large shops, narrow the window or aggregate upstream.
- Store attribution depends on a `store` note attribute, `store_name` or `location_id` on POS orders.
  Edit `storeOf()` for your setup.
- Shopify, Telegram, Slack and the Anthropic HTTP call were not run against a live instance when this template was written.
  The model name in `Config.llmModel` is a placeholder: use one your account can access.
