---
name: hafsql-hive
description: >
  Build correct, performant queries against HafSQL — the PostgreSQL layer over
  HAF-indexed Hive blockchain data. Covers which tables to query for each
  operation type, the rshares/vote-value formula, VESTS-to-HP conversion
  (vests_per_hive is per-block and VESTS-per-HIVE), time-range filtering
  with id_from_timestamp, production pool setup with circuit breaker,
  operation timeline cursor pagination, haf_operations body wrapping, comment
  edit dedup, DHF proposal payouts, hivemind community joins, index-aware
  query patterns for dashboards, and Hive API response parsing patterns.
  Trigger whenever writing backend code that queries HafSQL or calls
  condenser_api / rc_api for a Hive application.
---

# HafSQL × Hive — Query & API Skill

## 0  Connecting to the public HafSQL instance

A read-only public instance is maintained by @mahdiyari. Use it to test queries
or as the data source for Hive applications without running your own HAF node.

```
Host:     hafsql-sql.mahdiyari.info
Port:     5432
Database: haf_block_log
User:     hafsql_public
Password: hafsql_public
SSL:      false (not required)
```

**psql one-liner:**
```bash
psql "host=hafsql-sql.mahdiyari.info port=5432 dbname=haf_block_log user=hafsql_public password=hafsql_public sslmode=disable"
```

**Node.js (pg):**
```js
const { Pool } = require('pg');
const pool = new Pool({
  host:     'hafsql-sql.mahdiyari.info',
  port:     5432,
  database: 'haf_block_log',
  user:     'hafsql_public',
  password: 'hafsql_public',
  ssl:      false,
});
```

**Environment variables (.env):**
```
HAFSQL_HOST=hafsql-sql.mahdiyari.info
HAFSQL_PORT=5432
HAFSQL_DB=haf_block_log
HAFSQL_USER=hafsql_public
HAFSQL_PASSWORD=hafsql_public
HAFSQL_SSL=false
```

> This is a shared public node — avoid full-table scans and always use
> `id_from_timestamp` / indexed filters (see §3). Cache results on your side.


## 1  Schema map — which table does what

HafSQL has three layers. Always prefer the **dedicated operation tables** (layer 2) over the generic views (layer 1) for any operation-specific work.

### Layer 0 — raw HAF tables (direct blockchain index)

| Table | Key columns | Use for |
|-------|------------|---------|
| `hafd.accounts` | `id` (int), `name` | Account id↔name mapping for joins |
| `hafd.account_operations` | `account_id`, `operation_id`, `account_op_seq_no` | Account operation index; use for custom_json timeline or cursor-paginated operation history |

### Layer 1 — generic views (slow, general)

| View | Use for |
|------|---------|
| `hafsql.haf_account_operations` | Account–op index with account_name (`account_id`, `account_name`, `operation_id`) |
| `hafsql.haf_operations` | All ops, **real and virtual** (`id`, `op_type_id`, `timestamp`, `body` JSONB as `{type, value}`) — a view over `hafd.operations` |
| `hafsql.haf_operation_types` | Op type registry (`id`, `name`, `is_virtual`) |

**`haf_operations` includes virtual operations** (author/curation/benefactor rewards, producer_reward, fill_vesting_withdraw, proposal_pay, …) — verified 2026-11: the same 76 `author_reward` ops in one hour as `operation_author_reward_table`. There is **no `hafsql.operations` view**. Still prefer the layer-2 tables below: they are indexed by account and have typed columns, while filtering `haf_operations` by JSONB body fields is slow.

**Op type name format:** `haf_operation_types.name` is prefixed with `hive::protocol::` (e.g. `hive::protocol::vote_operation`). Strip for display: `.replace(/^hive::protocol::/, '')`. Filter with the full prefixed name. Skip `hive::protocol::effective_comment_vote_operation` — it is noisy and fires on every vote.

**Op type registries:** use `hafsql.haf_operation_types` (or `hafd.operation_types`).
There is **no `hafsql.operation_types`** — a query against it errors, and if it is
wrapped in try/catch it fails silently.

### Layer 2 — dedicated operation tables (fast, correct)

Use these instead of JSONB body parsing via `haf_operations` (faster, typed, account-indexed):

| Table | Filter column | Key value columns |
|-------|--------------|-------------------|
| `hafsql.operation_author_reward_table` | `author` | `hive_payout`, `hbd_payout`, `vesting_payout` (all numeric) |
| `hafsql.operation_curation_reward_table` | `curator` | `reward` (numeric VESTS) |
| `hafsql.operation_comment_benefactor_reward_table` | `benefactor` | `hive_payout`, `hbd_payout`, `vesting_payout` (numeric) |
| `hafsql.operation_comment_reward_table` | `author` | `payout`, `author_rewards`, `total_payout_value`, … |
| `hafsql.operation_transfer_table` | `from_account`, `to_account` | `amount`, `symbol`, `memo` |
| `hafsql.operation_vote_view` | `voter` or `author` | `weight` (negative = downvote), `timestamp` |
| `hafsql.operation_comment_view` | `author` | `permlink`, `parent_author`, `body`, `timestamp` |
| `hafsql.operation_fill_vesting_withdraw_table` | `from_account` / `to_account` | `withdrawn`, `deposited` |
| `hafsql.operation_delegate_vesting_shares_table` | `delegator` / `delegatee` | `vesting_shares` |
| `hafsql.operation_transfer_to_vesting_table` | `from_account` / `to_account` | `amount` |
| `hafsql.operation_custom_json_view` | `custom_id`; per-account via `hafd.account_operations` (§12) | `json` (**text**, not `payload`) → `hafsql.to_json(json)`, `required_auths`, `required_posting_auths` (JSONB) |
| `hafsql.operation_proposal_pay_table` | `receiver`, `proposal_id` | `payment`, `symbol`, `payer` — DHF payouts, **virtual op** (§17) |
| `hafsql.operation_withdraw_vesting_table` | `account` | `vesting_shares` (VESTS), `symbol` — power-down *starts*; **no timestamp column** (use `get_timestamp(id)`); `vesting_shares = 0` rows are cancellations |
| `hafsql.operation_transfer_to_savings_table` | `from_account` / `to_account` | `amount`, `symbol`, `memo` |
| `hafsql.operation_effective_comment_vote_view` | `voter` / `author` | `rshares`, `vote_value`, `pending_payout` — virtual op fired per vote with the realised rshares |

