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
- Pitfalls checklist

## Usage

Once installed, Claude applies these patterns automatically when writing backend code that queries HafSQL or calls `condenser_api` / `rc_api`.

Trigger explicitly:

```
/hafsql-hive
```
