# n8n Retail Templates

Three n8n workflow templates for retail and e-commerce brands.
Each one runs on first import with no credentials, using synthetic demo data, and switches to live integrations by changing one `mode` value in a `Config` node.
Deterministic logic lives in JavaScript Code nodes.
An LLM, where used, only writes the human-readable summary and is optional.

| Template | Value |
|----------|-------|
| [order-data-quality-alert](order-data-quality-alert/) | Flags orders with missing or invalid phone, email or address and suspicious discounts, alerts each store, and sends a daily missing-info rate per store. |
| [size-stockout-detector](size-stockout-detector/) | Finds best-selling sizes that are empty in a store while stock exists elsewhere, proposes capped, ranked transfers, and requires human approval. |
| [daily-sales-brief-agent](daily-sales-brief-agent/) | Computes yesterday's revenue, orders, AOV, top products and day and week comparisons in code, then posts a facts-only brief to Telegram or Slack. |

## Quick start

1. In n8n, choose Workflows > Import from file and select a `workflow.json`.
2. Click "Test workflow" on the Manual Trigger.
   The result appears on the last node.
3. For live use, follow the "Demo vs live" section in the template's README.

Nodes that need credentials are disabled by default and labelled `LIVE`, so the demo imports and runs without credential prompts.

## Requirements

- A recent n8n version (templates use current node versions, including Schedule Trigger 1.3, Merge 3.2, Google Sheets 4.7, Wait 1.1 with the form resume option).
- All data in the demos is synthetic: stores "Store A01" to "Store A12", SKUs "SKU-1001" and up, "Customer 001" and up, `example.com` emails, `555` phone numbers.

## Validation

Each file was validated offline with `n8nctl workflow validate <file>`.
All three were imported into a clean n8n 2.48.4 container and executed in demo mode with `n8n execute`; every node finished with status success (2026-11-17).
The live-mode nodes (Shopify, Telegram, Slack, Google Sheets, Anthropic) were not executed, because they need real credentials.
See each README for what was not verified.

## License

MIT, see [LICENSE](LICENSE).
