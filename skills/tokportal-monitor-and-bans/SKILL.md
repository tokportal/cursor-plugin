---
name: tokportal-monitor-and-bans
description: Monitor a TokPortal account network — read the Analytics v2 dashboard, per-account and per-video metrics, time series, comment pulse and CSV exports, create shareable HTML reports, follow the ban lifecycle (appeal_pending → appeal_accepted / appeal_refused / no_appeal_banned, staff resolution refund / remake / no_remake) with tokportal_list_account_bans as the only source of truth, and set up signed webhooks (bans, cancellations, remakes, Coverage lapses, video/account transitions). Use when the user asks how accounts are performing, whether accounts are banned, wants reports, alerts or webhook integration.
metadata:
  author: TokPortal
  version: "1.0.0"
  homepage: https://developers.tokportal.com/bans-and-appeals
---

# Monitor performance, bans and events

All tools here are read-only except report creation, analytics refresh, webhook management and Coverage changes. Read-only tools never spend credits; still confirm before creating endpoints, refreshing analytics or touching Coverage.

## 1. Analytics

Access tiers differ per plan (`tokportal_get_analytics_contract` tells you what the key can read; full-tier only features are noted).

| Need | Tool | Key params |
| --- | --- | --- |
| Portfolio overview: totals, accounts, top posts, audience, facets | `tokportal_get_analytics_dashboard` | `platform` (repeatable), `country`, `account` (saved account id, repeatable), `from`, `to`, `workspace` |
| One account drilldown | `tokportal_get_analytics_account` `{id}` / `tokportal_get_account_analytics` `{id}` (compat) | saved account id |
| Trend chart | `tokportal_get_analytics_series` | `metric` (`views`,`likes`,`comments`,`shares`,`followers`), `granularity` (`day`,`week`), `mode` (`cumulative`,`gained`,`snapshot`), `account`, `from`, `to`; stored points, no live refresh |
| Posts of an account | `tokportal_list_account_video_analytics` `{id, sort_by, sort_order}` | e.g. `sort_by: views` |
| Single post | `tokportal_get_video_analytics` `{id: video_id}` | |
| Comment pulse (full tier) | `tokportal_get_comment_pulse` / `tokportal_list_analytics_account_comments` | `platform`, `limit`, `trackedPostId` |
| CSV of posts | `tokportal_export_analytics_videos` | `account` filter |
| Raw provider snapshots | `tokportal_list_analytics_account_raw_snapshots` / `tokportal_list_analytics_post_raw_snapshots` | `source`, `limit` |
| Force refresh (rate-limited) | `tokportal_can_refresh_account_analytics` then `tokportal_refresh_account_analytics` / `tokportal_refresh_analytics_account` | ask before refreshing many accounts |

Working method: start with the dashboard filtered by platform/country/date, rank accounts (`engagement`, `views`, `followers`), drill into outliers, then pull the series for the period. Always state the data window (`from`/`to`) and that numbers come from stored snapshots.

**Shareable report**: `tokportal_create_analytics_report` `{body:{title, platforms, countries, accountIds, from, to, brandName, brandAccent, template}}` returns a public `url` and a private `token`. **Never send `idempotency_key`** (secret-returning; `IDEMPOTENCY_KEY_NOT_ALLOWED_FOR_SENSITIVE_RESPONSE`), never paste the token in logs; if the outcome was uncertain, list existing reports before creating another. `tokportal_export_analytics_report_html` renders the HTML.

## 2. Ban lifecycle

`tokportal_list_account_bans` is the **only source of truth** for ban state. Never infer a ban from missing analytics, a `not_found` profile or a cancelled bundle; report statuses exactly as returned. Only validated reports (by the manager or staff) are listed.

```
Manager reports ban
├─ appeal available → appeal_pending  (limbo: unavailable, NOT banned, bundles stay open, no recreation)
│    ├─ accepted     → appeal_accepted (account survived, closed)
│    └─ refused      → appeal_refused  (ban validated)
└─ no appeal        → no_appeal_banned (ban validated)
Validated ban → staff resolution: refund (refund_credits) | remake (same bundle_id, account.remade) | no_remake (reason_code e.g. tos_ban)
```

