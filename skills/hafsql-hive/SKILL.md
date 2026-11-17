---
name: hafsql-hive
description: >
  Build correct, performant queries against HafSQL — the PostgreSQL layer over
  HAF-indexed Hive blockchain data. Covers which tables to query for each
  operation type (especially the virtual-op trap), the rshares/vote-value
  formula, VESTS-to-HP conversion, milli-VESTS scaling, time-range filtering
  with id_from_timestamp, production pool setup with circuit breaker,
  operation timeline cursor pagination, haf_operations body wrapping, and
  Hive API response parsing patterns.
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
| `hafsql.haf_operations` | Raw non-virtual ops (`id`, `op_type_id`, `timestamp`, `body` JSONB) |
| `hafsql.haf_operation_types` | Op type registry (`id`, `name`, `is_virtual`) |
| `hafsql.operations` | **Combined** view — real + virtual ops (use when you need to find virtual-op ids) |

**Critical rule:** `haf_operations` does **not** contain virtual operations (author rewards, curation rewards, beneficiary rewards, fill-vesting-withdraw, etc.). Joining through it to find those types returns zero rows silently.

**Op type name format:** `haf_operation_types.name` is prefixed with `hive::protocol::` (e.g. `hive::protocol::vote_operation`). Strip for display: `.replace(/^hive::protocol::/, '')`. Filter with the full prefixed name. Skip `hive::protocol::effective_comment_vote_operation` — it is noisy and fires on every vote.

### Layer 2 — dedicated operation tables (fast, correct)

Use these instead of JSONB body parsing via `haf_operations`:

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
| `hafsql.operation_custom_json_view` | join via `hafd.account_operations` | `custom_id`, `payload` bytes → `hafsql.to_json(payload)` |

### Rich account / global views

| View | Purpose |
|------|---------|
| `hafsql.accounts` | Per-account HP, delegation totals, reputation, last activity, keys (`creator`, `recovery`). **VESTS columns are milli-VESTS — see §11.** |
| `hafsql.balances` | Current HIVE / HBD / VESTS / hp_equivalent per account. `account_name` is the join key. **`vests` column is milli-VESTS — see §11.** |
| `hafsql.dynamic_global_properties` | Chain globals incl. `vests_per_hive` direct ratio, `total_vesting_fund_hive`, `total_vesting_shares` |
| `hafsql.reputations` | `rep` (display score), raw `reputation`, `account_name` |
| `hafsql.delegations` | Active delegations with HP equivalent |
| `hafsql.comments` | All authored content (root posts **and** replies). `parent_author = ''` → root posts only. Payout columns are "N.NNN HBD" strings — see §14. |
| `hivemind_app.hive_accounts_info_view` | `followers`, `following`, `reputation`, `posting_json_metadata`, `post_count` (all authored content, not just root posts) |
| `hivemind_app.hive_votes_view` | Per-vote rshares: `voter`, `author`, `permlink`, `rshares` — use for vote-value computation from actual rshares |

### HafSQL utility functions

| Function | Returns | Use for |
|----------|---------|---------|
| `hafsql.id_from_timestamp(ts, round_up)` | operation id | time-range lower bound in WHERE (§3) |
| `hafsql.get_timestamp(id)` | timestamptz | recover a human timestamp from an op id |
| `hafsql.to_json(payload)` | JSONB | parse custom_json payload bytes to JSONB for field extraction |

```sql
-- get_timestamp: convert op id back to date for grouping
SELECT hafsql.get_timestamp(id)::date AS day, SUM(amount) AS volume
FROM hafsql.operation_transfer_table
WHERE ...
GROUP BY 1 ORDER BY 1;

-- to_json: extract fields from custom_json payload
SELECT (hafsql.to_json(payload))->>'subject' AS subject,
       (hafsql.to_json(payload))->>'action'  AS action
FROM hafsql.operation_custom_json_view
WHERE (hafsql.to_json(payload))->>'action' IN ('create', 'edit');
```

### `hafsql.operations` vs `hafsql.haf_operations`

`hafsql.operations` (no prefix) is the **combined** view that includes both real
and virtual operations. `hafsql.haf_operations` excludes virtual ops.
Use `hafsql.operations` when you need to find virtual-op ids
(e.g., linking fill_vesting_withdraw_table rows to their op ids).


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

```sql
-- From HafSQL (preferred — no API call)
SELECT vests_per_hive FROM hafsql.dynamic_global_properties LIMIT 1;
-- hp = vests * vests_per_hive  (direct ratio, no division)
```

Or fetch from `condenser_api.get_dynamic_global_properties` and compute:
```js
const vestsToHp = totalFundHive / totalVestShares;
// where both values come from parseAmt() — see §7
```

