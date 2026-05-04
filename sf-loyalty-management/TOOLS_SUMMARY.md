# Salesforce Loyalty Management MCP Tools: Summary & Tribal Knowledge

These tools target the **Salesforce Loyalty Management** API surface
through the MCP server. They cover the public Connect APIs (Loyalty
Business APIs), the underlying sObjects (LoyaltyProgram, Member, Tier,
Voucher, Promotion, Game, Benefit, etc.), and the Standard / Custom
Invocable Actions used to run loyalty program processes.

Tools are split into two top-level directories by **access mode**:

- `read/` — read-only tools: fetch programs, members, points,
  vouchers, transaction history, promotions, games, badges, partners,
  benefits.
- `write/` — DML / process invocation: enroll members, credit/debit
  points, issue/redeem/cancel vouchers, create/update promotions,
  change tiers, run loyalty program processes, manage program metadata.

Inside each, tools are grouped by domain:

- `programs/` — LoyaltyProgram, LoyaltyProgramCurrency, LoyaltyTier,
  LoyaltyTierGroup
- `members/` — LoyaltyProgramMember, member profile, points balance,
  current tier, member promotions
- `transactions/` — TransactionJournal (Accrual / Redemption / Manual /
  Adjustment), transaction history, ledger summary
- `vouchers/` — VoucherDefinition, Voucher, member vouchers,
  issue/redeem/cancel
- `promotions/` — Promotion, PromotionTier, eligible promotions,
  promotion details, member promotion enrollment
- `games/` — GameDefinition, GameReward, GameParticipant, member games
- `badges/` — LoyaltyProgramBadge, LoyaltyProgramMemberBadge
- `partners/` — LoyaltyProgramPartner, LoyaltyPartnerProduct,
  LoyaltyPgmPartnerCurrency, LoyaltyPgmPartnerPromotion
- `benefits/` — Benefit, BenefitType, LoyaltyTierBenefit, MemberBenefit,
  BenefitAction
- `processes/` (write only) — generic `run_program_process` runner for
  any named Loyalty Program Process

---

## Salesforce APIs in use

| API                          | Base path                                                | Used for                                                              |
|------------------------------|----------------------------------------------------------|-----------------------------------------------------------------------|
| Loyalty Connect API          | `/services/data/v66.0/connect/loyalty/...`               | Member enrollment, profile, benefits, transactions, vouchers, promotions |
| Loyalty Program Process API  | `/services/data/v66.0/connect/loyalty-program-process/...` | Run a named Loyalty Program Process (cancel voucher, credit points, enroll for promotions, etc.) |
| Standard Invocable Actions   | `/services/data/v66.0/actions/standard/...`              | Apex-backed invocable actions (e.g. ChangeTier, GetMemberTier, GetMemberPoints) |
| REST API (Data)              | `/services/data/v66.0/sobjects/...` and `/query/`        | SOQL on LoyaltyProgram, LoyaltyProgramMember, Voucher, Promotion, etc. |
| Metadata API                 | `/services/data/v66.0/metadata/deployRequest`            | LoyaltyProgramSetup, BenefitAction, decision tables                   |
| Tooling API                  | `/services/data/v66.0/tooling/sobjects/...`              | BenefitAction tooling sObject and developer-time entities             |