> `operation_comment_view` has **one row per edit** — see §16 before counting or
> reading bodies from it.

### Rich account / global views

| View | Purpose |
|------|---------|
| `hafsql.accounts` | Per-account HP, delegation totals, reputation, last activity, keys (`creator`, `recovery`). VESTS columns are whole VESTS, `*_hp` columns are HP (§11). |
| `hafsql.balances` | Current HIVE / HBD / VESTS / hp_equivalent per account. `account_name` is the join key. `vests` is whole VESTS (§11). |
| `hafsql.dynamic_global_properties` | **One row per block** (view over `hafd.blocks`). Always `ORDER BY block_num DESC LIMIT 1`. `vests_per_hive` = VESTS per HIVE; `total_*` columns are raw integer units (§4) |
| `hafsql.reputations` | `rep` (**already display-scale** — no `rawRepToDisplay` needed), raw `reputation`, `account_name` |
| `hafsql.delegations` | Active delegations with HP equivalent |
| `hafsql.rc_delegations` | `delegator`, `delegatee`, `rc`, `hp_equivalent` |
| `hafsql.proposal_approvals` | `proposal_id`, `voter` — current DHF proposal votes |
| `hafsql.comments` | All authored content (root posts **and** replies). `parent_author = ''` → root posts only. Also `last_edited`, `deleted`, `beneficiaries`, `root_author`/`root_permlink`. Payout columns: see §14. |
| `hivemind_app.hive_accounts_info_view` | `followers`, `following`, `reputation`, `posting_json_metadata`, `post_count` (all authored content, not just root posts), `created_at` (filter for new accounts in a date range) |
| `hivemind_app.hive_votes_view` | Per-vote rshares: `voter`, `author`, `permlink`, `rshares`, `percent`, `weight`, `is_effective`, `num_changes` — use for vote-value computation from actual rshares |
| `hivemind_app.hive_posts` + `hive_permlink_data` + `hive_category_data` | Post → community/category resolution (§18) |

### HafSQL utility functions

| Function | Returns | Use for |
|----------|---------|---------|
| `hafsql.id_from_timestamp(ts, round_up)` | operation id | time-range lower bound in WHERE (§3) |
| `hafsql.get_timestamp(id)` | timestamptz | recover a human timestamp from an op id |
| `hafsql.to_json(json)` | JSONB | parse the custom_json `json` text column to JSONB for field extraction |

```sql
-- get_timestamp: convert op id back to date for grouping
SELECT hafsql.get_timestamp(id)::date AS day, SUM(amount) AS volume
FROM hafsql.operation_transfer_table
WHERE ...
GROUP BY 1 ORDER BY 1;

-- to_json: extract fields from custom_json (the column is `json`, type text)
SELECT (hafsql.to_json(json))->>'subject' AS subject,
       (hafsql.to_json(json))->>'action'  AS action
FROM hafsql.operation_custom_json_view
WHERE custom_id = 'your-app-id'
  AND id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '30 days', true)
  AND (hafsql.to_json(json))->>'action' IN ('create', 'edit');
-- ❌ to_json(payload) — there is no `payload` column on this view
```


## 2  Reward op timing trap — `operation_comment_reward_table`

`operation_comment_reward_table` rows fire at **post payout time (7 days after
creation)**, not at post creation. If you query it with a 7-day window you will
get payouts for posts that were created 14 days ago. Always use a window at least
**30 days** wide when correlating with posting activity in that period.

```sql
-- ✅ 30-day window captures payouts for posts from last ~23 days
WHERE id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '30 days', true)

-- ❌ 7-day window misses payouts for posts from last week
WHERE id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '7 days', true)
```


## 3  Time-range filtering — always use `id_from_timestamp`

All `operation_*` tables use an `id` that is the blockchain operation id.
Never filter by timestamp via a JOIN — use the built-in function instead:

```sql
-- Last 30 days
WHERE r.id > hafsql.id_from_timestamp(
  (NOW() AT TIME ZONE 'UTC') - INTERVAL '30 days', true
)
```

The second argument (`true`) means "round up to the next block", giving an
inclusive lower bound. Use `false` for an exclusive bound.

### Multi-window aggregation pattern (all-time / 30d / 7d in one query)

```sql
WITH t AS (
  SELECT
    hafsql.id_from_timestamp((NOW() AT TIME ZONE 'UTC') - INTERVAL '30 days', true) AS id_30d,
    hafsql.id_from_timestamp((NOW() AT TIME ZONE 'UTC') - INTERVAL '7 days',  true) AS id_7d
)
SELECT
  SUM(hive_payout)                                              AS hive_all,
  SUM(CASE WHEN r.id > t.id_30d THEN hive_payout ELSE 0 END)  AS hive_30d,
  SUM(CASE WHEN r.id > t.id_7d  THEN hive_payout ELSE 0 END)  AS hive_7d
FROM hafsql.operation_author_reward_table r, t
WHERE r.author = $1
```

The `FROM ..., t` is a cross-join; the single-row CTE makes `t.id_30d` /
`t.id_7d` available as constants in the CASE expressions.

### Per-voter parallel scan (downvotes / vote queries over many accounts)

A combined `WHERE voter = ANY($1::text[])` with many voters triggers a slow bitmap
scan. For downvote or vote intelligence queries, **run one query per voter in
parallel** so PostgreSQL uses the direct `(voter, id)` index on each:

```js
const rows = (await Promise.all(
  trackedVoters.map(voter =>
    query(
      `SELECT voter, author, permlink, weight, timestamp
       FROM hafsql.operation_vote_view
       WHERE voter = $1
         AND weight < 0
         AND id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - $2::interval, true)
       ORDER BY id DESC
       LIMIT $3`,
      [voter, interval, perVoterLimit],
      null, 0, 45000
    ).catch(() => [])
  )
)).flat();
```


## 4  VESTS → HP conversion

`hafsql.dynamic_global_properties` is a view over `hafd.blocks`: **one row per
block** (~100 M rows). Two traps:

```sql
-- ✅ Current ratio (fast — block_num is the primary key)
SELECT vests_per_hive FROM hafsql.dynamic_global_properties
ORDER BY block_num DESC LIMIT 1;

-- ❌ LIMIT 1 without ORDER BY returns an arbitrary, possibly years-old block
--    (observed 2026-11: a 2025-08 row, ratio ~4 % off)
-- ❌ Never join/cross-join this view without that ORDER BY … LIMIT 1 subquery —
--    it multiplies your result by every block in history
```

`vests_per_hive` is **VESTS per HIVE** (~1600), so **divide**:

```js
const hp = vests / vestsPerHive;   // ✅  e.g. 174 110 743 VESTS / 1601.33 = 108 728 HP
// ❌ vests * vestsPerHive — 2.5 million times too big
```

The raw `total_vesting_fund_hive` / `total_vesting_shares` columns are **integer
chain units** (milli-HIVE ×1e3 and micro-VESTS ×1e6). Dividing them directly gives
a HIVE-per-VEST ratio **1000× too small**:

```js
const hivePerVest = (row.total_vesting_fund_hive / 1e3) / (row.total_vesting_shares / 1e6);
// or simply: 1 / row.vests_per_hive
```

From the Hive API instead (`condenser_api.get_dynamic_global_properties`):
```js
const hivePerVest = parseAmt(g.total_vesting_fund_hive) / parseAmt(g.total_vesting_shares); // see §7
```

`hafsql.balances.hp_equivalent` and `hafsql.accounts.incoming_hp` / `outgoing_hp`
are already HP — HafSQL computes them with the latest block's ratio.


## 5  Vote value formula (rshares scale)

```js
// Value of one 100 % weight vote at 100 % voting power (verified against peakd, 2026-11)
const rshares100   = effectiveVests * 1_000_000 * 0.02;   // = effectiveVests * 20_000
const voteHive100  = rshares100 / recentClaims * rewardBalance;
const voteValue100Usd = voteHive100 * hivePrice;
```

- `effectiveVests` = `vesting_shares - delegated_vesting_shares + received_vesting_shares` (raw VESTS from `condenser_api`)
- `recentClaims` = `parseFloat(fund.recent_claims)` from `condenser_api.get_reward_fund`
- `rewardBalance` = `parseAmt(fund.reward_balance)` (see §7)
- `hivePrice` = median feed price (`get_current_median_history_price`: `base / quote`, HBD per HIVE)

**Why these constants:**
- `1_000_000`: rshares are computed from VESTS in their integer "satoshi" form.
  VESTS have 6 decimal places, so 1 VEST = 1 000 000 units.
- `0.02`: a 100 % vote spends **2 % of the voter's voting mana** (`used_mana = current_mana × weight / 10000 / 50`).
  rshares come from the mana spent, not from the whole stake.
- The result is the vote's **full** contribution to the post's payout. This is what peakd shows.
  At payout it is split 50/50 between author and curators (post-HF21), and beneficiaries come out of the author half.

> Older versions of this skill wrote this as `* 1_000_000 / 25 * 0.5` and called the `0.5` the "author share".
> `/25 × 0.5` is the same number as `× 0.02`, so the values were right but the explanation was wrong.
> If you want the author-side value specifically, multiply the result by `0.5`.

**Common wrong formulas and their symptoms:**

| Expression for `rshares100` | Symptom |
|-----------------------------|---------|
| `effectiveVests * 1e6 * 0.02` | ✅ correct, matches peakd |
| `effectiveVests * 1e6` (mana fraction omitted) | **50× too large** |
| `effectiveVests * 1e6 * 0.02 * 0.5` | ½ of the displayed value (author share only) |
| `effectiveVests` (1e6 omitted) | 1 000 000× too small, rounds to `$0.0000` |

`voteValue100Usd` is the dollar value of one **100 % weight, 100 % VP** vote.
Scale linearly: `voteValue100Usd * (vpPercent / 100) * (weight / 10000)`.

> The ◆ symbol in peakd's vote value UI is **HBD** (≈ $1 each), not HIVE.
> 0.234 ◆ means $0.234, not $0.234 × hivePrice.

### Vote value from actual rshares (hivemind_app.hive_votes_view)

For a specific cast vote (not the full-weight estimate), look up its rshares
from `hivemind_app.hive_votes_view` and convert directly:

```js
// rs = row.rshares from hive_votes_view (BigInt from DB — coerce with Number())
// The rshares already include the 2 % mana fraction, so don't apply 0.02 again.
const vote_usd = Math.round(
  (Math.abs(rs) / recentClaims) * rewardBalance * hivePrice * 10000
) / 10000 * Math.sign(rs);
// Full vote value, consistent with voteValue100Usd above.
// Multiply by 0.5 for the author-side share only.
```

Batch lookup for a page of vote_operations:

```sql
SELECT v.voter, v.author, v.permlink, v.rshares
FROM hivemind_app.hive_votes_view v
JOIN (VALUES ($1,$2,$3), ($4,$5,$6), ...) AS w(voter,author,permlink)
  ON v.voter = w.voter AND v.author = w.author AND v.permlink = w.permlink
```


## 6  RC percentage

