---
name: tokportal-launch-network
description: Launch a network of real TikTok or Instagram accounts in a target country with TokPortal — size the order with credit costs, create one bundle or a bulk batch (accounts + video slots + optional Advanced Niche Warming), configure each account profile, upload media and configure the first videos, check publish readiness, publish, then track delivery through the bundle lifecycle (pending_setup → published → accepted → completed) with webhooks or polling. Use when the user wants new geo-targeted accounts created and operated at scale, e.g. "spin up 20 US TikTok accounts with 5 videos each".
metadata:
  author: TokPortal
  version: "1.1.0"
  homepage: https://developers.tokportal.com/create-bundle
---

# Launch an account network with TokPortal

One **bundle** = one account (new or existing) + optional video slots, in one platform and one country. A network is N bundles. Everything below uses the `tokportal_*` MCP tools; each call runs as the connected TokPortal account and spends its credits.

## 0. Preconditions

- `tokportal_get_current_user` succeeds (see the `tokportal-setup` skill otherwise).
- Only `account_only` / `account_and_videos` create new accounts. `videos_only` re-uses a delivered account and requires `account_id`.
- Advanced Niche Warming is TikTok/Instagram only. `wants_deep_warming` is discontinued (`DEEP_WARMING_DEPRECATED`); `wants_moderation` is ignored — use the Comments API instead.

## 1. Collect the order

Ask for or infer: platform (`tiktok` | `instagram`; YouTube via `tokportal_list_platforms` if listed), country, number of accounts, videos per account (0 allowed), edits per account (≤ videos), warming (recommended: `advanced_warming_terms_count` 3-30 in multiples of 3, or explicit terms), `title` / `external_ref` for correlation, `auto_finalize_videos` (default true).

Validate inputs against reference data:
1. `tokportal_list_platforms`.
2. `tokportal_list_countries` — the country must be in `data` (open for new accounts); `videos_only_countries` only allows existing-account video work. `US`/`GB` aliases are accepted.

## 2. Price it and get consent (mandatory)

1. `tokportal_get_credit_costs` — read `costs.account_creation`, `costs.video_upload`, `costs.video_edit`, `costs.advanced_warming`, and `contract_bundle_allowance` if present (it overrides the standard rate for qualifying bundles).
2. `tokportal_get_credit_balance` — `total_credits`.
3. Compute and show a per-bundle and total estimate, e.g. `accounts × (account + videos × video_upload + edits × video_edit + warming_targets × advanced_warming)`. Standard reference: 32 + 2/video + 3/edit + 5/target (15 minimum). Also mention TokPortal Coverage: 25 credits / 30 days per delivered eligible account, first 30 days included, charged later — not at checkout.
4. If the estimate exceeds the balance, stop and point to https://app.tokportal.com/dashboard/credits.
5. **Ask the user to confirm the exact order before creating anything.** Creation is the paid checkout: credits are debited atomically on success, publication never charges again.

## 3. Create the bundles

- **Single bundle**: `tokportal_create_bundle` with `body: {bundle_type: "account_and_videos" | "account_only", platform, country, videos_quantity, edits_quantity, title, external_ref, wants_advanced_warming: true, advanced_warming_terms_count: 9, auto_finalize_videos}` and an `idempotency_key` (unique per logical order, reuse it only to retry the very same body).
- **Batch (same country)**: `tokportal_create_bundles_bulk` with `body: {platforms: ["tiktok"], country, accounts_count (1-100), upload_accounts_count (≤ accounts_count), videos_per_account (≤ 500), wants_advanced_warming, advanced_warming_terms_count, external_ref}`. It creates `accounts_count × platforms.length` bundles atomically; a batch over the workspace's rolling capacity is rejected in full before any debit. Different countries → one call per country.
- Record every returned `bundle_id`, `credits_charged`, `cost_breakdown`. Report them to the user.
- Errors: `INSUFFICIENT_CREDITS` (402) → stop; `BUNDLE_PRICING_CHANGED` (409) → refetch `tokportal_get_credit_costs`, re-quote, retry with a **new** idempotency key; `DUPLICATE_ACCOUNT_BUNDLE` → an identical `external_ref` already exists (`details.existing_bundle_id`), do not create again without asking; `COUNTRY_NOT_ENABLED` → pick another country from step 1.

## 4. Configure each account profile

For every bundle in `pending_setup`, call `tokportal_configure_bundle_account` with `body: {username (1-24 chars, [a-zA-Z0-9_.]), visible_name (1-30), biography (≤ 80), profile_picture_url, link_in_bio (Instagram only)}`.