> **API version**: Tools default to **v66.0 (Spring '26)**. Reference
> documentation lives next to this file (`loyalty_api.pdf`).

---

## Authentication & headers

The MCP server holds a long-lived OAuth credential and exchanges it for
session tokens. From a tool caller's perspective no headers need to be
passed; upstream calls always send:

- `Authorization: Bearer <access_token>`
- `Content-Type: application/json`
- `Accept: application/json`

---

## Identity model

| Concept                | API name / format                                      | Examples                                              |
|------------------------|--------------------------------------------------------|-------------------------------------------------------|
| Loyalty Program Name   | Program API name (`LoyaltyProgram.Name`)               | `MTG_Rewards`, `Acme_Frequent_Flyer`                  |
| Loyalty Program Member | `LoyaltyProgramMember.Id` (18-char, key prefix `0lM`)  | `0lMQg00000abcDE2`                                    |
| Membership Number      | `LoyaltyProgramMember.MembershipNumber` (string)        | `MEM-100023`                                          |
| Loyalty Tier           | `LoyaltyTier.Id` + `Name`                              | "Gold", "Platinum"                                    |
| Tier Group             | `LoyaltyTierGroup.Id` + `Name`                         | "Status Tiers", "Lifetime Tiers"                      |
| Program Currency       | `LoyaltyProgramCurrency.Name` (e.g. "Points", "Miles") | Used as a parameter on most points operations         |
| Voucher Definition     | `VoucherDefinition.Id` + `Name`                        | `VD-Pizza-10off`                                      |
| Voucher                | `Voucher.Id` + `VoucherCode`                           | `VC-89231`                                            |
| Promotion              | `Promotion.Id` + `Name`                                | `Q3-Earn-Double-Points`                               |
| Transaction Journal    | `TransactionJournal.Id` + `JournalTypeId`              | One row per accrual / redemption / adjustment         |
| Game Definition        | `GameDefinition.Id`                                    | "Spin the Wheel", "Scratch Card"                       |

> **Member ID vs Membership Number**: The Connect API consistently
> accepts the 18-char `LoyaltyProgramMember.Id`. Some partner endpoints
> also accept `MembershipNumber` — the per-tool spec calls out which.

---

## Common parameters used by Connect APIs

Most loyalty endpoints take a `programName` path parameter. Many also
take a `programCurrencyName` body parameter to disambiguate which
points balance is involved. Tools treat both as required where the
upstream API does, and surface the same error codes.

---

## Loyalty Program Process invocations

The Loyalty Program Process API gives admins a generic way to invoke a
named process by name. Common process names (each has a dedicated
write tool; the generic `run_program_process` accepts any of them):

- `Cancel a Voucher`
- `Credit Points to Members`
- `Debit Points from Members`
- `Issue a Voucher`
- `Enroll for Promotions`
- `Opt Out from a Promotion`
- `Get Member Promotions`
- `Update Member Details`
- `Update Member Tier`
- `Unenroll a Member`

> Most clients use the **specific** named tool (e.g.
> `credit_points`, `issue_voucher`) because the server validates the
> known parameter shape. Use `run_program_process` for custom processes
> or rare ones not covered by a dedicated tool.

---

## Async vs sync behavior

| Operation                        | Behavior                                                    |
|----------------------------------|-------------------------------------------------------------|
| Get points / tier / profile      | Synchronous, returns the full payload.                      |
| Credit / debit points            | Synchronous in most cases; the platform may queue if heavy. |
| Issue / redeem / cancel voucher  | Synchronous.                                                |
| Transaction Journal (Execute)    | Synchronous; runs accrual/redemption rules immediately.     |
| Transaction Journal (Simulate)   | Synchronous; runs rules without persisting; returns deltas. |
| Loyalty Program Process          | May be sync or async depending on process definition.       |
| Promotion creation               | Synchronous Connect API call; deploys backing metadata.     |

---

## Idempotency & limits

- **Member enrollment** is **not idempotent**: a second enrollment of
  the same Account/Contact creates a duplicate `LoyaltyProgramMember`.
  Use `JournalSubType` / `MemberStatus` checks before enrolling, or
  pass an external Id and the upstream upsert pattern via
  `dml_records` (see `salesforce-core`).
- **Issue voucher** is **not idempotent** unless you supply an
  external transaction Id; otherwise re-running will issue a second
  voucher.
- **Credit / debit points** is **not idempotent**; pass a stable
  external transaction reference (tools accept `externalTransactionId`)
  to deduplicate at the program-process level.
- **Transaction Journal Execute** writes ledger rows; not idempotent.
- **Transaction Journal Simulate** is read-only and safe to retry.
- **Limits**: Loyalty endpoints count toward DailyApiRequests + the
  program-specific transaction-journal capacity; check
  `salesforce-core/read/org/get_org_limits` for headroom.

---

## Error model

Every tool surfaces the Salesforce error array verbatim:

```json
{ "errorCode": "INVALID_FIELD",
  "message": "No such column 'Foo__c' on entity 'LoyaltyProgramMember'.",
  "fields": ["Foo__c"] }
```

Loyalty-specific error codes you'll see:

| Code                                  | Meaning / fix                                            |
|---------------------------------------|----------------------------------------------------------|
| `LOYALTY_PROGRAM_NOT_FOUND`           | `programName` is wrong or the program is inactive.       |
| `LOYALTY_MEMBER_NOT_FOUND`            | `memberId` is wrong, deleted, or in a different program. |
| `LOYALTY_INSUFFICIENT_POINTS`         | Debit/redemption exceeds the member's available balance. |
| `LOYALTY_TIER_INVALID`                | Target tier doesn't exist in the member's tier group.    |
| `LOYALTY_PROMOTION_NOT_ELIGIBLE`      | Member doesn't meet promotion entry criteria.            |
| `LOYALTY_VOUCHER_EXPIRED`             | Voucher is past `EffectiveDate`/`ExpirationDate`.        |
| `LOYALTY_VOUCHER_ALREADY_REDEEMED`    | Voucher already in `Redeemed` status.                    |
| `LOYALTY_VOUCHER_CANCELLED`           | Voucher already in `Canceled` status.                    |
| `LOYALTY_PROCESS_NOT_FOUND`           | Process name doesn't match an active LoyaltyProgramProcess. |
| `INSUFFICIENT_ACCESS_OR_READONLY`     | Running user doesn't have the Loyalty perm set.          |

---

## Required permissions

Most Loyalty tools require **one of**:

- `Loyalty Management` permission set (read paths).
- `Loyalty Management Admin` permission set (write paths).
- `Marketing User` for promotion creation.

The MCP server exchanges its OAuth credential as the configured
loyalty admin user; tools surface `INSUFFICIENT_ACCESS_OR_READONLY`
when the configured user is missing perms.

---

## Tool index

### read/ — 35 tools

- **`read/programs/` (5)** — `list_loyalty_programs`,
  `get_loyalty_program`, `list_loyalty_currencies`,
  `list_loyalty_tiers`, `list_loyalty_tier_groups`
- **`read/members/` (7)** — `list_members`, `get_member`,
  `get_member_profile`, `get_member_benefits`,
  `get_member_points_balance`, `get_member_tier`,
  `list_member_promotions`
- **`read/transactions/` (3)** — `list_transaction_journals`,
  `get_transaction_history`, `get_transaction_ledger_summary`
- **`read/vouchers/` (3)** — `list_voucher_definitions`,
  `list_member_vouchers`, `get_voucher`
- **`read/promotions/` (3)** — `list_promotions`,
  `get_promotion_details`, `list_eligible_promotions`
- **`read/games/` (4)** — `list_games`, `list_game_rewards`,
  `list_game_participants`, `list_member_games`
- **`read/badges/` (2)** — `list_loyalty_badges`,
  `list_member_badges`
- **`read/partners/` (4)** — `list_loyalty_partners`,
  `list_partner_products`, `list_partner_currencies`,
  `list_partner_promotions`
- **`read/benefits/` (4)** — `list_benefits`, `list_benefit_types`,
  `list_tier_benefits`, `list_member_benefits`

### write/ — 38 tools

- **`write/programs/` (4)** — `manage_loyalty_program`,
  `manage_currency`, `manage_tier`, `manage_tier_group`
- **`write/members/` (7)** — `enroll_individual_member`,
  `enroll_corporate_member`, `update_member_details`,
  `update_member_tier`, `change_member_tier`, `merge_members`,
  `unenroll_member`
- **`write/transactions/` (4)** — `execute_transaction_journal`,
  `simulate_transaction_journal`, `credit_points`, `debit_points`
- **`write/vouchers/` (4)** — `manage_voucher_definition`,
  `issue_voucher`, `cancel_voucher`, `redeem_voucher`
- **`write/promotions/` (4)** — `manage_promotion`,
  `create_promotion_via_connect`, `enroll_for_promotion`,
  `opt_out_from_promotion`
- **`write/games/` (4)** — `manage_game_definition`,
  `manage_game_reward`, `assign_game_to_member`, `claim_game_reward`
- **`write/badges/` (3)** — `manage_loyalty_badge`,
  `assign_member_badge`, `revoke_member_badge`
- **`write/partners/` (4)** — `manage_loyalty_partner`,
  `manage_partner_product`, `manage_partner_currency`,
  `manage_partner_promotion`
- **`write/benefits/` (3)** — `manage_benefit`, `manage_tier_benefit`,
  `manage_benefit_action`
- **`write/processes/` (1)** — `run_program_process` (generic
  Loyalty Program Process runner)

---

## Per-tool spec format

Each tool has its own `.txt` file. Same shape as `salesforce-core`:

```
Tool: <tool_name>

Description
<one paragraph: what it does, when to use it, related tools>

Salesforce API
<Loyalty Connect | Loyalty Program Process | Standard Invocable | REST | Metadata | Tooling>, vXX.0

Endpoint
<METHOD https://{instance}/services/data/v66.0/...>

Headers
<auth + content-type>

Body
<JSON shape if applicable, or "None (GET)">

Input (parameters)
<MCP tool parameters: name, type, required/optional, description>

Result
<response shape: keys + types + meaning>

Required permissions
<perm set / profile permissions the running user must have>

Idempotency & limits
<retry safety, governor / limit notes, sync vs async>

Errors
<key error codes the tool surfaces>

Example request
<JSON body or full URL>

Example result
<JSON shape>
```

For full per-API reference see the bundled `loyalty_api.pdf`.