The `rc_manabar.current_mana` field is a **snapshot at `last_update_time`** — it
does not advance in real time. Regenerate forward the same way VP is regenerated:

```js
// rc_api.find_rc_accounts — params MUST be a plain object, NOT wrapped in an array
const rcData = await hiveAPI('rc_api.find_rc_accounts', { accounts: [username] });
// ❌ Wrong:  hiveAPI('rc_api.find_rc_accounts', [{ accounts: [username] }])
//            → "Bad Cast: Invalid cast from type 'array_type' to Object"

const rcAcc = rcData?.rc_accounts?.[0];
if (rcAcc) {
  const rcMax        = parseFloat(rcAcc.max_rc ?? 0);
  const rcStored     = parseFloat(rcAcc.rc_manabar?.current_mana ?? 0);
  const rcLastUpdate = rcAcc.rc_manabar?.last_update_time ?? 0;
  const rcElapsed    = Date.now() / 1000 - rcLastUpdate;
  const rcRegenRate  = rcMax / (5 * 24 * 3600);  // full regen in 5 days
  const rcCurrent    = Math.min(rcMax, rcStored + rcElapsed * rcRegenRate);
  const rcPercent    = rcMax > 0 ? Math.round((rcCurrent / rcMax) * 1000) / 10 : null;
  // null → show "—"
}
```

`max_rc` and `current_mana` are both large-integer strings — always wrap in `parseFloat`.
If `max_rc` is `"0"` or absent, `rcPercent` is null — show `—`.


## 7  Hive API response parsing (`parseAmt`)

`condenser_api` returns asset values in two formats depending on the node:

| Format | Example | `parseAmt` result |
|--------|---------|-------------------|
| Old string | `"227945.374 HIVE"` | `parseFloat(s)` → `227945.374` ✓ |
| New asset object | `{ amount: "227945374", precision: 3, nai: "@@000000021" }` | must divide: `parseFloat(s.amount) / 10 ** s.precision` |

**Always use a precision-aware parseAmt:**

```js
function parseAmt(s) {
  if (s == null) return 0;
  if (typeof s === 'object' && s.amount != null)
    return (parseFloat(s.amount) || 0) / Math.pow(10, s.precision ?? 3);
  return parseFloat(typeof s === 'string' ? s : String(s)) || 0;
}
```

> The bare `parseFloat(s.amount)` pattern (without dividing by precision) will
> give a value 1 000× or 1 000 000× too large for HIVE/HBD/VESTS objects
> respectively, while still computing HP correctly if both `vesting_shares`
> and `total_vesting_shares` are both off by the same factor (the error cancels
> in the ratio). The *vote value formula* will silently inflate.


## 8  Common Hive API calls (backend / Node.js)

```js
async function hiveAPI(method, params) {
  const res = await fetch('https://api.hive.blog', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', method, params, id: 1 }),
    signal: AbortSignal.timeout(10000),
  });
  return (await res.json()).result;
}

// Account data
const [acc] = await hiveAPI('condenser_api.get_accounts', [[username]]);

// Reward fund (for vote-value formula)
const fund = await hiveAPI('condenser_api.get_reward_fund', ['post']);

// Chain globals (vesting ratio, supply)
const globals = await hiveAPI('condenser_api.get_dynamic_global_properties', []);

// RC mana
const rcData = await hiveAPI('rc_api.find_rc_accounts', { accounts: [username] });
```


## 9  Profile metadata — read from hivemind, not Hive API

`hivemind_app.hive_accounts_info_view.posting_json_metadata` is the most
efficient source for profile data (name, bio, location, website, profile_image).
Avoids a separate Hive API call per account:

```js
// In SQL: SELECT posting_json_metadata FROM hivemind_app.hive_accounts_info_view WHERE name = $1
// Then in JS:
function parseProfile(rawMeta) {
  try {
    const meta = typeof rawMeta === 'string' ? JSON.parse(rawMeta) : rawMeta;
    return meta?.profile ?? {};
  } catch { return {}; }
}
const profile = parseProfile(hmRow.posting_json_metadata);
const location = profile.location?.trim() || null;
const name     = profile.name?.trim()     || null;
const website  = profile.website?.trim()  || null;
```

> `json_metadata` (non-posting) is less reliable for recent accounts — always
> prefer `posting_json_metadata`.


## 10  HAFAH broken fields (confirmed 2026-11)

`condenser_api.get_accounts` on HAFAH nodes silently returns zeroed values for
some fields. Do not rely on them:

| Field | HAFAH returns | Correct source |
|-------|--------------|----------------|
| `reputation` | `0` always | `hivemind_app.hive_accounts_info_view.reputation` (primary) or `follow_api.get_account_reputations` (fallback) |
| `comment_count` | `0` always | `hafsql.comments WHERE author=$1` count |
| `post_count` | counts **all** authored content (posts + comments) | `hafsql.comments WHERE author=$1 AND parent_author=''` for root posts only |

### Reputation — correct fetch order

```js
// 1. Primary: hivemind view (fastest, same DB)
const hmRow = await query(
  `SELECT reputation FROM hivemind_app.hive_accounts_info_view WHERE name = $1`,
  [username]
);
const hmRawRep = hmRow[0]?.reputation ?? null;

// 2. Fallback: follow_api (Hive API call)
const repResult = await hiveAPI('follow_api.get_account_reputations',
  { account_lower_bound: username, limit: 1 }
).catch(() => null);
const apiRawRep = repResult?.reputations?.[0]?.name === username
  ? repResult.reputations[0].reputation : null;

const rawRep = hmRawRep ?? apiRawRep ?? acc.reputation; // acc.reputation is 0 on HAFAH
```

