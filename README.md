# gcp-bq

Query **GCP price & cost** — by **billing account → project → service → SKU**, for **one month or a
month range** — from BigQuery billing exports behind a **private Cloud Run** endpoint. Read-only.

Each reseller has its own export table (own region / currency). **Every call queries ONE reseller** —
you pass its table id as `bq_table`. No MCP server, proxy, or binary; the agent signs a Google ID
token with `gcloud` and `curl`s the endpoint. You only need `gcloud` with Cloud Run `run.invoker`.

## Reseller table ids

They are **not** in this repo. `list_resellers` returns them at call time — reseller, currency,
`bq_table`, region — so the ids live only in the private deployment and behind the endpoint,
which is IAM-gated. Anyone who can call the tool already has access to the data.

Adding a reseller therefore needs no change here and no server redeploy of the query tools —
add it to `list_resellers` and grant the reader service account `dataViewer` on the new
project.

## Tools

Every tool takes `bq_table` + a month range (`from_month`/`to_month`, `YYYYMM`; single month = same).
Rows return `currency`, `list_price`, `gross_cost`, `total_credits`, `net_cost`.

| Tool | Breakdown |
|---|---|
| `list_resellers` | the allow-list of reseller export tables — call first |
| `total_cost` | per month + per-currency total |
| `cost_by_billing_account` | per billing account |
| `cost_by_project` | per project (optional `billing_account`) |
| `cost_by_service` | per service (optional `billing_account`) |
| `cost_by_sku` | per SKU (optional `billing_account` / `project`) |

Long lists return top `limit` rows + one `~ All other …` rollup so the total stays complete.

## Quick start

1. `gcloud auth login`.
2. Ask your agent about billing cost (e.g. "cost for <reseller> in July"). The skill looks the
   reseller up with `list_resellers`, picks its table and the right tool.
   See **[`SKILL.md`](./SKILL.md)**.

## Notes

- **Currency**: results are native (no conversion). Never sum across different currencies — for an
  org-wide total the agent adds per currency.
- **Cost & safety**: each query is `_PARTITIONTIME`-pruned (~1.5-month window) and capped at 64 GiB
  (`maximumBytesBilled`); typical query ~$0.002–0.16.
- **Margin caveat**: BQ knows your **cost** and Google's **list price**, not your sell price.

## Changelog

### 2.1.0
- Multi-reseller via a `bq_table` **template parameter** — one tool set queries any reseller export
  (any region/currency); adding a reseller needs no change to this plugin.
- `currency` returned on every row; never merge currencies.

### 2.0.0
- Rebuilt to 5 drill-down tools (price + cost by account/project/service/sku), month-range params,
  partition pruning + `maximumBytesBilled` cost guards. Renamed `gcp-bq-mcp` → `gcp-bq`.
