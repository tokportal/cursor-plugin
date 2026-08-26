---
name: tokportal-post-videos-at-scale
description: Post videos, carousels and stories across many TokPortal-managed TikTok/Instagram accounts — upload media (direct, presigned or external URL), add video slots to active bundles or create videos_only bundles on delivered accounts, configure and schedule slots one by one, in batch or via CSV import while respecting the 3-videos-per-day-per-bundle cap and minimum lead times, publish them, then follow video statuses (configured → published → in_review → finalized), finalize or request corrections. Use when the user wants to schedule or distribute content at volume through TokPortal accounts.
metadata:
  author: TokPortal
  version: "1.1.0"
  homepage: https://developers.tokportal.com/configure-videos
---

# Post videos at scale with TokPortal

Human account managers publish through the native apps, so there is no per-account OAuth and no 25-posts/day API cap — but each bundle (one account) accepts **at most 3 videos per target day**. Scale comes from many bundles, not from stacking posts on one account.

## 0. Preconditions and pricing

- Every video slot costs `costs.video_upload` credits (standard 2), edits `costs.video_edit` (3), sound volume 1 credit the first time it is set on a video, `story_repost_url` / `instant_repost_as_story` 1 credit each, ad code 7. Slots inside an existing bundle are already paid; **new** slots (`tokportal_add_video_slots`, new bundles) are debited immediately.
- Before any paid step: `tokportal_get_credit_costs` + `tokportal_get_credit_balance`, show the total, get the user's OK. Retry paid calls only with the same `idempotency_key`.
- The target account must have active TokPortal Coverage or be grandfathered (`MANAGED_ACCOUNT_TASK_BLOCKED` otherwise → check `tokportal_get_account_managed_subscription`; reactivate only after the user agrees).

## 1. Find where the videos go

| Situation | Do this |
| --- | --- |
| Slots already exist in a bundle | `tokportal_get_bundle` → `videos_quantity`, `videos_configured_count`; `tokportal_list_bundle_videos` → free positions (`status: "pending"`). |
| Active bundle, need more slots | `tokportal_add_video_slots` `{id, body:{quantity}}` (paid; bundle not `completed`). Preferred: keeps the same manager. |
| Delivered account with no live bundle | `tokportal_list_accounts` → `saved_account_id`; `tokportal_create_bundle` `{bundle_type:"videos_only", account_id, videos_quantity}` (platform/country derived; `account_id` is only valid with `videos_only`). A `videos_only` bundle has no account-profile step: configure the videos, then publish the bundle. |
| Many accounts | Loop over bundles; keep a table `bundle_id → positions → dates`. |

## 2. Prepare media

- Local file: `tokportal_upload_video_direct` `{file_path, bundle_id}` → `public_url` (use as `video_url`). Images: `tokportal_upload_image_direct` → `storage_path` (use for `carousel_images` / `profile_picture_url`), or `tokportal_upload_image_from_url` for a public image URL.
- Presigned (browser flows): `tokportal_upload_video` / `tokportal_upload_image` `{filename, bundle_id, content_type}` — **never** with `idempotency_key` — then PUT the bytes to `upload_url` with that Content-Type; use `public_url` / `storage_path`. Do not log or persist `upload_url`.
- External URLs (Google Drive, Dropbox, direct links) can be passed straight into `video_url` / `story_image_url` / CSV: TokPortal downloads and re-hosts them. Broken link later → `tokportal_fix_bundle_video_download`.

## 3. Build the schedule (respect the cap)

- `target_publish_date` is `YYYY-MM-DD` and is the **first day of a 2-day publishing window**, not a fixed date: the manager may post on that day or the next. The API stores `target_publish_start_date` = the day you sent and `target_publish_end_date` = that day + 1. The end day is derived and **cannot be chosen** when configuring (single slot, batch or CSV) — sending `target_publish_end_date` there is rejected with `UNKNOWN_FIELD` (400), no longer silently dropped. Only `tokportal_patch_bundle_video` accepts an explicit window.
- Minimum lead time (UTC): **≥ 3 days ahead while the account is still being created, ≥ 1 day** once the bundle runs on an existing account or its account is delivered/finalized. Violations → `INVALID_DATE` with `details.min_days_ahead` and `details.earliest_allowed` (`YYYY-MM-DD`) — schedule from `details.earliest_allowed` rather than guessing.
- **Max 3 videos per day per bundle**, counting every non-cancelled slot already dated that day. A 4th → `VIDEOS_PER_DAY_EXCEEDED` (`details.date`, `details.current_count`). Spread over more days or more bundles; 3+ posts/day on one account is rarely healthy anyway.
- Algorithm: for each bundle, read existing dates from `tokportal_list_bundle_videos`, then assign new positions to the earliest allowed date with < 3 videos, in position order. Show the resulting calendar before writing.
- Reconfiguring a video on its own current day does not count against itself.

