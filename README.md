# hive-skills

Claude Code skills for building on the Hive blockchain.

## Install

```bash
npx skills add rantree-hive/hive-skills
```

## Skills

### `hafsql-hive`

Build correct, performant queries against **HafSQL** — the PostgreSQL layer over HAF-indexed Hive blockchain data.

Covers:
- Which HafSQL table to query for each operation type (virtual vs non-virtual ops)
- Multi-window aggregation pattern with `id_from_timestamp`
- VESTS → HP conversion
- Vote value / rshares formula (and the common scaling bug that gives $0.00)
- RC% calculation from `rc_api`
- Hive API response parsing (`parseAmt` with precision handling)
- Pitfalls checklist

## Usage

Once installed, Claude will automatically apply these patterns when writing backend code that queries HafSQL or calls `condenser_api` / `rc_api`.

You can also trigger it explicitly:

```
/hafsql-hive
```
