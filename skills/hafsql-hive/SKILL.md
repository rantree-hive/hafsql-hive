---
name: hafsql-hive
description: >
  Build correct, performant queries against HafSQL — the PostgreSQL layer over
  HAF-indexed Hive blockchain data. Covers which tables to query for each
  operation type (especially the virtual-op trap), the rshares/vote-value
  formula, VESTS-to-HP conversion, time-range filtering with
  id_from_timestamp, and Hive API response parsing patterns.
  Trigger whenever writing backend code that queries HafSQL or calls
  condenser_api / rc_api for a Hive application.
---

# HafSQL × Hive — Query & API Skill

## 1  Schema map — which table does what

HafSQL has two layers. Always prefer the **dedicated operation tables** (layer 2) over the generic views (layer 1) for any operation-specific work.

### Layer 1 — generic views (slow, general)

| View | Use for |
|------|---------|
| `hafsql.haf_account_operations` | account–op index (`account_id`, `account_name`, `operation_id`) |
| `hafsql.haf_operations` | raw non-virtual ops (`id`, `op_type_id`, `timestamp`, `body` JSONB) |
| `hafsql.haf_operation_types` | op type registry (`id`, `name`, `is_virtual`) |

**Critical rule:** `haf_operations` does **not** contain virtual operations (author rewards, curation rewards, beneficiary rewards, fill-vesting-withdraw, etc.). Joining through it to find those types returns zero rows silently.

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

### Rich account / global views

| View | Purpose |
|------|---------|
| `hafsql.accounts` | Per-account HP, delegation totals, reputation, last activity, keys (`creator`, `recovery`) |
| `hafsql.balances` | Current HIVE / HBD / VESTS / hp_equivalent per account |
| `hafsql.dynamic_global_properties` | Chain globals incl. `vests_per_hive` direct ratio |
| `hafsql.reputations` | `rep` (human-readable), raw `reputation` |
| `hafsql.delegations` | Active delegations with HP equivalent |
| `hafsql.comments` | All authored content (root posts **and** replies). Filter `parent_author = ''` for root posts only |
| `hivemind_app.hive_accounts_info_view` | followers, following, reputation, posting/json metadata. `post_count` column counts all authored content (posts + comments) — do **not** use it for root-post-only counts |
| `operation_custom_json_view` | Custom JSON operations; payload is bytes — use `hafsql.to_json(payload)` to get JSONB |

### HafSQL utility functions

| Function | Returns | Use for |
|----------|---------|---------|
| `hafsql.id_from_timestamp(ts, round_up)` | operation id | time-range lower bound in WHERE (§2) |
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


## 3  VESTS → HP conversion

```sql
-- From HafSQL (preferred — no API call)
SELECT vests_per_hive FROM hafsql.dynamic_global_properties LIMIT 1;
-- hp = vests * vests_per_hive  (direct ratio, no division)
```

Or fetch from `condenser_api.get_dynamic_global_properties` and compute:
```js
const vestsToHp = totalFundHive / totalVestShares;
// where both values come from parseAmt() — see §6
```

`hafsql.balances.hp_equivalent` already stores the HP value directly for
the current state; `hafsql.accounts.incoming_hp` / `outgoing_hp` do too.


## 4  Vote value formula (rshares scale)

```js
// Correct — matches Hive's internal rshares scale (empirically verified 2026-11)
const voteHive100 = (effectiveVests * 1_000_000 / 25) / recentClaims
                    * rewardBalance * 0.5;
const voteValue100Usd = voteHive100 * hivePrice;
```

- `effectiveVests` = `vesting_shares - delegated_vesting_shares + received_vesting_shares` (raw VESTS from `condenser_api`)
- `recentClaims` = `parseFloat(fund.recent_claims)` from `condenser_api.get_reward_fund`
- `rewardBalance` = `parseAmt(fund.reward_balance)` (see §6)
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


## 5  RC percentage

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


## 6  Hive API response parsing (`parseAmt`)

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


## 7  Common Hive API calls (backend / Node.js)

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
const rcData = await hiveAPI('rc_api.find_rc_accounts', [{ accounts: [username] }]);
```


## 8  Profile metadata — read from hivemind, not Hive API

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


## 9  HAFAH broken fields (confirmed 2026-11)

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


## 9  Pitfalls checklist

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Querying `haf_operations` for virtual ops | Returns 0 rows for reward ops | Use dedicated `operation_*` tables (§1) |
| Wrong rshares factor in vote value | Vote values too small or `$0.00` | Use `* 1_000_000 / 25` (§4) |
| `parseAmt` without precision division | HP shows correct (error cancels) but vote values are 1000× inflated | Use precision-aware parseAmt (§6) |
| `rc_api.find_rc_accounts` params wrapped in array | `"Bad Cast: Invalid cast from array_type to Object"` | Pass plain object: `{ accounts: [u] }`, not `[{ accounts: [u] }]` |
| RC/VP `current_mana` used raw | Shows stale low % that ignores time since last activity | Regenerate forward: `stored + elapsed * (max / regen_secs)` (§5) |
| `acc.reputation` from HAFAH `get_accounts` | Always `0` | Use hivemind view or `follow_api.get_account_reputations` (§8) |
| `hive_accounts_info_view.post_count` | Includes comments, overstates root-post count | Query `hafsql.comments WHERE parent_author=''` (§8) |
| `comments.id` used as op id | Wrong results on time-filter | `comments.id` is NOT an op id; filter by `created` timestamp |
| Time-filtering via JS after big fetch | Slow, hits LIMIT before covering full range | Use `id_from_timestamp` in SQL WHERE (§2) |
| `fmtUsd` (toFixed(2)) for vote values | Small values show as `$0.00` | Use `toFixed(4)` for vote table cells |