`hafsql.balances.hp_equivalent` already stores the HP value directly for
the current state; `hafsql.accounts.incoming_hp` / `outgoing_hp` do too.

> **Important:** `hafsql.balances.vests`, `hafsql.accounts.incoming_vests`, and
> `hafsql.accounts.outgoing_vests` are stored in **milli-VESTS** (÷1000 of whole VESTS).
> Multiply by 1000 before the VESTS→HP conversion. See §11 for details.


## 5  Vote value formula (rshares scale)

```js
// Correct — matches Hive's internal rshares scale (empirically verified 2026-11)
const voteHive100 = (effectiveVests * 1_000_000 / 25) / recentClaims
                    * rewardBalance * 0.5;
const voteValue100Usd = voteHive100 * hivePrice;
```

- `effectiveVests` = `vesting_shares - delegated_vesting_shares + received_vesting_shares` (raw VESTS from `condenser_api`)
- `recentClaims` = `parseFloat(fund.recent_claims)` from `condenser_api.get_reward_fund`
- `rewardBalance` = `parseAmt(fund.reward_balance)` (see §7)
- `0.5` = 50 % author share post-HF21
- The factor `1_000_000 / 25 = 40_000` is the correct rshares-per-VEST scale constant as of HAFAH. `recent_claims` accumulates in those same units.

**Derivation:** At 100 % VP/weight, back-solving from peakd's displayed vote value:
`factor = target_HBD × recentClaims / (effectiveVests × rewardBalance × HBD_per_HIVE × 0.5) ≈ 40 000`

**Common wrong factors and their symptoms:**

| Factor | Code expression | Symptom |
|--------|----------------|---------|
| `1_000_000 / 25 = 40_000` | ✅ correct | matches peakd reference |
| `1_000_000 / 100 = 10_000` | 4× too small | shows ¼ of real value |
| `1_000_000 / 200 = 5_000` | 8× too small | shows ⅛ of real value |
| `1` (factor omitted) | 40 000× too small | rounds to `$0.0000` |

`voteValue100Usd` is the dollar value of one **100 % weight, 100 % VP** vote.
Scale to other VP levels: `voteValue100Usd * (vpPercent / 100)`.

> The ◆ symbol in peakd's vote value UI is **HBD** (≈ $1 each), not HIVE.
> 0.234 ◆ means $0.234, not $0.234 × hivePrice.

### Vote value from actual rshares (hivemind_app.hive_votes_view)

For a specific cast vote (not the full-weight estimate), look up its rshares
from `hivemind_app.hive_votes_view` and convert directly:

```js
// rs = row.rshares from hive_votes_view (BigInt from DB — coerce with Number())
const vote_usd = Math.round(
  (Math.abs(rs) / recentClaims) * rewardBalance * 0.5 * hivePrice * 10000
) / 10000 * Math.sign(rs);
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


## 11  milli-VESTS scaling — `hafsql.accounts` and `hafsql.balances`

`hafsql.balances.vests`, `hafsql.accounts.incoming_vests`, and
`hafsql.accounts.outgoing_vests` store VESTS in **milli-VESTS** (1/1000 of a whole
VEST). Always multiply by 1000 before the vestsToHp conversion.

```js
const VESTS_SCALE = 1000;

// From hafsql.balances
const ownVests  = parseAmt(dbRow.vests)          * VESTS_SCALE;
const delVests  = parseAmt(dbRow.outgoing_vests)  * VESTS_SCALE;
const recVests  = parseAmt(dbRow.incoming_vests)  * VESTS_SCALE;
const effectiveVests = ownVests - delVests + recVests;

const vestsToHp = totalVestShares > 0 ? totalFundHive / totalVestShares : 0;
const hp = Math.round(effectiveVests * vestsToHp);
```

The same scale factor applies when converting `vesting_payout` from the reward
tables (operation_author_reward_table etc.):

```js
// vesting_payout from operation_*_table is also milli-VESTS
const authorHp = Math.round(
  parseFloat(ar.vests_all || 0) * VESTS_SCALE * vestsToHp * 1000
) / 1000;
```

> `hafsql.accounts.incoming_hp`, `outgoing_hp`, and `balances.hp_equivalent` are
> already in HP — no scaling needed. The raw VESTS columns are the ones that need it.


## 12  Operation timeline — `haf_account_operations` cursor pagination

For paginated operation history (e.g. "show account ops for a given day"):

```sql
-- Paginated daily operation history
-- cursor is the account_op_seq_no from the last item of the previous page
SELECT
  o.id,
  ao.account_op_seq_no AS seq_no,
  o.timestamp,
  ot.name        AS op_type,
  ot.is_virtual,
  o.body