Convert raw reputation to the human score (25–80 range):
```js
function rawRepToDisplay(raw) {
  if (!raw || raw === '0') return 25;
  const r = typeof raw === 'string' ? parseFloat(raw) : raw;
  if (!r || r === 0) return 25;
  const neg = r < 0;
  const log = Math.log10(Math.abs(r));
  const score = Math.max(-9, log - 9) * 9 + 25;
  return Math.round((neg ? 50 - (score - 25) : score) * 10) / 10;
}
```

### Post count — root posts only

```sql
-- Counts only root posts (not comments/replies)
SELECT COUNT(*)::int AS root_post_count
FROM hafsql.comments
WHERE author = $1
  AND parent_author = ''
```

`hivemind_app.hive_accounts_info_view.post_count` includes replies — it will
overcount for accounts that comment frequently.


## 11  VESTS units in HafSQL views — all whole VESTS

Every VESTS column in the `hafsql` views and operation tables is in **whole VESTS**
(the views already divide the chain's micro-VESTS by 1 000 000). Verified 2026-11
against the view definitions and data:

| Column | Unit |
|--------|------|
| `hafsql.balances.vests` | VESTS |
| `hafsql.accounts.incoming_vests` / `outgoing_vests` / `vesting_withdraw_rate` | VESTS |
| `operation_*_reward_table.vesting_payout`, `operation_curation_reward_table.reward` | VESTS |
| `operation_withdraw_vesting_table.vesting_shares` | VESTS |
| `hafsql.dynamic_global_properties.total_vesting_shares` | **micro-VESTS (raw)** — the exception |

```js
const [{ vests_per_hive }] = await query(
  `SELECT vests_per_hive FROM hafsql.dynamic_global_properties ORDER BY block_num DESC LIMIT 1`);
const effectiveVests = +row.vests - +row.outgoing_vests + +row.incoming_vests;
const hp = effectiveVests / +vests_per_hive;
```

> Older versions of this skill said these columns were **milli-VESTS** and multiplied
> them by 1000. That only produced correct HP because the ratio was computed from the
> raw `total_*` columns, which is 1000× too small (§4) — the two errors cancelled.
> Fix both together if you are migrating code.


## 12  Operation timeline — cursor pagination on `account_op_seq_no`

Paginate by `account_op_seq_no` alone, picking the page in a `MATERIALIZED` CTE so
PostgreSQL walks the unique index `(account_id, account_op_seq_no)` backwards and
joins only the 51 rows it keeps:

```sql
WITH page AS MATERIALIZED (
  SELECT ao.account_op_seq_no, ao.operation_id
  FROM hafd.account_operations ao
  WHERE ao.account_id = (SELECT id FROM hafd.accounts WHERE name = $1)
    AND ao.op_type_id <> 72                 -- effective_comment_vote: noisy, fires on every vote
    AND ao.account_op_seq_no < $2           -- cursor: omit on first page
  ORDER BY ao.account_op_seq_no DESC
  LIMIT 51                                  -- hasMore = rows.length === 51; serve 50
)
SELECT p.account_op_seq_no AS seq_no, o.id, o.timestamp, ot.name AS op_type, ot.is_virtual, o.body
FROM page p
JOIN hafsql.haf_operations o       ON o.id  = p.operation_id
JOIN hafsql.haf_operation_types ot ON ot.id = o.op_type_id
ORDER BY p.account_op_seq_no DESC
```

```js
const hasMore    = rows.length === 51;
const ops        = rows.slice(0, 50);
const nextCursor = hasMore ? ops[ops.length - 1].seq_no : null;
// For a date range: keep paging until ops[ops.length-1].timestamp < rangeStart.
```

> ❌ Do **not** bound this query with `ao.operation_id >= id_from_timestamp(...)`.
> There is no index on `(account_id, operation_id)`, so it becomes a filter over the
> account's entire history — 20 s+ timeouts for active accounts (verified 2026-11 on
> an account with ~2 M ops). The seq_no version above runs in ~350 ms.

The `account_op_seq_no` cursor is stable even when new ops arrive during pagination.

### Reading custom_json via account operation index (hafd layer 0)

When you need custom_json ops for a specific account+custom_id from the full
history (not just recent), join through `hafd.accounts` → `hafd.account_operations`.
The `op_type_id = 18` filter matters: without it the join times out for active
accounts; with it, ~1.7 s on the public node.

> Built-in ids like `follow` and `reblog` carry a JSON **array**
> (`["follow", {...}]`), so read fields with `->1->>'following'`, not `->>'following'`.

```sql
WITH account_notes AS (
  SELECT
    v.id,
    v.json   AS payload,
    v.timestamp
  FROM hafd.accounts a
  JOIN hafd.account_operations ao ON ao.account_id = a.id
                                 AND ao.op_type_id = 18   -- custom_json: uses the (account_id, op_type_id) index
  JOIN hafsql.operation_custom_json_view v ON v.id = ao.operation_id
  WHERE a.name = 'your-account'
    AND v.custom_id = 'your-app-id'
)
SELECT
  (hafsql.to_json(payload))->>'subject' AS subject,
  (hafsql.to_json(payload))->>'action'  AS action
FROM account_notes
WHERE (hafsql.to_json(payload))->>'action' IN ('create', 'edit')
  AND (hafsql.to_json(payload))->>'subject' = ANY($1::text[])
```


## 13  `haf_operations.body` — always `{type, value}`

The `hafsql.haf_operations` view builds `body` as
`jsonb_build_object('type', <op name without prefix>, 'value', <op body>)`, so the
operation fields are **always** one level down:

```js
const op = row.body.value;      // ✅ op.voter, op.author, op.permlink
// ❌ row.body.voter — undefined
```

```sql
SELECT body->'value'->>'voter' FROM hafsql.haf_operations WHERE ...
```

(`row.body?.value ?? row.body` is a safe pattern if the same code also handles
op bodies from the Hive API, which are not wrapped this way.)


## 14  `hafsql.comments` payout columns

