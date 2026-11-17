# Size Stockout Detector

Finds hero SKU sizes that are empty at a store while the warehouse or another store still has stock, and proposes transfers.
Proposals are ranked by lost demand, capped per run, and need a person's approval before they are marked approved.

## Who it is for

Footwear and apparel retailers with a store network and a central warehouse, where a missing size in a best-selling style means a lost sale.
Typical users: merchandising, inventory planning and regional managers.

## How it decides

| Term | Definition |
|------|------------|
| Hero SKU | A SKU listed in `Config.heroSkus` |
| Gap | A hero SKU size with `on_hand = 0` at a store and demand above 0 |
| Demand | The store's `sold_30d`. If it sold 0 during a long stock-out, the average of stores that still have stock (`demand_source` says which) |
| Cover days | `on_hand / (sold_30d / 30)`. A location with no sales has unlimited cover |
| Source | The warehouse first (`preferWarehouse = true`), otherwise the store with the most cover days |
| Donor limits | A donor never drops below `donorMinKeep` units and needs at least `minDonorCoverDays` of cover |
| Ranking | By demand, highest first. Stock is reserved as it is allocated, so one unit is never proposed twice |
| Cap | At most `maxTransfers` proposals per run. The rest are reported as deferred |

All of this is plain JavaScript in `Detect Transfers`.
The LLM only rewrites a 5-line summary.
Gaps with no stock anywhere are listed as `unmatched` (reorder candidates).

## Demo vs live

**Demo (default).**
Click "Test workflow" on `Manual Trigger (demo run)`.
No credentials are needed.
`Demo Inventory (DEFAULT)` generates a snapshot for 12 stores, 1 warehouse, 20 SKUs and sizes 38 to 44.
The result is JSON with the ranked table, the summary and the facts.
The demo never reaches the approval step and writes nothing.

**Live.**
1. In `Config` set `mode` to `live`, `heroSkus`, `warehouseName`, `sheetId` and `telegramChatId`.
2. Create a Google Sheet with a tab `inventory` (columns `store, sku, size, on_hand, sold_30d`; warehouse rows use `store = warehouseName`) and empty tabs `proposals` and `approved_transfers`.
3. Enable `Sheets: Read Inventory`, `Sheets: Append Proposals`, `Sheets: Append Approved` and `Telegram: Ask Approval`, and select credentials.
4. Optional LLM summary: set `useLlm` to true, enable `Anthropic: Write Summary`, select a Header Auth credential (header name `x-api-key`) and set `llmModel`.
5. Activate the workflow.
   It runs daily at 08:00 in the workflow timezone.

The Google Sheets nodes append rows using the header names written by `Proposal Rows` and `Approved Rows` (`run_date, rank, sku, size, to_store, from_location, qty, ...`).
Put those names in row 1 of each tab.

## Human approval

In live mode:
1. Ranked proposals are appended to the `proposals` tab with `status = proposed`.
2. Telegram sends the summary, the top 10 proposals and a link to an n8n Form.
3. `Wait for Approval (form)` pauses the execution for up to 24 hours.
4. Only `Decision = approve` continues.
   Rows with the approver name and time are appended to `approved_transfers`.
5. `reject`, or no answer within 24 hours, ends at `Rejected or timed out`.
   Nothing is written to `approved_transfers`.

This template never moves stock.
`approved_transfers` is the hand-off list for the people or system that do.
The form is used instead of a plain link so a Telegram link preview cannot approve by accident.

## Credentials needed (live only)

- Google Sheets (OAuth2 or service account)
- Telegram Bot API
- Optional: Header Auth for an LLM API (Anthropic Messages API as configured)

## Node map

```
Schedule (08:00) --+
Manual (demo) -----+--> Config --> Mode is live?
                                    |live: Sheets: Read Inventory --+
                                    |demo: Demo Inventory ----------+--> Detect Transfers
 --> Build Summary Prompt --> Use LLM? --yes--> Anthropic: Write Summary --+
                                        --no-----------------------------+--> Merge Summary --> Mode is live? (output)
   live: Proposal Rows --> Sheets: Append Proposals
   live: Telegram: Ask Approval --> Wait for Approval (form) --> Approved? --yes--> Approved Rows --> Sheets: Append Approved
                                                                          --no ---> Rejected or timed out
   demo: Demo Output
```

## Customize

- Hero list, caps and donor rules: `Config`.
- Different inventory source (ERP, POS, database): replace `Sheets: Read Inventory` with any node that returns `store, sku, size, on_hand, sold_30d`.
- Ranking: change the `gaps.sort` line in `Detect Transfers` (for example to weight by margin).
- Approval via Telegram reply instead of a form: replace the Wait node with a Telegram "Send and Wait for Response" node.
- Timeout: change `resumeAmount` and `resumeUnit` on the Wait node.

## Limitations

- Uses a 30-day sales window only.
  It does not model seasonality, inbound purchase orders or transfers already in transit.
- One source per proposal.
  If the warehouse has 1 unit and the target is 2, the proposal is for 1 unit.
- Transfer cost and minimum shipment size are not modelled.
- The LLM output is accepted only if it has exactly 5 lines and all numbers appear in the facts.
  It is not checked for wording.
- The Wait node form settings were checked against the n8n source (resume option `form`, limit-wait parameters), but the form field shape was not run in an instance.
- The Google Sheets and Telegram nodes and the Anthropic HTTP call were not run against a live instance when this template was written.
- The offline validator in `n8nctl` 1.5 has an outdated catalog and reports the `form` resume option as a warning.
