# Order Data Quality Alert

Checks every incoming order for missing or malformed customer data and suspicious discounts, groups the findings by store, and sends one alert per store.
A daily digest shows the share of orders with missing contact or address information per store.

## Who it is for

Retail and e-commerce teams that sell through both stores (POS) and an online shop, and want bad order data caught the same day, while a store can still fix it.
Typical users: operations, CRM and store managers.

## What it checks

| Code | Rule |
|------|------|
| `missing_phone` | No phone on the order, the customer, or the shipping or billing address |
| `invalid_phone` | Contains characters other than digits, spaces, `+ ( ) . -`, has fewer than `minPhoneDigits` or more than `maxPhoneDigits` digits, or is one repeated digit |
| `missing_email` | No email on the order or the customer |
| `missing_shipping_address` | Missing address line, city or country. Skipped for POS and pickup orders |
| `suspicious_discount` | `total_discounts` divided by `total_line_items_price` is above `discountThresholdPct` |

All checks are plain JavaScript in the Code node `Check Orders`.
No LLM is used in this template.

## Demo vs live

**Demo (default).**
Import the workflow and click "Test workflow" on `Manual Trigger (demo run)`.
No credentials are needed.
`Demo Orders (DEFAULT)` creates 72 synthetic orders across 12 stores ("Store A01" to "Store A12") plus online orders.
The output is JSON with the messages that would be sent, for the per-store alerts and the daily digest.
Nothing leaves n8n.

**Live.**
1. Open `Config` and set `mode` to `live`.
2. Set `telegramChatId`.
   Optionally set `storeChatMap` to a JSON object such as `{"Store A01": "<chat id>"}` to route a store to its own chat.
3. Real-time alerts: point a Shopify "order created" webhook at the URL of `Webhook (real-time order)`, or enable `Shopify Trigger (LIVE - disabled)`.
4. Daily digest: enable `Shopify: Get Orders (LIVE - disabled)` (reads the last 24 hours of orders) and activate the workflow.
5. Enable `Telegram: Send Alert (LIVE - disabled)` and select a credential.

Nodes that need a credential ship disabled so the workflow imports and runs cleanly.
If you set `mode` to `live` and forget to enable the Shopify node, `Check Orders` stops with an error that names the node.

## Credentials needed (live only)

- Shopify Access Token (for the Shopify nodes)
- Telegram Bot API (for the alert node)

The webhook has no authentication by default.
Add header auth in the Webhook node before exposing it.

## Node map

```
Manual Trigger ----------+
Webhook (POST order) ----+
Shopify Trigger (LIVE) --+--> Config --> Payload received?
Schedule Trigger (08:00)-+                 |yes: Normalize Payload ---------+
                                           |no : Mode is live?              |
                                                  |live: Shopify: Get Orders +--> Check Orders
                                                  |demo: Demo Orders --------+        |
                                                                                Format Messages
                                                                                      |
                                                                              Send to Telegram?
                                                                                |yes        |no
                                                                      Telegram: Send Alert  |
                                                                                +--> Join Branches --> Summarize Result --> Respond to Webhook
```

`Config` sets `run_type` from the trigger: `order` (webhook or Shopify trigger), `digest` (schedule), `demo` (manual, runs both).

## Customize

- Thresholds and routing: edit the `Config` node.
- Store attribution: edit `storeOf()` in `Check Orders`.
  The default reads the order note attribute `store`, then `store_name`, then a `store:NAME` tag, then `location_id`.
- Add a rule: add one block in the loop in `Check Orders` that pushes `{ code, detail }`, and a label in `Format Messages`.
- Other channels: replace the Telegram node with Slack, email or a ticketing node.
  It receives `message` and `chat_id`.
- Digest schedule: edit the cron expression in `Schedule Trigger (daily digest)`.

## Limitations

- The digest covers the orders returned by the Shopify node (last 24 hours).
  The workflow keeps no history, so it cannot show trends.
- The phone check is generic (length and characters).
  It does not validate country codes or numbering plans.
- Orders are checked once when they arrive.
  Later edits in Shopify are not re-checked.
- Shopify node field names (`createdAtMin`, `status`) and the Telegram and Shopify nodes were not run against a live instance when this template was written.
- Telegram messages over 4096 characters are rejected by Telegram.
  Each store alert lists at most 15 orders.