## 4. Configure slots

Required per slot: `video_type` (`video` | `carousel` | `story`) and `target_publish_date`; `description` required for video/carousel (stories have none); `video_url` for video; `carousel_images[]` (storage paths) for carousel; `tiktok_sound_url` required for TikTok carousels; `instagram_content_type` (`reel` | `post`) required on Instagram; story = exactly one of `video_url` / `story_image_url`. Free flags: `ai_content_disclaimer`, `disclose_as_ads`. Optional: `editing_instructions`, `name`, `external_ref`, `instagram_location`, `instagram_collaborators`, `instagram_audio_name`, `instagram_add_to_story`.

- One slot: `tokportal_configure_bundle_video` `{id, position, body}`.
- Many slots in one bundle: `tokportal_batch_configure_bundle_videos` `{id, body:{videos:[{position, video_type, ...}], auto_publish?}}` — partial failures come back per position; fix and resend only those. `auto_publish` here is **top-level only**: one publish attempt per call, applying to every video in it. Putting it inside a `videos[]` item is rejected ("auto_publish is a top-level field on this endpoint and applies to every video in the call — move it out of videos[]"). The per-item fields are the same as the single-slot configure **minus** `auto_publish` and `target_publish_end_date`.
- Spreadsheet: `tokportal_import_bundle_videos_csv` `{id, file_path, auto_publish?}`. Columns: `position, video_type, description, target_publish_date, video_url, carousel_images (; separated), tiktok_sound_url, instagram_content_type, instagram_location, instagram_collaborators, instagram_add_to_story, editing_instructions, external_ref, ai_content_disclaimer, disclose_as_ads, volume_original_sound, volume_added_sound, instant_repost_as_story`. Response lists `imported`, `skipped`, `errors[{row,message}]`; valid rows are still imported. Templates: https://developers.tokportal.com/csv-import.
- Small metadata/schedule tweaks later: `tokportal_patch_bundle_video` — the **only** call that can set an explicit window, via the `target_publish_start_date` / `target_publish_end_date` pair (`target_publish_date` alone still means "this day + the next"). Since 2026-08-26 it enforces the same minimum lead time as the initial configuration, so rescheduling to today or to a past day is refused with `INVALID_DATE`. Wrong slot: `tokportal_reset_bundle_video` / `tokportal_unschedule_bundle_video` (destructive — confirm with the user first).

## 5. Publish

- Bundle still `pending_setup` → `tokportal_get_bundle_publish_readiness` then `tokportal_publish_bundle` (or `auto_publish: true` on the configure/CSV call). `capacity_cooldown` → wait, retry later.
- Publishing may **move dates that no longer clear the lead time**. `tokportal_publish_bundle` and any `auto_publish` result may carry `adjusted_videos: [{video_id, position, previous_date, new_date}]` + `adjusted_videos_note`; `tokportal_publish_bundle_video` may carry `date_adjusted` / `original_date` / `new_date` / `hint`; `tokportal_publish_all_bundle_videos` may carry `dates_adjusted` + `adjusted_videos` + `hint`. Always read these back and report the real calendar to the user instead of the one you requested.
- Bundle already `published` / `published_priority` / `accepted` → new `configured` slots are **not** visible to the manager until you call `tokportal_publish_all_bundle_videos` `{id}` (or `tokportal_publish_bundle_video` `{id, position}`). Always finish with this step after adding slots to an active bundle.
- Typical loop for an active bundle: `add_video_slots` → `batch_configure_bundle_videos` → `publish_all_bundle_videos`.

## 6. Follow and close the loop

Video statuses: `pending → configured → published → accepted → in_review → finalized`.

- `tokportal_list_bundle_videos` / `tokportal_get_bundle_video` for status and the posted URL once published.
- `in_review` (manager posted, awaiting your approval): `tokportal_finalize_bundle_video` to approve, or `tokportal_request_bundle_video_corrections` `{id, position, comment, fields:{...}}`. No action within 72 h → auto-finalized. `auto_finalize_videos: true` (default) skips the review wait.
- Ad code (Spark / Partner code, 7 credits) on a finalized video: `tokportal_create_video_ad_code_request` → `tokportal_get_video_ad_code_request`.
- Webhooks for push updates: `video.configured`, `video.published`, `video.in_review`, `video.pending_corrections`, `video.finalized`, plus `bundle.cancelled` / `account.banned` to stop scheduling on a dead account (see `tokportal-monitor-and-bans`).

## 7. Report

Return: per bundle the positions configured, dates, published count, credits spent, CSV row errors, and any `request_id` from failures. Never paste API keys or upload URLs into the report.
