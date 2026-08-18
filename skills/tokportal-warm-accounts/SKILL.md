---
name: tokportal-warm-accounts
description: Warm TokPortal-managed TikTok and Instagram accounts into a niche with Advanced Niche Warming — buy niche targets at bundle creation (count-only or explicit terms), generate search terms for free from a niche description, configure targets on a bundle, order a rewarm on an already-delivered account, and track sessions and verified terms via warming session tools and webhooks. Use when the user asks to warm, niche, re-warm, "make the algorithm learn" or prepare accounts before posting.
metadata:
  author: TokPortal
  version: "1.0.0"
  homepage: https://developers.tokportal.com/advanced-warming
---

# Warm accounts with Advanced Niche Warming

**Advanced Niche Warming** is the recorded, verified warming product for TikTok and Instagram. For each **niche target** (a search term) the account manager starts a screen recording on the account profile (handle visible), searches the term, watches results, likes/saves several videos and leaves a comment. Recordings are spread over **3 calendar days** (manager timezone), each is verified before it counts, and every verified target produces a report (videos watched, likes, saves, comments, keywords, proof recording). Deep warming is discontinued (`DEEP_WARMING_DEPRECATED`); review-based `wants_niche_warming` (7 credits) is being deprecated and is mutually exclusive with advanced warming (`WARMING_CONFLICT`).

## 0. Pricing and consent

- Price: `costs.advanced_warming` credits **per target** (standard 5), bought **3-30 targets in multiples of 3** → minimum 15 credits. Never per day. `contract_bundle_allowance` may return a different effective rate.
- Always call `tokportal_get_credit_costs` and `tokportal_get_credit_balance` first, show `targets × rate` and the resulting balance, and get the user's explicit OK before `create_bundle`, `create_bundles_bulk` or `rewarm_account`.
- Generating terms is free (`tokportal_generate_warming_terms`); reading sessions is free.

## 1. Choose the entry point

| Case | Tool | Notes |
| --- | --- | --- |
| New account(s) not yet created | `tokportal_create_bundle` / `tokportal_create_bundles_bulk` with `wants_advanced_warming: true` and **either** `advanced_warming_terms_count` (recommended: buy now, configure later) **or** `advanced_warming_terms` (one-shot) | Count is per account in bulk. Charged at creation checkout. |
| Bundle created with a count, targets not yet written | `tokportal_configure_bundle_warming_terms` `{id, body:{advanced_warming_terms:[...]}}` | Works at any account status. Must match the purchased count exactly. One-shot. |
| Delivered account you already own | `tokportal_rewarm_account` `{id: saved_account_id, body:{search_terms:[...]}, idempotency_key}` | Terms required up front, session starts immediately. |

Requirements for rewarm: active TokPortal Coverage or permanent grandfathering, a routable active manager backed by a non-cancelled support order (a completed bundle is fine, a cancelled one is not — `REWARM_NO_ACTIVE_ORDER`), no active session (`REWARM_ALREADY_ACTIVE`), TikTok/Instagram only.

## 2. Produce good targets

1. Ask the user for a niche description (product, audience, language, country). Terms are generated in the same language.
2. `tokportal_generate_warming_terms` `{body:{text, platform: "tiktok"|"instagram", count}}` with `count` a multiple of 3 (3-30). Free; call it several times to compare.
3. Curate: each term 2-50 characters, distinct after case-insensitive trim/dedup (duplicates are removed server-side and would break the exact-count rule), search-style phrases people actually type ("high protein recipes"), not hashtags or brand slogans. Show the final list to the user and confirm the count matches what was or will be purchased.

## 3. Buy / configure

- **Count-only purchase (recommended)**: `tokportal_create_bundle` `{body:{bundle_type:"account_only"|"account_and_videos", platform, country, wants_advanced_warming:true, advanced_warming_terms_count:9, ...}, idempotency_key}`. Response `cost_breakdown.advanced_warming` shows the charge. Then `tokportal_configure_bundle_warming_terms` with exactly 9 terms. Not configured within **14 days** → session auto-cancelled and **fully refunded**.
- **Explicit terms at creation**: pass `advanced_warming_terms` instead of the count (3-30, multiple of 3, each 2-50 chars); skips step 2 of the flow.
- **Bulk**: `tokportal_create_bundles_bulk` `{body:{platforms, country, accounts_count, wants_advanced_warming:true, advanced_warming_terms_count}}` then loop `tokportal_configure_bundle_warming_terms` per bundle_id — vary the terms per bundle for a natural network (generate a fresh set or shuffle/curate variants).
- **Rewarm**: after consent, `tokportal_rewarm_account`. Response returns the session; the `warming.session_started` webhook fires immediately.
- Legacy: `tokportal_configure_bundle_account` still accepts `advanced_warming_terms` while the profile is editable, but new integrations should use `configure_bundle_warming_terms`.

Errors: `ADVANCED_WARMING_TERMS` (bad count/length or `details.purchased_terms_count` mismatch → fix the list), `WARMING_TERMS_ALREADY_SET` (one-shot; already configured, do not retry), `WARMING_SESSION_NOT_FOUND` (bundle bought no warming), `ADVANCED_WARMING_PLATFORM`, `MANAGED_ACCOUNT_TASK_BLOCKED` (Coverage inactive/detached/banned — check `tokportal_get_account_managed_subscription`; reactivation costs credits, ask first).

## 4. When warming starts and how it runs

- Account/order already active (existing-account bundle with an assigned manager, or new account already submitted) → starts as soon as targets are configured.
- Otherwise → starts automatically when the manager submits the new account (or accepts the order).
- Tasks are dispatched over 3 days; earlier-day tasks remain open until done. Once tasks exist they never expire and are not refunded for lateness. If Coverage pauses/lapses mid-session, tasks are withheld (not late) and resume on reactivation.

## 5. Track

- `tokportal_list_account_warming_sessions` `{id: saved_account_id}` and `tokportal_get_warming_session` `{id: session_id}` → `status`, `terms_total`, `terms_verified`, `terms_configured`, `started_at`, `completed_at`, per-term reports and proof links.
- The bundle account payload (`tokportal_get_bundle_account`) carries an `advanced_warming` summary with the same fields.
- Webhooks: `warming.session_started`, `warming.term_verified`, `warming.session_completed` (aggregated report). Create the endpoint with `tokportal_create_webhook_endpoint` (no `idempotency_key`; store `signing_secret` once).

## 6. Advice to give the user

- Warm **before** heavy posting: create the bundle with warming, configure targets immediately, then schedule videos ≥ 3 days out so warming runs first.
- 9-15 targets is a common sweet spot; 3 is the minimum viable order.
- Rewarm periodically when pivoting niche or after a long pause, one session at a time per account.
- Report per account: session id, targets configured/verified, start/completion times, credits spent, and any `request_id` on errors.
