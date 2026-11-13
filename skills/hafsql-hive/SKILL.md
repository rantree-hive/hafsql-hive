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
| `hafsql.accounts` | Per-account HP, delegation totals, reputation, last activity, keys |
| `hafsql.balances` | Current HIVE / HBD / VESTS / hp_equivalent per account |
| `hafsql.dynamic_global_properties` | Chain globals incl. `vests_per_hive` direct ratio |
| `hafsql.reputations` | `rep` (human-readable), raw `reputation` |
| `hafsql.delegations` | Active delegations with HP equivalent |
| `hafsql.comments` | Hivemind post/comment view with payout columns |
| `hivemind_app.hive_accounts_info_view` | followers, following, post_count, profile JSON |


## 2  Time-range filtering — always use `id_from_timestamp`

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
// Correct — matches Hive's internal rshares scale
const voteHive100 = (effectiveVests * 1_000_000 / 100) / recentClaims
                    * rewardBalance * 0.5;
const voteValue100Usd = voteHive100 * hivePrice;
```

- `effectiveVests` = `vesting_shares - delegated_vesting_shares + received_vesting_shares` (raw VESTS from `condenser_api`)
- `recentClaims` = `parseFloat(fund.recent_claims)` from `condenser_api.get_reward_fund`
- `rewardBalance` = `parseAmt(fund.reward_balance)` (see §6)
- `0.5` = 50 % author share post-HF21
- The factor `1_000_000 / 100 = 10_000` is the rshares scaling constant; `recent_claims` accumulates in those same scaled units. **Removing it gives vote values ~10 000× too small (rounds to $0.00).**

`voteValue100Usd` is the dollar value of one **100 % weight, 100 % VP** vote.
Scale to other VP levels: `voteValue100Usd * (vpPercent / 100)`.


## 5  RC percentage

```js
const rcAcc = rcAccounts?.rc_accounts?.[0];
if (rcAcc) {
  const rcMax     = parseFloat(rcAcc.max_rc ?? 0);
  const rcCurrent = parseFloat(rcAcc.rc_manabar?.current_mana ?? 0);
  const rcPercent = rcMax > 0 ? Math.round((rcCurrent / rcMax) * 1000) / 10 : null;
}
```

API call: `rc_api.find_rc_accounts` with `{ accounts: [username] }`.
`max_rc` and `current_mana` are both large-integer strings.
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


## 8  Pitfalls checklist

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Querying `haf_operations` for virtual ops | Returns 0 rows for reward ops | Use dedicated `operation_*` tables (§1) |
| Missing rshares factor in vote value | Vote values round to `$0.00` | Keep `* 1_000_000 / 100` in formula (§4) |
| `parseAmt` without precision division | HP shows correct (error cancels) but vote values are 1000× inflated | Use precision-aware parseAmt (§6) |
| `comments.id` used as op id | Wrong results on time-filter | `comments.id` is NOT an op id; filter by `created` timestamp |
| Time-filtering via JS after big fetch | Slow, hits LIMIT before covering full range | Use `id_from_timestamp` in SQL WHERE (§2) |
| `fmtUsd` (toFixed(2)) for vote values | Small values show as `$0.00` | Use `toFixed(4)` for vote table cells |
