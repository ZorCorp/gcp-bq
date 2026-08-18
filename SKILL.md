---
name: billing-cost
description: Use when the user asks about GCP cloud cost / billing spend for the org's reseller billing exports — price and cost by billing account, project, service, or SKU, for one month or a month range. You pass the reseller's BigQuery table id per call. Reads a read-only billing export behind a private Cloud Run endpoint by signing a Google ID token with gcloud and calling it over curl (no MCP server / proxy / binary).
---

# Billing Cost (GCP BigQuery)

Query GCP **price & cost** — by **billing account → project → service → SKU**, for **one month or a
month range** — from a BigQuery billing export behind a **private Cloud Run** endpoint. Read-only.

Each reseller has its OWN export table (in its own region/currency). **Every call queries ONE
reseller** — you pass its table id as `bq_table`. Get the legal ids from `list_resellers`;
never invent or guess one.

No MCP server, proxy, or binary. Sign a short-lived Google ID token with `gcloud`, `curl` the endpoint.

## Step 1 — get the reseller allow-list

Call **`list_resellers`** (no arguments) before anything else. It returns one row per
reseller — `reseller`, `currency`, `bq_table`, `region` — and every later call must pass one
of those `bq_table` values verbatim. The endpoint rejects anything else, so guessing an id
just wastes a round trip. The list is short and changes rarely; fetch it once per session and
reuse it. If the user names a reseller ("TW", "the Singapore one"), match it against the
`reseller` column rather than asking them for a table id.

## Setup

- `gcloud` installed; your account needs Cloud Run `run.invoker` (ask an admin on `HTTP 403`).

## Step 0 — ensure gcloud is logged in (do this first)

Run this yourself — do **not** ask the user to log in manually:
```bash
gcloud auth print-identity-token >/dev/null 2>&1 || gcloud auth login
```

## How to query

Endpoint: `https://bq-gcp-billing-586459078049.asia-east2.run.app/mcp/billing`

Use exactly this path — it serves the 6 tools below and nothing else.

```bash
TOKEN=$(gcloud auth print-identity-token)
curl -s -X POST https://bq-gcp-billing-586459078049.asia-east2.run.app/mcp/billing \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"total_cost","arguments":{"bq_table":"<a bq_table from list_resellers>","from_month":"202607","to_month":"202607"}}}'
```
Rows are in `result.content[].text` (JSON). To print: `| python3 -c 'import sys,json;print("\n".join(c["text"] for c in json.load(sys.stdin)["result"]["content"]))'`

## Tools

Every tool takes `bq_table` + a **month range** `from_month`/`to_month` (`YYYYMM`; single month = same).
Every row returns `currency`, `list_price` (public list), `gross_cost` (before credits),
`total_credits` (negative), `net_cost` (after credits = actual cost).

| Tool | Breakdown | Arguments |
|---|---|---|
| `list_resellers` | the allow-list of reseller export tables — **call this first** | *(none)* |
| `total_cost` | per month + per-currency `TOTAL` row | `bq_table, from_month, to_month` |
| `cost_by_billing_account` | per billing account | `bq_table, from_month, to_month` |
| `cost_by_project` | per project | `bq_table, from_month, to_month, billing_account, limit` |
| `cost_by_service` | per service | `bq_table, from_month, to_month, billing_account, limit` |
| `cost_by_sku` | per SKU (finest) | `bq_table, from_month, to_month, billing_account, project, limit` |

- `billing_account` / `project`: `""` for all, or an id to drill down (e.g. SKUs of one project).
- `limit`: how many rows individually; the rest fold into a `~ All other …` rollup. Keep **≤ 950**.

## Rules

1. **Pick the reseller → pass its `bq_table`** (only ids from the list above; never invent one).
2. **Report `net_cost`** (after credits) unless asked otherwise. `list_price` is public list; not margin.
3. **Month = `YYYYMM`.** Range e.g. May–July 2026 → `from_month:"202605", to_month:"202607"`.
4. **Org-wide / "all resellers" total** = call each reseller's `total_cost` and add up — but **only add
   within the same `currency`**. Never sum across different currencies; report per currency.
5. A `~ All other …` row is a rollup — include it when summing; never drop it.
6. A month with no data returns **empty** (not an error) — e.g. a reseller that started recently.
7. Read-only. Cannot change data. A very wide range may hit the per-query byte cap → split it.

## Examples

(`<id>` below means a `bq_table` value you got from `list_resellers`.)

- "<reseller> total in June" → `total_cost(bq_table=<id>, 202606, 202606)`.
- "Total across all resellers in July" → call `total_cost` once per row from `list_resellers`
  (202607), then sum only rows sharing a `currency`.
- "Which SKUs cost most in project <p> for <reseller> in June" →
  `cost_by_sku(bq_table=<id>, 202606, 202606, billing_account:"", project:"<p>", limit:50)`.
- "<reseller> cost per service in July" →
  `cost_by_service(bq_table=<id>, 202607, 202607, billing_account:"", limit:45)`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `HTTP 403` | account lacks `run.invoker`; ask an admin |
| `gcloud ... print-identity-token` error | not logged in — run `gcloud auth login` (Step 0) |
| empty result | that reseller has no data for that month (e.g. just onboarded) |
| bytes-billed limit error | range too wide (per-query cap) — split into smaller ranges |
