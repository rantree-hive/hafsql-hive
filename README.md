# hafsql-hive

A Claude Code skill for building correct, performant queries against **HafSQL** — the PostgreSQL layer over HAF-indexed Hive blockchain data.

## Install

```bash
npx skills add rantree-hive/hafsql-hive
```

## What it covers

- Which HafSQL table to use for each operation type — including the **virtual-op trap** (`haf_operations` silently returns 0 rows for reward ops)
- Multi-window aggregation with `id_from_timestamp` (30d / 7d / all-time in one query)
- VESTS → HP conversion via `hafsql.dynamic_global_properties.vests_per_hive`
- Vote value / rshares formula — the `* 1_000_000 / 100` scaling factor that prevents `$0.00` values
- RC% calculation from `rc_api.find_rc_accounts`
- Precision-aware `parseAmt` for both old string and new asset-object API formats
- Pitfalls checklist

## Usage

Once installed, Claude applies these patterns automatically when writing backend code that queries HafSQL or calls `condenser_api` / `rc_api`.

Trigger explicitly:

```
/hafsql-hive
```