FROM hafsql.haf_account_operations ao
JOIN hafsql.haf_operations o       ON ao.operation_id = o.id
JOIN hafsql.haf_operation_types ot ON o.op_type_id    = ot.id
WHERE ao.account_name = $1
  AND ao.operation_id >= hafsql.id_from_timestamp($2::timestamptz, false)
  AND ao.operation_id <= COALESCE(hafsql.id_from_timestamp($3::timestamptz, true),
                                  9223372036854775807)
  AND ot.name NOT IN ('hive::protocol::effective_comment_vote_operation')
  AND ao.account_op_seq_no < $4   -- cursor: omit on first page
ORDER BY ao.account_op_seq_no DESC
LIMIT 51  -- fetch 51, hasMore = (rows.length === 51), serve rows.slice(0, 50)
```

```js
const hasMore    = rows.length === 51;
const ops        = rows.slice(0, 50);
const nextCursor = hasMore ? ops[ops.length - 1].seq_no : null;
```

The `account_op_seq_no` cursor is stable across queries and works correctly
even when new ops arrive during pagination.

### Reading custom_json via account operation index (hafd layer 0)

When you need custom_json ops for a specific account+custom_id from the full
history (not just recent), join through `hafd.accounts` → `hafd.account_operations`:

```sql
WITH account_notes AS (
  SELECT
    v.id,
    v.json   AS payload,
    v.timestamp
  FROM hafd.accounts a
  JOIN hafd.account_operations ao ON ao.account_id = a.id
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


## 13  `haf_operations.body` — HAFAH body wrapping

On HAFAH nodes the `body` JSONB column of `haf_operations` may be wrapped in an
extra `{ value: ... }` object. Always unwrap before accessing fields:

```js
const body = row.body?.value ?? row.body;
// ✅ body.voter, body.author, body.permlink — safe
// ❌ row.body.voter — will be undefined on HAFAH
```

This applies to any query that reads from `hafsql.haf_operations` or
`hafsql.operations` and accesses JSONB body fields in JS.


## 14  `hafsql.comments` payout columns

Payout columns in `hafsql.comments` are stored as `"N.NNN HBD"` strings, not
plain numerics. Use `SPLIT_PART` to extract the number in SQL, or parse in JS:

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


## 16  Pitfalls checklist

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Querying `haf_operations` for virtual ops | Returns 0 rows for reward ops | Use dedicated `operation_*` tables (§1) |
| Wrong rshares factor in vote value | Vote values too small or `$0.00` | Use `* 1_000_000 / 25` (§5) |
| `parseAmt` without precision division | HP shows correct (error cancels) but vote values are 1000× inflated | Use precision-aware parseAmt (§7) |
| `rc_api.find_rc_accounts` params wrapped in array | `"Bad Cast: Invalid cast from array_type to Object"` | Pass plain object: `{ accounts: [u] }`, not `[{ accounts: [u] }]` |
| RC/VP `current_mana` used raw | Shows stale low % that ignores time since last activity | Regenerate forward: `stored + elapsed * (max / regen_secs)` (§6) |
| `acc.reputation` from HAFAH `get_accounts` | Always `0` | Use hivemind view or `follow_api.get_account_reputations` (§10) |
| `hive_accounts_info_view.post_count` | Includes comments, overstates root-post count | Query `hafsql.comments WHERE parent_author=''` (§10) |
| `comments.id` used as op id | Wrong results on time-filter | `comments.id` is NOT an op id; filter by `created` timestamp |
| Time-filtering via JS after big fetch | Slow, hits LIMIT before covering full range | Use `id_from_timestamp` in SQL WHERE (§3) |
| `fmtUsd` (toFixed(2)) for vote values | Small values show as `$0.00` | Use `toFixed(4)` for vote table cells |
| `balances.vests` / `accounts.incoming_vests` / `outgoing_vests` used raw for HP | HP is 1000× too small | These are milli-VESTS — multiply by 1000 before `× vestsToHp` (§11) |
| `haf_operations.body.voter` (or any field) is undefined | HAFAH wraps body in `{ value: {...} }` | Use `row.body?.value ?? row.body` before accessing fields (§13) |
| Op type name without `hive::protocol::` prefix | `WHERE ot.name = 'vote_operation'` matches nothing | Full name is `hive::protocol::vote_operation`; strip for display only (§1) |
| `pending_payout_value` / `total_payout_value` cast directly to float | Cast fails (string "1.234 HBD") | Use `SPLIT_PART(col::text, ' ', 1)::numeric` in SQL (§14) |
| Combined `WHERE voter = ANY($1)` over many voters | Slow bitmap scan; times out for wide windows | One query per voter in parallel (§3 — per-voter parallel scan) |
| Querying `operation_comment_reward_table` with a 7-day window | Returns payouts for posts from 2 weeks ago, misses this week's | Use 30-day window (§2) |