On the current public node the payout columns in `hafsql.comments` are `numeric`
(verified 2026-11). Older HafSQL versions returned `"N.NNN HBD"` strings, where a
direct `::float` cast fails. If your code must run against several nodes,
`SPLIT_PART(col::text, ' ', 1)::numeric` works for **both** forms:

```sql
-- In SQL: extract numeric value with SPLIT_PART
SELECT
  permlink,
  title,
  created,
  cashout_time,
  SPLIT_PART(pending_payout_value::text, ' ', 1)::numeric  AS pending_payout,
  SPLIT_PART(total_payout_value::text,   ' ', 1)::numeric  AS total_payout,
  SPLIT_PART(curator_payout_value::text, ' ', 1)::numeric  AS curator_payout
FROM hafsql.comments
WHERE author = $1
  AND parent_author = ''
  AND SPLIT_PART(pending_payout_value::text, ' ', 1)::numeric > 0.001
ORDER BY pending_payout DESC
LIMIT 100
```

```js
// In JS: simple string parse (HBD-string format)
const pendingPayout = parseFloat(row.pending_payout_value) || 0;
// parseFloat("1.234 HBD") === 1.234 ✓
```

**Active vs paid-out posts:** A post is active (within payout window) when
`created > NOW() - INTERVAL '7 days'` or `cashout_time > NOW()`. Use `pending_payout_value`
for active posts and `total_payout_value + curator_payout_value` for paid-out posts.

**Tags filtering:** `comments.tags` is a JSONB array. Filter with the containment
operator:
```sql
WHERE tags @> jsonb_build_array($1::text)   -- posts with a specific tag
-- or
WHERE category = $1                          -- posts in a community / main tag
```

**Sub-account query pattern:** To include dot-prefixed sub-accounts (e.g. `user.curation`):
```sql
WHERE (from_account = $1 OR from_account LIKE $1 || '.%')
```


## 15  Production pool setup (circuit breaker + retry)

For applications connecting to the public node or any HafSQL instance under load,
use a pool with connection limits, a per-query statement timeout, retry logic for
transient errors, and a circuit breaker to stop hammering during outages:

```js
const { Pool } = require('pg');
const NodeCache = require('node-cache');

const cache = new NodeCache({ stdTTL: 300, checkperiod: 60 });

const pool = new Pool({
  host:                   process.env.HAFSQL_HOST,
  port:                   parseInt(process.env.HAFSQL_PORT || '5432'),
  database:               process.env.HAFSQL_DB,
  user:                   process.env.HAFSQL_USER,
  password:               process.env.HAFSQL_PASSWORD,
  max:                    5,       // respect shared-node connection limit
  min:                    0,
  idleTimeoutMillis:      5000,
  connectionTimeoutMillis: 5000,   // fail fast so circuit trips quickly
  allowExitOnIdle:        true,
  ssl:                    process.env.HAFSQL_SSL === 'true'
                            ? { rejectUnauthorized: false } : false,
});

pool.on('error', err => console.error('[HafSQL] Pool error:', err.message));

// Circuit breaker — 5 consecutive failures → open for 60 s
const circuit = { failures: 0, openUntil: 0 };
function circuitOpen() {
  if (circuit.failures < 5) return false;
  if (Date.now() >= circuit.openUntil) { circuit.failures = 0; return false; }
  return true;
}

const RETRYABLE = err =>
  err.message?.includes('terminated')   ||
  err.message?.includes('Connection')   ||
  err.message?.includes('timeout exceeded') ||
  err.code === 'ECONNRESET'             ||
  err.code === 'ENOTFOUND'              ||
  err.code === 'ECONNREFUSED';

// max_client_conn = server is at capacity; don't retry, trip circuit immediately
const FAIL_FAST = err => err.message?.includes('max_client_conn');

async function query(sql, params = [], cacheKey = null, ttl = 300, timeout = 15000) {
  if (cacheKey) {
    const hit = cache.get(cacheKey);
    if (hit !== undefined) return hit;
  }
  if (circuitOpen()) throw new Error('HafSQL unavailable — retrying in a moment');

  let lastError;
  for (let attempt = 1; attempt <= 3; attempt++) {
    let client;
    try {
      client = await pool.connect();
      await client.query(`SET statement_timeout = ${timeout}`);
      const result = await client.query(sql, params);
      circuit.failures = 0;
      if (cacheKey) cache.set(cacheKey, result.rows, ttl);
      return result.rows;
    } catch (err) {
      lastError = err;
      if (FAIL_FAST(err)) {
        circuit.failures++;
        circuit.openUntil = Date.now() + 60000;
        throw err;
      }
      if (RETRYABLE(err)) {
        circuit.failures++;
        circuit.openUntil = Date.now() + 60000;
        await new Promise(r => setTimeout(r, attempt * 1000));
        continue;
      }
      throw err;
    } finally {
      try { client?.release(); } catch {}
    }
  }
  throw lastError;
}

module.exports = { query, cache };
```

> Keep `max: 5` when targeting the public node — it has limited connection slots.
> Use `statement_timeout` per query (not globally) so heavy queries time out
> without killing the connection for others.


## 16  Comment edits, stale bodies, and bounding `hafsql.comments` scans

**Every edit is a new op.** `hafsql.operation_comment_view` holds one row per
`comment_operation`, and editing a post or reply broadcasts another one with the
same `author` + `permlink`. Counting rows overcounts, and reading `body` may give
an old version. Deduplicate to the latest version with `DISTINCT ON`:

```sql
SELECT DISTINCT ON (c.permlink)
  c.author, c.permlink, c.parent_author, c.parent_permlink, c.json_metadata,
  hafsql.get_timestamp(c.id) AS created_at        -- timestamp of the latest edit
FROM hafsql.operation_comment_view c
WHERE c.author = $1
  AND c.parent_author != ''                       -- replies only
  AND c.id > hafsql.id_from_timestamp(NOW() AT TIME ZONE 'utc' - INTERVAL '90 days', true)
ORDER BY c.permlink, c.id DESC                    -- DESC → newest version wins
LIMIT 15000
```