- `profile_picture_url` must be the `storage_path` returned by `tokportal_upload_image_direct` (local file) or `tokportal_upload_image_from_url` (public image URL) — **not** `public_url`.
- Use **unique wording** per account: identical words in usernames/nicknames across many accounts raise mass-ban risk. Generate distinct handles and bios per bundle.
- If warming was bought as a count, write the targets with `tokportal_configure_bundle_warming_terms` (see `tokportal-warm-accounts`; `tokportal_generate_warming_terms` is free). Unconfigured purchases auto-refund after 14 days.

## 5. Upload media and configure the first videos

- Video files: `tokportal_upload_video_direct` (`file_path`, `bundle_id`) → use `public_url` as `video_url`. Any public external URL (Drive, Dropbox, direct link) also works as `video_url`: TokPortal downloads and re-hosts it.
- Presigned alternative for browsers: `tokportal_upload_video` (never with `idempotency_key`) → PUT the file to `upload_url` with the given Content-Type → use `public_url`.
- `tokportal_batch_configure_bundle_videos` with `body: {videos: [{position, video_type: "video", video_url, description, target_publish_date: "YYYY-MM-DD", instagram_content_type ("reel"|"post", Instagram only), tiktok_sound_url (optional)}], auto_publish?}`. `auto_publish` is **top-level only** — one publish attempt for the whole call; inside a `videos[]` item it is rejected.
- `target_publish_date` is the **first day of a 2-day window** (the manager may post that day or the next); the end day is derived and cannot be set when configuring — `target_publish_end_date` is rejected here (`UNKNOWN_FIELD`) and only `tokportal_patch_bundle_video` accepts an explicit window. Lead time: ≥ 3 days ahead while the account is still being created, ≥ 1 day for an existing or delivered account (`INVALID_DATE` → `details.earliest_allowed`); **max 3 videos per day per bundle** (`VIDEOS_PER_DAY_EXCEEDED`). Sound volume fields cost 1 credit the first time; `story_repost_url` / `instant_repost_as_story` cost 1 credit each. Detail: `tokportal-post-videos-at-scale`.
- `account_and_videos` bundles need at least 1 configured video to publish; slots can be filled later and pushed with `tokportal_publish_all_bundle_videos` once the bundle is active.

## 6. Check readiness, then publish

1. `tokportal_get_bundle_publish_readiness` (read-only) for each bundle → fix every `blockers[]` entry (`ACCOUNT_MISSING_FIELDS`, `NO_VIDEOS_CONFIGURED`, `BUNDLE_ALREADY_PUBLISHED`, ...).
2. Confirm with the user, then `tokportal_publish_bundle` per bundle. Publishing is free (already paid) but irreversible once a manager accepts. `capacity_cooldown` (429) → the workspace's rolling publication capacity is exhausted; wait and retry later, do not spam.
3. Optional shortcut: `auto_publish: true` on the video configuration call publishes immediately after a successful configuration.
4. Publish results may carry `adjusted_videos: [{video_id, position, previous_date, new_date}]` + `adjusted_videos_note` when a date no longer cleared the lead time. Report the adjusted calendar, not the requested one.
5. To edit the account after publishing, `tokportal_unpublish_bundle` first (destructive; ask; `already_accepted` means too late).

## 7. Track delivery

Bundle lifecycle: `pending_setup → published (published_priority) → accepted → completed`; video phase inside: `accepted → in_review → finalized → completed`; `cancelled` is terminal (typically a validated ban).

- Poll `tokportal_list_bundles` (`status`, `account_status`, `platform`, `country` filters) or `tokportal_get_bundle` per id: `is_published`, `account_configured`, `videos_configured_count`, `account_status` (`pending → configured → in_review → finalized`).
- Account in `in_review`: approve with `tokportal_finalize_bundle_account` or `tokportal_request_bundle_account_corrections` (`comment`, `fields`). No action for 72 h → auto-finalized.
- Videos: `tokportal_list_bundle_videos`; `in_review` → `tokportal_finalize_bundle_video` / `tokportal_request_bundle_video_corrections`; 72 h auto-finalize applies too.
- Delivered accounts appear in `tokportal_list_accounts` / `tokportal_get_account` (`saved_account_id` from `account.in_review` onwards). Ordinary account tools never return passwords or codes.
- Push instead of polling: `tokportal_create_webhook_endpoint` with at least `bundle.published`, `account.in_review`, `account.finalized`, `video.published`, `video.finalized`, `account.banned`, `bundle.cancelled`, `account.remade` (see `tokportal-monitor-and-bans`).

## 8. Report

Give the user a table: bundle_id, platform/country, status, account_status, videos configured/published, credits charged, warming state, plus next actions and any error `request_id`.
