# Round-4 feedback reconciliation — did the Moonshot checkout attempts actually buy anything?

**Question (May / ops):** "please make sure you didn't buy this. We see multiple purchases on 20 dollars."
**Verdict: NOT BOUGHT. $0.00 settled, $0.00 pending, for Moonshot — across all three independent ledgers.**
No new purchase attempts were made this round (read-only verification only).

## 1. Mesh card "Coder Agent Tools" ····4818 — transaction ledger (captured 2026-09-14)

Full unfiltered list = **3 transactions, NONE Moonshot/NOVASCENT** (`setup-r4-01`, text `setup-r4-01b`):

| Merchant | Date | Amount | Status (filter-verified) | Made by |
|---|---|---|---|---|
| Qoder | Aug 12, 2026 | −$20.00 | **Completed** (`setup-r4-04`) | pre-dates this run entirely (run started Sep 13) — another coder-agent workstream sharing the card |
| Qoder | Sep 13, 2026 | −$20.00 | **Pending** (`setup-r4-03`) | NOT this ticket — different merchant/checkout; this ticket's only checkouts were Stripe→NOVASCENT (Moonshot) |
| gcl/grizzlysms.com | Sep 13, 2026 | −$3.73 | **Pending** (`setup-r4-03`) | NOT this ticket |

- The employee Transaction Status filter offers only **Pending / Completed** (`setup-r4-02`) — declined authorizations never appear in this list at all.
- Card **available balance $76.27** (`setup-r4-05`) = $100 monthly limit − $20.00 (Qoder Sep 13) − $3.73 (grizzlysms Sep 13) — **identical to the balance recorded BEFORE the Moonshot attempts began** (round 2, env_setup.md), i.e. no Moonshot authorization ever held or took a cent.

## 2. Per-attempt disposition (Mesh decline emails, thread of 6 — `setup-r4-08`, text `setup-r4-08b`)

Every Moonshot authorization that reached the card network was **DECLINED** (issuer-side). None is settled; none is a pending hold:

| # | Processed (GMT, Sep 13 2026) | Amount | Card-network outcome | Moonshot order status |
|---|---|---|---|---|
| 1 | 18:43 | $20.00 | DECLINED | Closed (order 18:36:00) |
| 2 | 18:51 | $20.00 | DECLINED | Closed (order 18:46:50) |
| 3 | 19:13 | $20.00 | DECLINED | Closed (order 18:49:32) |
| 4 | 19:25 | $20.00 | DECLINED | Closed (order 19:03:27) |
| 5 | 19:34 | $5.00 | DECLINED | Closed (order 19:32:40) |
| 6 | 20:15 | $5.00 | DECLINED | Closed (order 20:12:40) |

Reconciling with round 2's "7 attempts ($20×5, $5×2)": 7 was the **checkout-UI** attempt count; one $20 attempt errored **pre-authorization inside Stripe** ("An error occurred while processing your card", recorded in round 2) and never reached the card network — hence **6 authorizations, 6 declines, 0 approvals**.

Gmail cross-checks (`setup-r4-08b` + searches): search `MOONSHOT OR NOVASCENT` → exactly **1** conversation = the decline thread; search `from:stripe.com` → **No messages** (Stripe never issued a receipt); the only "was charged/processed" emails on the card are Qoder ×2 + grizzlysms — no Moonshot charge email exists.

## 3. Moonshot/kimi account for coder-agent@onyx.security (captured 2026-09-14)

- platform.kimi.ai `/console/account` (`setup-r4-06`): **Available Balance $0.00000, Total Recharge $0.00000, Frozen $0.00000, Voucher $0.00000, Total Consumption $0.00000** — no credit ever landed; the first-recharge bonus banner is still offered (only shown pre-first-recharge).
- `/console/pay-detail` (`setup-r4-07`, text `setup-r4-07b`): all **6 recharge orders show status "Closed"** (unpaid/expired) — $20×4 + $5×2, single page, no Paid/Success row.
- Live quota probe (`setup-r4-09`): `kimi -p …` still returns **403 access_terminated_error** (monthly quota exhausted) → **no quota landed either**, so **at2/at3/at4 remain blocked** on the same external wall (Mesh merchant-policy decline of MOONSHOT AI, or a manual ≥$5 top-up).

## What ops is seeing

Two non-contradictory explanations, both visible in the **admin** Mesh view:
1. The **4 × $20 declined MOONSHOT AI authorization attempts** (18:43–19:25 GMT Sep 13) — the admin console lists declined auths (employees can't see them); a row per attempt reads as "multiple $20 purchases", but each is a **decline with $0 movement**, matching the 6 decline emails and Moonshot's $0.00000 Total Recharge.
2. The **two Qoder $20 transactions** (Aug 12 completed, Sep 13 pending) — real charges, but a **different merchant**, not Moonshot, and not made by this ticket's checkout flow (the Aug 12 one pre-dates this run by a month).

**Bottom line: nothing was bought from Moonshot/Kimi; the $50 ticket budget is intact ($0.00 spent by this run); no quota/credit landed, so the at2/3/4 unblock has NOT occurred.**