Fetch bodies separately for only the subset you need (`LEFT(body, 500)` keeps rows small).

**`hafsql.comments.body` can be stale** for very recent content: it may still
hold the original body before the author edited it. When an edit matters (e.g.
detecting a mention added after posting), re-fetch the latest body from a Hive
API node (`condenser_api.get_content`) for recent posts with no match in HafSQL.

**Bound `hafsql.comments` by `created`.** `comments.id` is not an op id, so
`id_from_timestamp` does not apply. Filter on `created` (and `author`) instead,
and always add an upper age bound for multi-account lookups:

```sql
SELECT author, MAX(created) AS last_post
FROM hafsql.comments
WHERE author = ANY($1::text[])
  AND created > NOW() - INTERVAL '3 years'       -- avoids a full scan for inactive accounts
GROUP BY author
```


## 17  DHF payouts — `operation_proposal_pay_table`

Hive Decentralized Fund (DHF / proposal) payouts are **virtual ops**
(`proposal_pay_operation`), not transfers. They never appear in
`operation_transfer_table`; use the dedicated table:

```sql
SELECT
  payment::float                        AS amount,
  proposal_id,
  hafsql.get_timestamp(id)              AS ts,
  hafsql.get_timestamp(MIN(id) OVER())  AS first_inflow,
  COUNT(*) OVER()                       AS total_payment_count,
  SUM(payment) OVER()::float            AS total_received_all
FROM hafsql.operation_proposal_pay_table
WHERE payer = 'hive.fund'
  AND receiver = $1
  AND symbol = 'HBD'
  AND proposal_id != 0         -- proposal 0 is the Return Proposal (funds back to the DHF)
ORDER BY id DESC
LIMIT 200
```

- No timestamp column: use `hafsql.get_timestamp(id)`.
- Group by `proposal_id, receiver`, not just `receiver`: one account can run several proposals.
- Current proposal votes are in `hafsql.proposal_approvals`; live proposal state
  (daily_pay, funded vs. unfunded) comes from `condenser_api.list_proposals`. Proposals
  ranked below the Return Proposal's vote total receive nothing.


## 18  Community / category of a post — hivemind join

`hafsql.comments.category` gives the root post's main tag or community, but for
data from `operation_comment_view` (or to classify replies by the community of
their root post), join through hivemind:

```sql
SELECT b.*, hcd.category
FROM base b                                         -- e.g. the DISTINCT ON CTE from §16
JOIN hivemind_app.hive_accounts      ha  ON ha.name        = b.author
JOIN hivemind_app.hive_permlink_data hpd ON hpd.permlink   = b.permlink
JOIN hivemind_app.hive_posts         hp  ON hp.author_id   = ha.id
                                        AND hp.permlink_id = hpd.id
LEFT JOIN hivemind_app.hive_category_data hcd ON hcd.id    = hp.category_id
```

**Microblog / snap detection:** classify by community id, not by `parent_permlink`
patterns. Replies to another user's snap carry a random permlink with no marker.

| Community | Platform |
|-----------|----------|
| `hive-124838` | PeakSnaps / Snapie |
| `hive-125125` | Ecency Waves |
| `hive-115814` | Sportsbites |
| `hive-167922` | LeoFinance (mixed blog and threads, so detect LeoThreads by `json_metadata.app` starting with `leothreads` or by `parent_author`) |


## 19  Index cheat-sheet — write queries that hit these

| Table | Useful indexes | Implication |
|-------|---------------|-------------|
| `operation_transfer_table` | `(from_account, id)`, `(to_account, id)`, `(memo, id)`, `(amount)` | Always filter on an account **or** memo, plus `id >` a bound. Memo lookups (deposit IDs) are indexed |
| `operation_vote_view` | `(voter, id)`, `(author, id)` | One voter per query (§3 parallel scan) |
| `operation_*_table` (most) | `(<account col>)`, unique `(id)` | Account filter first, `id_from_timestamp` for time |
| `operation_proposal_pay_table` | `(receiver)`, `(proposal_id)` | |
| `hafd.account_operations` | unique `(account_id, account_op_seq_no)`, `(account_id, op_type_id)` | Cursor pagination (§12); per-account op-type scans |
| `hafd.operations` | partial on `body_value->>'author'` (+ `permlink`) for `op_type_id IN (0, 1, 72)` | Author/permlink lookups on vote (0), comment (1), and effective_comment_vote (72) are indexed. Other body fields are not |
| `hafd.operation_types` | unique `(name)` | |

A filter on an unindexed column (e.g. `memo LIKE '%x%'`, `amount::text`, JSONB
fields other than the above) over a wide id range will time out on the public node.


## 20  Query patterns for multi-account and dashboard work

### Full-history totals + a limited page in one query

Window functions are evaluated **before** `LIMIT`, so `SUM(...) OVER()` returns the
total over all matching rows while only N rows are returned (see the §17 query).
That saves a separate `COUNT/SUM` query.

### Per-account time bounds with parallel `unnest`

