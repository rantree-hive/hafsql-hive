# hafsql-hive

A Claude Code skill for building correct, performant queries against **HafSQL** — the PostgreSQL layer over HAF-indexed Hive blockchain data.

## Install

```bash
npx skills add rantree-hive/hafsql-hive
```

## What it covers

- Which HafSQL table to use for each operation type — including the **virtual-op trap** (`haf_operations` silently returns 0 rows for reward ops)
- HafSQL utility functions: `id_from_timestamp`, `get_timestamp`, `to_json`
- Multi-window aggregation with `id_from_timestamp` (30d / 7d / all-time in one query)
- `operation_comment_reward_table` timing trap (fires at payout, not post creation — use 30d window)
- VESTS → HP conversion via `hafsql.dynamic_global_properties.vests_per_hive`
- Vote value / rshares formula — the **correct** `* 1_000_000 / 25` scaling factor (empirically verified 2026-11); common wrong factors and their symptoms
- RC% calculation with mana regeneration forward-projection (raw `current_mana` is stale)
- `rc_api.find_rc_accounts` correct params form: plain object, not array-wrapped
- HAFAH broken fields: `reputation: 0`, `comment_count: 0` from `condenser_api.get_accounts` — correct sources
- `hive_accounts_info_view.post_count` includes comments — use `hafsql.comments WHERE parent_author=''` for root-only count
- Profile metadata from `hivemind_app.hive_accounts_info_view.posting_json_metadata`
- Precision-aware `parseAmt` for both old string and new asset-object API formats
- Schema layers: hafd layer-0 raw tables, generic views, dedicated op tables, and the `op_type` name format
- Per-voter parallel scan for vote queries over many accounts (avoids slow bitmap scans on `voter = ANY(...)`)
- Vote value from actual rshares via `hivemind_app.hive_votes_view`
- **milli-VESTS scaling** in `hafsql.balances.vests` and `hafsql.accounts.incoming_vests` / `outgoing_vests`
- Operation timeline with `haf_account_operations` cursor pagination, plus reading `custom_json` via the hafd account-operation index
- `haf_operations.body` wrapping on HAFAH nodes (`body?.value`)
- `hafsql.comments` payout columns: `SPLIT_PART` string format, tags JSONB filter, active vs. paid-out detection, sub-account `LIKE` pattern
- Production pool setup with circuit breaker, retry, and `statement_timeout`
- Correct `custom_json` column (`json` text → `hafsql.to_json(json)`) and op-type registry names
- **Comment edits are separate ops**: `DISTINCT ON` dedup in `operation_comment_view`, stale `hafsql.comments.body`, bounding comment scans by `created`
- DHF / proposal payouts via `operation_proposal_pay_table` (a virtual op, not a transfer)
- Post community/category resolution through hivemind joins, plus snap/microblog community IDs
- Index cheat-sheet: which filters hit an index on transfers, votes, `hafd.account_operations`, and `hafd.operations`
- Dashboard query patterns: `SUM() OVER()` totals before `LIMIT`, per-account bounds with `unnest`, chunked scans, memo matching, new-account discovery
- Pitfalls checklist

## Public instance

A read-only public HafSQL instance is available for development and testing:

```
host=hafsql-sql.mahdiyari.info  port=5432  dbname=haf_block_log
user=hafsql_public  password=hafsql_public  sslmode=disable
```

Maintained by @mahdiyari. Use indexed queries and cache results — it's a shared node.

## Usage

Once installed, Claude applies these patterns automatically when writing backend code that queries HafSQL or calls `condenser_api` / `rc_api`.

Trigger explicitly:

```
/hafsql-hive
```