- Params: `status` (`appeal_pending`|`appeal_accepted`|`appeal_refused`|`no_appeal_banned`), `resolution` (`refund`|`remake`|`no_remake`|`pending`), `account_id`, `since` (ISO watermark = highest `updated_at` seen), `include_screenshots` (signed 7-day evidence URL), `page`, `per_page` (≤ 100).
- Each row: `id`, `account_id`, `username`, `platform`, `bundle_id`, `order_id`, `status`, `reported_at`, `decided_at`, `updated_at`, `resolution{resolution, reason_code, refund_credits, resolved_at}`.
- A validated ban cancels every active bundle/order on the account (`bundle.cancelled`, publish → `409 BUNDLE_INVALID_STATUS`), and marks it `banned` in `tokportal_list_accounts` / `tokportal_get_account` (still visible). `remake` keeps the `bundle_id`; the rebuilt account surfaces via `account.*` events on the same bundle. `tos_ban` = no refund. Coverage credit restoration requires active Coverage at ban time, no reveal, claim within 15 days; restored credits expire after 60 days (`credits.restored`).
- Polling recipe: `tokportal_list_account_bans` with `resolution: "pending"` for open cases and `since` for deltas; cross-check `tokportal_list_account_bundles {id}` before scheduling anything on a flagged account.
- Never call `tokportal_reveal_account_credentials` / `tokportal_retrieve_account_verification_code` "to check" an account: they are the 428 two-step, credit-charging, management-ending tools (see `tokportal-setup`).

## 3. Webhooks (push instead of polling)

1. `tokportal_list_webhook_events` (free) → catalog with `emitted` flags, payload schema, signature scheme.
2. Ask the user for an HTTPS URL they control. Choose events:
   - Bans & lifecycle: `account.ban_appeal.submitted`, `account.ban_appeal.resolved`, `account.banned`, `account.ban_resolution.decided`, `bundle.cancelled`, `account.remade`.
   - Coverage / management: `subscription.renewed`, `subscription.lapsed`, `subscription.cancelled`, `subscription.reactivated`, `subscription.ended`, `account.revealed`, `credits.restored` — stop tasks for that `saved_account_id` after lapse/cancel/end or `account.revealed` with `management_ended: true`; resume only on `subscription.reactivated`.
   - Delivery: `bundle.created`, `bundle.published`, `account.configured`, `account.published`, `account.in_review`, `account.pending_corrections`, `account.finalized`, `video.configured`, `video.published`, `video.in_review`, `video.pending_corrections`, `video.finalized`, `warming.session_started`, `warming.term_verified`, `warming.session_completed`, `bundle.archived`.
3. Confirm, then `tokportal_create_webhook_endpoint` `{body:{url, events:[...], description, enabled}}` — **no `idempotency_key`**. The response contains `signing_secret` **once**: tell the user to store it now; never log it or send it to support. If the secret was lost, `tokportal_delete_webhook_endpoint` (destructive, confirm) and create a replacement.
4. Verify: `tokportal_test_webhook_endpoint {id}` → check `tokportal_list_webhook_deliveries {id}`; failed → `tokportal_retry_webhook_delivery {id, delivery_id}`. Manage with `tokportal_list_webhook_endpoints`, `tokportal_get_webhook_endpoint`, `tokportal_update_webhook_endpoint`.
5. Signature: header `TokPortal-Signature: t=<timestamp>,v1=<hex>`, HMAC-SHA256 over `<t>.<raw body>` with `signing_secret`, constant-time compare, reject stale timestamps. Payload identifiers: `bundle_id` and account listing `account_id` are stable across remakes; `saved_account_id` (what `get_account` uses) changes on every remake and is `null` before `account.in_review`. Code samples: https://developers.tokportal.com/webhooks.

## 4. Coverage checks

`tokportal_get_account_managed_subscription {id}` shows whether Coverage is active/lapsed/cancelled/ended (25 credits / 30 days, first period included). `tokportal_reactivate_account_managed_subscription` charges exactly the unpaid periods — quote it and get consent; `tokportal_cancel_account_managed_subscription` is destructive (stops all tasks, no refund) — require explicit confirmation. Terminal (`ended`) Coverage cannot be reactivated.

## 5. Report

Summarize: window, top/bottom accounts with metrics, open ban cases by status/resolution, Coverage problems, webhook endpoint ids + events + last delivery status, and next actions. Include `request_id` for any error.