When each account needs its own window (e.g. "first 90 days after account
creation"), pass parallel arrays and join on them:

```sql
SELECT c.author, LEFT(c.body, 20000) AS body, c.created AS ts
FROM (SELECT unnest($1::text[])        AS author,
             unnest($2::timestamptz[]) AS account_created) accs
JOIN hafsql.comments c ON c.author = accs.author
WHERE c.parent_author = ''
  AND c.created >= accs.account_created
  AND c.created <= accs.account_created + ($3 || ' days')::interval
ORDER BY c.author, c.created ASC
```

For thousands of accounts, run it in **chunks of ~1000**, drop row references after
each chunk (`rows = null`), and `await new Promise(r => setImmediate(r))` between
chunks so V8 can garbage-collect. Without this, long scans run out of memory.

### Sub-accounts and memo matching

Include dot sub-accounts with `(from_account = $1 OR from_account LIKE $1 || '.%')`.
When matching exchange deposit memos, drop memos that aren't deposit IDs:

```sql
AND (length(trim(memo)) >= 8 OR trim(memo) ~ '^[0-9]{5,}$')   -- long or numeric deposit IDs
AND trim(memo) NOT LIKE '#%'                                    -- encrypted memos
AND NOT (trim(memo) ~ '^[a-z][a-z0-9.-]{2,19}$')                -- username-like memos
```

### Finding new accounts in a period

```sql
SELECT name, created_at, post_count, reputation
FROM hivemind_app.hive_accounts_info_view
WHERE created_at >= $1::timestamp AND created_at < $2::timestamp
  AND post_count > 0
ORDER BY created_at DESC
```


## 21  Pitfalls checklist

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Filtering `haf_operations` by JSONB body fields | Slow scans / timeouts | Use the typed, account-indexed `operation_*` tables (§1) |
| `dynamic_global_properties … LIMIT 1` without `ORDER BY block_num DESC` | Stale ratio from an arbitrary old block | `ORDER BY block_num DESC LIMIT 1` (§4) |
| Joining `dynamic_global_properties` without the LIMIT subquery | Result × 100 M rows; client OOM | Scalar subquery `(SELECT vests_per_hive … ORDER BY block_num DESC LIMIT 1)` (§4) |
| `vests * vests_per_hive` | HP millions of times too big | `vests / vests_per_hive` — it is VESTS per HIVE (§4) |
| Wrong rshares factor in vote value | Vote values 50× too large (no `0.02`) or `$0.00` (no `1e6`) | `rshares = effectiveVests * 1e6 * 0.02` (§5) |
| `parseAmt` without precision division | HP shows correct (error cancels) but vote values are 1000× inflated | Use precision-aware parseAmt (§7) |
| `rc_api.find_rc_accounts` params wrapped in array | `"Bad Cast: Invalid cast from array_type to Object"` | Pass plain object: `{ accounts: [u] }`, not `[{ accounts: [u] }]` |
| RC/VP `current_mana` used raw | Shows stale low % that ignores time since last activity | Regenerate forward: `stored + elapsed * (max / regen_secs)` (§6) |
| `acc.reputation` from HAFAH `get_accounts` | Always `0` | Use hivemind view or `follow_api.get_account_reputations` (§10) |
| `hive_accounts_info_view.post_count` | Includes comments, overstates root-post count | Query `hafsql.comments WHERE parent_author=''` (§10) |
| `comments.id` used as op id | Wrong results on time-filter | `comments.id` is NOT an op id; filter by `created` timestamp |
| Time-filtering via JS after big fetch | Slow, hits LIMIT before covering full range | Use `id_from_timestamp` in SQL WHERE (§3) |
| `fmtUsd` (toFixed(2)) for vote values | Small values show as `$0.00` | Use `toFixed(4)` for vote table cells |
| Ratio from raw `total_vesting_fund_hive / total_vesting_shares` | HP 1000× too small | Raw columns are ×1e3 / ×1e6 — use `1 / vests_per_hive` (§4, §11) |
| `haf_operations.body.voter` (or any field) is undefined | The view wraps body as `{ type, value }` | Use `row.body.value` / `body->'value'` (§13) |
| History page bounded by `operation_id >= id_from_timestamp(...)` | Timeout for active accounts | Page by `account_op_seq_no` in a MATERIALIZED CTE (§12) |
| Account-index custom_json join without `op_type_id = 18` | Timeout | Add `ao.op_type_id = 18` (§12) |
| Op type name without `hive::protocol::` prefix | `WHERE ot.name = 'vote_operation'` matches nothing | Full name is `hive::protocol::vote_operation`; strip for display only (§1) |
| `pending_payout_value` / `total_payout_value` cast directly to float | Cast fails (string "1.234 HBD") | Use `SPLIT_PART(col::text, ' ', 1)::numeric` in SQL (§14) |
| Combined `WHERE voter = ANY($1)` over many voters | Slow bitmap scan; times out for wide windows | One query per voter in parallel (§3 — per-voter parallel scan) |
| Querying `operation_comment_reward_table` with a 7-day window | Returns payouts for posts from 2 weeks ago, misses this week's | Use 30-day window (§2) |
| `hafsql.to_json(payload)` on `operation_custom_json_view` | `column "payload" does not exist` | The column is `json` (text): `hafsql.to_json(json)` (§1) |
| Querying `hafsql.operation_types` | `relation does not exist` (silent if caught) | Use `hafsql.haf_operation_types` / `hafd.operation_types` with the `hive::protocol::` prefix (§1) |
| Counting rows in `operation_comment_view` | Edited posts counted 2+ times; old body text | `DISTINCT ON (permlink) … ORDER BY permlink, id DESC` (§16) |
| Trusting `hafsql.comments.body` for just-edited posts | Text added in an edit is missing | Re-fetch recent bodies from the Hive API (§16) |
| Looking for DHF payouts in transfers | 0 rows. Proposal pay is a virtual op | `operation_proposal_pay_table`, `payer='hive.fund'`, `proposal_id != 0` (§17) |
| Detecting snaps by `parent_permlink` pattern | Misses replies to other users' snaps | Classify by community id via the hivemind join (§18) |
| `hafsql.comments` lookup with no `created` bound | Full scan / timeout for inactive accounts | Add `created > NOW() - INTERVAL 'N'` (§16) |
| Applying `rawRepToDisplay` to `hafsql.reputations.rep` | Nonsense reputation | `rep` is already display-scale. Convert only the raw `reputation` (§1) |
| Big multi-account scan in a single query/array | Node heap OOM | Chunk ~1000 accounts, null rows, `setImmediate` between chunks (§20) |
