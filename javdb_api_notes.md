# JavDB APK API Notes

Target APK: `jdb_official_v1.9.35.apk`

Validated on `2026-04-19`.

## Overview

Base host:

```text
https://jdforrepam.com
```

Common characteristics observed so far:

- Most endpoints use `GET`
- Requests require a `jdsignature` header
- Server responses generally follow this envelope:

```json
{
  "success": 1,
  "action": null,
  "message": null,
  "data": {}
}
```

## Verification Scope

This document mixes two evidence levels:

- Live verification: the route was called from this workspace against a live host
- Static inference: the route or field shape was extracted from Flutter AOT strings, presenter names, or request-building logic

Unless a section explicitly says otherwise, "observed" means it came from a live request rather than from APK strings alone.

## Request Conventions

### Base request shape

The current Python helper in `src/javdb_api_client.py` reproduces the minimal request style that has worked so far:

```http
jdsignature: {timestamp}.lpw6vgqzsp.{md5(timestamp + suffix)}
user-agent: Mozilla/5.0
accept: application/json
```

Notes:

- No cookie jar has been required for the currently verified public endpoints
- `authorization` appears optional for public data routes and required for some user actions
- Multipart form bodies are needed for at least the login flow inferred from Flutter code
- The login token can be reused directly as the `authorization` header value
- The helper also supports `JAVDB_AUTHORIZATION` as a default auth token source

### Interpreting failures

The server often signals failure in the JSON payload instead of through a non-`200` HTTP code.

Useful failure categories seen so far:

- `action: "ParameterInvalid"`
  The route exists, but one or more required parameters are missing or empty.
- `action: "InvalidSignature"`
  The route exists and the current host validates `jdsignature`.
- `action: "JWTVerificationError"`
  The route exists and likely needs authenticated user context.
- `HTTP 404`
  Usually suggests the method is wrong, the route is unavailable on that host, or the real endpoint shape differs from the attempted request.

### Practical auth classification

For this workspace, routes are easiest to classify in three buckets:

- Public signed route
  Requires `jdsignature` but not `authorization` for basic access.
- Authenticated route
  Accepts the path and method, but returns `JWTVerificationError` without login context.
- Unconfirmed route
  Only seen in static code or still returning `404` or ambiguous failures.

## Repro Workflow

Recommended live test order for new endpoints:

1. Confirm the host with `GET /api/v1/startup` using the known-good Android params.
2. Call the target route with only `jdsignature`.
3. If the result is `ParameterInvalid`, add the missing params one by one.
4. If the result is `JWTVerificationError`, mark the route as auth-gated.
5. If the result is still ambiguous, compare with APK presenter code before documenting stronger conclusions.

This keeps the notes aligned with reproducible behavior instead of guesswork.

When parameters are missing or wrong, the server usually still returns `HTTP 200` and reports the error in JSON:

```json
{
  "success": 0,
  "action": "ParameterInvalid",
  "message": "參數不能爲空: ...",
  "data": null
}
```

## Authentication

This APK adds a `jdsignature` HTTP header on requests.

For this specific APK and original signing context, the working formula is:

```text
jdsignature = "{timestamp}.{middle}.{md5(timestamp + suffix)}"
```

Recovered constants:

```text
middle = lpw6vgqzsp
suffix = 71cf27bb3c0bcdf207b64abecddc970098c7421ee7203b9cdae54478478a199e7d5a6e1a57691123c1a931c057842fb73ba3b3c83bcd69c17ccf174081e3d8aa
```

Notes:

- `timestamp` is a Unix timestamp in seconds
- This was recovered from Flutter AOT plus `libsecurity.so`
- These values are tied to this APK sample's original signing context
- Re-signing the APK breaks the startup flow because the key chain changes

Example:

```text
1776543056.lpw6vgqzsp.a064c72d612073065fa5cb8662ac5fb9
```

Minimal request headers used successfully:

```http
jdsignature: {generated_value}
user-agent: Mozilla/5.0
accept: application/json
```

## Auth Matrix

This table summarizes the current practical auth state of routes already touched in this workspace.

`Public` means `jdsignature` was sufficient in live requests.

`Auth required` means the route shape was accepted, but the server responded with `JWTVerificationError` without a logged-in token.

`Unknown` means the route exists in static analysis or still needs a more complete live request shape.

| Route | Method | Current auth state | Evidence |
| --- | --- | --- | --- |
| `/api/v1/startup` | `GET` | Public | Live verified |
| `/api/v1/about` | `GET` | Public | Live verified |
| `/api/v1/helps` | `GET` | Public | Live verified |
| `/api/v3/plans` | `GET` | Public | Live verified |
| `/api/v1/movies/latest` | `GET` | Public | Live verified |
| `/api/v4/movies/%s` | `GET` | Public | Live verified |
| `/api/v1/movies/%s/magnets` | `GET` | Public | Live verified |
| `/api/v1/movies/%s/reviews` | `GET` | Public | Live verified |
| `/api/v2/search` | `GET` | Public | Live verified |
| `/api/v1/search_magnet` | `GET` | Public | Live verified |
| `/api/v1/reviews/hotly` | `GET` | Public with required `period` | Live verified |
| `/api/v1/actors` | `GET` | Public with required `type` | Live verified |
| `/api/v1/series` | `GET` | Public with required `type` | Live verified |
| `/api/v1/movies/%s/reviews/%s/like` | `POST` | Public for current test round | Live verified |
| `/api/v1/movies/%s/reviews/%s/report` | `POST` | Auth required | Live verified |
| `/api/v1/movies/%s/reviews/%s` | `DELETE` | Auth required | Live verified |
| `/api/v1/movies/%s/play` | `GET` | Auth required | Live verified |
| `/api/v1/movies/%s/resume_play` | `GET` | Auth required | Live verified |
| `/api/v1/sessions` | `POST` | Public login entrypoint | Live verified |
| `/api/v1/users` | `GET` | Auth required | Live verified |
| `/api/v1/users/recent_viewed` | `GET` | Auth required | Live verified |
| `/api/v1/wallets` | `GET` | Auth required | Live verified |
| `/api/v1/wallets/usdt_chain_types` | `GET` | Auth required | Live verified |
| `/api/v1/movies/top` | `GET` | Auth required | Live verified |
| `/api/v2/search_image` | Unknown | Unknown | Static inference plus failed naive `GET` |

Practical notes:

- `jdsignature` is still required even on routes that are otherwise public
- A successful login returns `data.token`, which can be replayed directly as the `authorization` header value
- No confirmed route has required a `Bearer ` prefix so far
- The `review like` route should still be treated cautiously until retested with a fresh anonymous client, because public write access would be unusual

## Reusable Response Objects

These field groups repeat across multiple endpoints and are useful as a shorthand when documenting new routes.

### `movie` object

Observed across:

- `/api/v1/movies/latest`
- `/api/v2/search`
- `/api/v4/movies/%s`

Common list/detail fields observed so far:

- `id`
- `number`
- `title`
- `origin_title`
- `thumb_url`
- `cover_url`
- `duration`
- `magnets_count`
- `can_play`
- `play_subtitle`
- `has_preview_video`
- `has_cnsub`
- `has_preview_images`
- `release_date`
- `new_magnets`
- `preview_images`

Detail-only or richer fields seen on `/api/v4/movies/%s`:

- `type`
- `number_letter`
- `summary`
- `score`
- `reviews_count`
- `comments_count`
- `want_watch_count`
- `watched_count`
- `play_sources`
- `top_rankings`
- `maker_id`
- `maker_name`
- `director_id`
- `director_name`
- `publisher_id`
- `publisher_name`
- `series_id`
- `series_name`
- `tags`
- `actors`
- `review`
- `preview_video_url`
- `relative_movies`
- `actor_movies`

### `review` object

Observed on:

- `/api/v1/movies/%s/reviews`

Fields observed:

- `id`
- `user_id`
- `username`
- `watched_count`
- `status`
- `status_title`
- `score`
- `content`
- `likes_count`
- `liked`
- `created_at`

Likely semantics:

- `status` and `status_title` appear to represent the reviewer's watch state or evaluation bucket
- `liked` is per-viewer state and may behave differently once authenticated requests are used

### `magnet` object

Observed on:

- `/api/v1/movies/%s/magnets`
- `/api/v1/search_magnet`

Common fields observed:

- `hash`
- `size`
- `files_count`
- `created_at`

Movie magnet list fields:

- `name`
- `cnsub`
- `hd`
- `pikpak_url`

Magnet search result fields:

- `id`
- `title`

Current inference:

- `/api/v1/movies/%s/magnets` returns the richer per-movie torrent entry shape
- `/api/v1/search_magnet` returns a flatter cross-movie search result shape

## Request Body Conventions

Most confirmed routes are signed `GET` requests with query parameters only, but the APK also uses form payloads for mutation and auth flows.

### Query-first routes

Observed examples:

- `/api/v1/startup`
- `/api/v1/helps`
- `/api/v2/search`
- `/api/v1/movies/%s/play`
- `/api/v1/movies/%s/resume_play`

Notes:

- Missing required query items usually produce `success:0` with `action: "ParameterInvalid"`
- Even state-changing behavior in the app sometimes still starts from `GET` routes rather than `POST`

### `application/x-www-form-urlencoded` or multipart form routes

Observed or inferred examples:

- `/api/v1/sessions`
- likely `/api/v2/search_image`
- likely some password reset and account mutation routes from the pending inventory

Current workspace guidance:

- Prefer multipart for routes directly inferred from Flutter `FormData.fromMap(...)`
- Use the client `request --form ... --multipart` path when reproducing those requests
- For unknown body routes, start with multipart before assuming JSON

### Authorization token reuse

After a successful login:

- copy `data.token`
- send it back as the raw `authorization` header value
- keep sending `jdsignature` as well

No successful route has yet shown a token-only flow that skips `jdsignature`.

## Verified Endpoints

### `GET /api/v1/startup`

Purpose:

- Startup configuration
- Splash ad config
- App settings
- Backup domains blob
- Search keyword presets

Confirmed query parameters:

```text
last_ad_id=
platform=android
app_channel=official
app_version=official
app_version_number=1.9.35
```

Confirmed behavior:

- Missing `jdsignature` -> `ParameterInvalid: jdsignature`
- Invalid `jdsignature` -> `InvalidSignature`
- Valid signature but missing query params -> parameter error
- Valid signature plus correct params -> `success:1`

Successful example:

```http
GET https://jdforrepam.com/api/v1/startup?last_ad_id=&platform=android&app_channel=official&app_version=official&app_version_number=1.9.35
jdsignature: {timestamp}.lpw6vgqzsp.{md5(timestamp + suffix)}
```

Observed top-level `data` fields:

- `splash_ad`
- `user`
- `backup_domains_data`
- `recent_keywords`
- `recent_magnet_keywords`
- `settings`
- `feedback`
- `logging_enabled`
- `recognize_actor_enabled`
- `recognize_movie_enabled`
- `web_image_prefix`
- `ypay_payment_enabled`

Useful nested values observed:

- `settings.UPDATE_DESCRIPTION`
- `settings.NOTICE`
- `settings.AGENT_GROUP`
- `settings.INSTALLATION_URL`
- `settings.VERSION`
- `settings.TESTFLIGHT_URL`
- `splash_ad.ad.media_url`
- `splash_ad.ad.link_url`
- `web_image_prefix`

Response shape excerpt:

```json
{
  "success": 1,
  "data": {
    "settings": {
      "INSTALLATION_URL": "https://ddbao.fssdag.com/jdb_official_v1.9.35.apk",
      "VERSION": "1.9.35|1"
    },
    "web_image_prefix": "https://c0.jdbstatic.com/",
    "ypay_payment_enabled": true
  }
}
```

### `GET /api/v1/about`

Purpose:

- Public links and distribution links

Required query params:

- None observed

Observed response:

- `success:1`
- `data` is an array of grouped link lists

Observed fields:

- `name`
- `meta`
- `url`

Observed values include:

- `https://javdb572.com`
- `https://javdb.com`
- `https://app.javdb572.com`
- `https://jav.app`
- `https://t.me/javdbnews`
- `https://t.me/javdbbot`

Example:

```http
GET https://jdforrepam.com/api/v1/about
jdsignature: {generated_value}
```

### `GET /api/v1/helps`

Purpose:

- FAQ / help center content

Required query params:

```text
category
```

Observed behavior:

- Without `category` -> `ParameterInvalid: category`
- With `category=app` -> `success:1`
- With `category=common` -> `success:1`

Both `app` and `common` currently returned the same FAQ dataset in this test round.

Observed response fields:

- `helps`
- `current_page`

Each help entry contains:

- `question`
- `answer`

Successful example:

```http
GET https://jdforrepam.com/api/v1/helps?category=app
jdsignature: {generated_value}
```

### `GET /api/v3/plans`

Purpose:

- Subscription plans
- Payment platforms and methods

Required query params:

- None observed

Observed response fields:

- `plan_message`
- `unavailable_plan_message`
- `intros`
- `plans`
- `platforms`

Observed nested plan fields:

- `id`
- `name`
- `price`
- `origin_price`
- `currency_unit`
- `currency_symbol`
- `days`

Observed nested payment fields:

- `platforms[].id`
- `platforms[].channels[].id`
- `platforms[].channels[].methods[].id`
- `platforms[].channels[].methods[].price_type`
- `platforms[].channels[].methods[].limited_prices`

Example:

```http
GET https://jdforrepam.com/api/v3/plans
jdsignature: {generated_value}
```

### `GET /api/v1/movies/latest`

Purpose:

- Latest movie feed

Required query params:

- None observed for the default first page

Observed response fields:

- `movies`
- `current_page`

Observed movie fields:

- `id`
- `number`
- `title`
- `origin_title`
- `thumb_url`
- `cover_url`
- `duration`
- `magnets_count`
- `can_play`
- `play_subtitle`
- `has_preview_video`
- `has_cnsub`
- `has_preview_images`
- `release_date`
- `new_magnets`
- `preview_images`

Example:

```http
GET https://jdforrepam.com/api/v1/movies/latest
jdsignature: {generated_value}
```

### `GET /api/v4/movies/%s`

Purpose:

- Movie detail
- Related movies
- Actor-side related feed
- Preview media and metadata

Path parameter:

- `%s` is the movie id

Validated example id:

```text
bKOeYA
```

Successful example:

```http
GET https://jdforrepam.com/api/v4/movies/bKOeYA
jdsignature: {generated_value}
```

Observed response fields:

- `share_info`
- `show_vip_banner`
- `movie`

Observed nested `movie` fields:

- `id`
- `type`
- `number`
- `number_letter`
- `title`
- `origin_title`
- `summary`
- `thumb_url`
- `cover_url`
- `has_cnsub`
- `has_preview_images`
- `duration`
- `score`
- `reviews_count`
- `comments_count`
- `want_watch_count`
- `watched_count`
- `magnets_count`
- `has_preview_video`
- `can_play`
- `play_subtitle`
- `play_sources`
- `release_date`
- `top_rankings`
- `maker_id`
- `maker_name`
- `director_id`
- `director_name`
- `publisher_id`
- `publisher_name`
- `series_id`
- `series_name`
- `tags`
- `actors`
- `review`
- `preview_video_url`
- `preview_images`
- `relative_movies`
- `actor_movies`

Notes:

- The detail response includes a preview m3u8 URL with its own `sign` and `t` query params
- This endpoint worked without extra query params beyond the movie id in the path

### `GET /api/v1/movies/%s/magnets`

Purpose:

- Movie-specific magnet list

Path parameter:

- `%s` is the movie id

Validated example ids:

```text
deQX39
bKOeYA
```

Successful examples:

```http
GET https://jdforrepam.com/api/v1/movies/deQX39/magnets
jdsignature: {generated_value}
```

```http
GET https://jdforrepam.com/api/v1/movies/bKOeYA/magnets
jdsignature: {generated_value}
```

Observed response fields:

- `magnets`

Observed magnet fields:

- `name`
- `hash`
- `size`
- `cnsub`
- `hd`
- `files_count`
- `created_at`
- `pikpak_url`

Notes:

- This endpoint returned data without extra query params
- The returned structure is richer than `/api/v1/search_magnet`, because it includes `cnsub`, `hd`, and `pikpak_url`

### `GET /api/v1/movies/%s/reviews`

Purpose:

- Movie comment/review list

Path parameter:

- `%s` is the movie id

Validated example ids:

```text
deQX39
bKOeYA
```

Successful examples:

```http
GET https://jdforrepam.com/api/v1/movies/deQX39/reviews
jdsignature: {generated_value}
```

```http
GET https://jdforrepam.com/api/v1/movies/bKOeYA/reviews
jdsignature: {generated_value}
```

Observed response fields:

- `reviews`

Observed review fields:

- `id`
- `user_id`
- `username`
- `watched_count`
- `status`
- `status_title`
- `score`
- `content`
- `likes_count`
- `liked`
- `created_at`

Observed behavior:

- Some movie ids return an empty review list
- Some movie ids return populated review arrays without extra query params

Notes:

- `bKOeYA` returned multiple reviews
- `deQX39` returned `reviews: []`
- No pagination parameter has been confirmed yet for this endpoint

### `POST /api/v1/movies/%s/reviews/%s/like`

Purpose:

- Like a review

Path parameters:

- First `%s` is the movie id
- Second `%s` is the review id

Validated example:

```http
POST https://jdforrepam.com/api/v1/movies/bKOeYA/reviews/222247168/like
jdsignature: {generated_value}
```

Observed behavior:

- `GET` on this route returned `404`
- `POST` on this route returned `success:1`

Observed response:

```json
{
  "success": 1,
  "action": null,
  "message": null,
  "data": null
}
```

Current note:

- In this test round, this action succeeded without an `authorization` header

### `POST /api/v1/movies/%s/reviews/%s/report`

Purpose:

- Report a review

Path parameters:

- First `%s` is the movie id
- Second `%s` is the review id

Validated example:

```http
POST https://jdforrepam.com/api/v1/movies/bKOeYA/reviews/222247168/report
jdsignature: {generated_value}
```

Observed behavior:

- `GET` on this route returned `404`
- `POST` on this route returned a login-required error

Observed response:

```json
{
  "success": 0,
  "action": "JWTVerificationError",
  "message": "請登錄帳號",
  "data": null
}
```

Current note:

- This route appears to require authenticated user context

### `DELETE /api/v1/movies/%s/reviews/%s`

Current verification status:

- `GET` on this route returned `404`
- AOT references this route from a delete-comment flow in `movie_presenter.dart`
- `DELETE` on this route returned a login-required error

Validated example:

```http
DELETE https://jdforrepam.com/api/v1/movies/bKOeYA/reviews/222247168
jdsignature: {generated_value}
```

Observed response:

```json
{
  "success": 0,
  "action": "JWTVerificationError",
  "message": "請登錄帳號",
  "data": null
}
```

Current inference:

- This route is not a public "review detail GET" endpoint
- It is more likely the delete-comment endpoint and requires login

### `GET /api/v2/search`

Purpose:

- Keyword-based movie search

Required query params:

```text
q
```

Observed optional query params:

```text
page
```

Observed behavior:

- Without `q` -> `ParameterInvalid: q`
- With `q=KV-316` -> returns matched movie list
- With `q=KV-316&page=2` -> returns `success:1` and `movies: []`, `current_page: 2`

Validated examples:

```http
GET https://jdforrepam.com/api/v2/search?q=KV-316
jdsignature: {generated_value}
```

```http
GET https://jdforrepam.com/api/v2/search?q=美ノ嶋めぐり
jdsignature: {generated_value}
```

Observed response fields:

- `movies`
- `current_page`

Observed movie item shape is very close to `/api/v1/movies/latest`:

- `id`
- `number`
- `title`
- `origin_title`
- `thumb_url`
- `cover_url`
- `duration`
- `magnets_count`
- `can_play`
- `play_subtitle`
- `has_preview_video`
- `has_cnsub`
- `has_preview_images`
- `release_date`
- `new_magnets`
- `preview_images`

Static notes:

- Flutter route references point to `/home/searchNewPage`
- AOT shows `/api/v2/search` is called from `search_new_presenter.dart`
- AOT also shows a `?type=keyword` navigation hint in `home_page.dart`, which likely relates to UI routing rather than the raw API requirement

### `GET /api/v1/search_magnet`

Purpose:

- Magnet keyword search

Required query params:

```text
q
```

Validated example:

```http
GET https://jdforrepam.com/api/v1/search_magnet?q=KV-316
jdsignature: {generated_value}
```

Observed response fields:

- `magnets`
- `current_page`

Observed magnet fields:

- `id`
- `title`
- `hash`
- `size`
- `files_count`
- `created_at`

### `/api/v2/search_image`

Current verification status:

- `GET /api/v2/search_image` returned `HTTP 404`

Static notes from AOT:

- This route is referenced by `HomePagePresenter::getImageSearchResult`
- The presenter passes both `params` and `queryParameters`
- The flow is tied to image pick/upload pages:
  - `search_image_page.dart`
  - `search_image_result_page.dart`
- The code path shows upload-failure handling, which strongly suggests this endpoint is not a plain GET API and likely expects an uploaded image payload

Current inference:

- This endpoint is probably an image-upload search API, likely multipart/form-data or another file-upload form
- It should be tested later with an actual image payload rather than with the generic GET helper

### `POST /api/v1/sessions`

Purpose:

- User login

Request method:

- `POST`

Observed transport style from AOT:

- Flutter builds a `FormData.fromMap(...)`
- The request is sent via the `params` argument, not as a plain GET query

Confirmed minimum required fields:

- `username`
- `password`

Additional fields confirmed in AOT request construction:

- `device_uuid`
- `device_name`
- `device_model`
- `platform`
- `system_version`
- `app_channel`
- `app_version`
- `app_version_number`

Observed fixed/default values:

- `platform=android`
- `app_version=official`
- `app_version_number=1.9.35`
- `app_channel` defaults to `official` unless `key_app_channel` exists in local storage

Observed behavior:

- `GET /api/v1/sessions` -> `404`
- `POST` without body -> `ParameterInvalid: username`
- `POST` with `username` but empty `password` -> `ParameterInvalid: password`

Static validation notes from AOT:

- Login UI validates the username/email field with:

```text
^[a-zA-Z0-9_.@-]{5,50}$
```

- The login page label says `Please input email or username`
- Password is collected as a normal text field value and then submitted in the form payload

Current inference:

- The server login contract is already clear enough to implement in a script
- A real login attempt should use multipart or form-style POST data with the device/app metadata above
- Successful login likely returns user info and a token that later becomes the `authorization` header value

Successful live login observed:

- `success:1`
- `data.user`
- `data.following_tags`
- `data.token`

Observed `data.user` fields:

- `id`
- `username`
- `email`
- `is_vip`
- `vip_expired_at`
- `want_watch_count`
- `watched_count`
- `promote_users_count`
- `share_url`
- `promotion_code`
- `promotion_days`
- `checkin_days`
- `last_checkin_at`

Observed token usage:

- The raw `data.token` value can be sent back directly in the `authorization` header
- No `Bearer ` prefix has been confirmed so far

Practical helper usage:

```powershell
$env:JAVDB_AUTHORIZATION="your_token"
python F:\codx\javdb_api\src\javdb_api_client.py get /api/v1/about
```

### `GET /api/v1/movies/%s/play`

Purpose:

- Start playback for a movie source

Path parameter:

- `%s` is the movie id

Observed behavior:

- `GET` returns `JWTVerificationError` when unauthenticated
- `POST` returns `404`

Validated examples:

```http
GET https://jdforrepam.com/api/v1/movies/bKOeYA/play
jdsignature: {generated_value}
```

```json
{
  "success": 0,
  "action": "JWTVerificationError",
  "message": "請登錄帳號",
  "data": null
}
```

Static notes from AOT:

- `movie_presenter.dart` calls `/api/v1/movies/%s/play`
- The AOT request builder includes these query fields:
  - `source_id`
  - `from_rankings`
  - `operation`

Current inference:

- This endpoint is a signed `GET` endpoint
- It requires authenticated user context
- Additional playback query params are likely required after login

### `GET /api/v1/movies/%s/resume_play`

Purpose:

- Resume playback from a specific episode/resolution/source combination

Path parameter:

- `%s` is the movie id

Observed behavior:

- `GET` returns `JWTVerificationError` when unauthenticated
- `POST` returns `404`

Validated examples:

```http
GET https://jdforrepam.com/api/v1/movies/bKOeYA/resume_play
jdsignature: {generated_value}
```

```json
{
  "success": 0,
  "action": "JWTVerificationError",
  "message": "請登錄帳號",
  "data": null
}
```

Static notes from AOT:

- `common_tools.dart` calls `/api/v1/movies/%s/resume_play`
- The AOT request builder includes these query fields:
  - `source_id`
  - `episode`
  - `resolution`
  - `platform=android`

Current inference:

- This endpoint is a signed `GET` endpoint
- It requires authenticated user context
- A valid logged-in request likely also needs the playback-specific query parameters above

## Verified Endpoint Index

This index is a quick navigation layer over the detailed endpoint sections above.

### Public signed routes

- `GET /api/v1/startup`
- `GET /api/v1/about`
- `GET /api/v1/helps`
- `GET /api/v3/plans`
- `GET /api/v1/movies/latest`
- `GET /api/v4/movies/%s`
- `GET /api/v1/movies/%s/magnets`
- `GET /api/v1/movies/%s/reviews`
- `GET /api/v2/search`
- `GET /api/v1/search_magnet`
- `GET /api/v1/reviews/hotly`
- `GET /api/v1/actors`
- `GET /api/v1/series`

### Public entrypoints that produce auth context

- `POST /api/v1/sessions`

### Auth-gated routes already confirmed

- `POST /api/v1/movies/%s/reviews/%s/report`
- `DELETE /api/v1/movies/%s/reviews/%s`
- `GET /api/v1/movies/%s/play`
- `GET /api/v1/movies/%s/resume_play`
- `GET /api/v1/users`
- `GET /api/v1/users/recent_viewed`
- `GET /api/v1/wallets`
- `GET /api/v1/wallets/usdt_chain_types`
- `GET /api/v1/movies/top`

### Public mutation that needs re-checking

- `POST /api/v1/movies/%s/reviews/%s/like`

Reason for caution:

- It succeeded anonymously in the current test round
- That is unusual for a social action endpoint
- It should be retried later with a clean IP/device context before treating anonymous likes as a stable contract

## Interface Reference

This section rewrites the currently verified routes in a more API-doc style format.

Unless stated otherwise, all interfaces below require:

```http
jdsignature: {generated_value}
user-agent: Mozilla/5.0
accept: application/json
```

### 1. Startup configuration

**Endpoint**

```http
GET /api/v1/startup
```

**Purpose**

- Load startup configuration used by the Android app
- Return splash ad data, startup settings, backup domains, and preset keywords

**Auth**

- Requires `jdsignature`
- Does not currently require `authorization`

**Query parameters**

| Name | Required | Example | Notes |
| --- | --- | --- | --- |
| `last_ad_id` | Yes | empty string | Current known-good request sends an empty value |
| `platform` | Yes | `android` | APK-specific client platform |
| `app_channel` | Yes | `official` | Distribution channel |
| `app_version` | Yes | `official` | Channel-style app version label |
| `app_version_number` | Yes | `1.9.35` | Numeric client version |

**Success response**

Top-level `data` fields observed:

- `splash_ad`
- `user`
- `backup_domains_data`
- `recent_keywords`
- `recent_magnet_keywords`
- `settings`
- `feedback`
- `logging_enabled`
- `recognize_actor_enabled`
- `recognize_movie_enabled`
- `web_image_prefix`
- `ypay_payment_enabled`

**Common failure cases**

- Missing `jdsignature` -> `ParameterInvalid`
- Invalid `jdsignature` -> `InvalidSignature`
- Missing required query params -> `ParameterInvalid`

### 2. About links

**Endpoint**

```http
GET /api/v1/about
```

**Purpose**

- Return official site links, app distribution links, and community links

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

- None confirmed

**Success response**

- `data` is an array-like grouped link structure
- Each item observed contains:
  - `name`
  - `meta`
  - `url`

**Typical use**

- Startup or settings page link aggregation

### 3. Help center

**Endpoint**

```http
GET /api/v1/helps
```

**Purpose**

- Return FAQ/help-center content

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

| Name | Required | Example | Notes |
| --- | --- | --- | --- |
| `category` | Yes | `app` | `app` and `common` were both accepted |

**Success response**

- `helps`
- `current_page`

Each help item contains:

- `question`
- `answer`

**Known behavior**

- Missing `category` -> `ParameterInvalid: category`

### 4. Subscription plans

**Endpoint**

```http
GET /api/v3/plans
```

**Purpose**

- Return subscription plans and payment channel metadata

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

- None confirmed

**Success response**

Top-level fields observed:

- `plan_message`
- `unavailable_plan_message`
- `intros`
- `plans`
- `platforms`

Observed plan item fields:

- `id`
- `name`
- `price`
- `origin_price`
- `currency_unit`
- `currency_symbol`
- `days`

Observed payment platform fields:

- `platforms[].id`
- `platforms[].channels[].id`
- `platforms[].channels[].methods[].id`
- `platforms[].channels[].methods[].price_type`
- `platforms[].channels[].methods[].limited_prices`

### 5. Latest movies

**Endpoint**

```http
GET /api/v1/movies/latest
```

**Purpose**

- Return the latest movie feed

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

- No required parameters confirmed for the default first page

**Success response**

- `movies`
- `current_page`

Each movie item commonly includes:

- `id`
- `number`
- `title`
- `origin_title`
- `thumb_url`
- `cover_url`
- `duration`
- `magnets_count`
- `can_play`
- `play_subtitle`
- `has_preview_video`
- `has_cnsub`
- `has_preview_images`
- `release_date`
- `new_magnets`
- `preview_images`

### 6. Movie detail

**Endpoint**

```http
GET /api/v4/movies/{movie_id}
```

**Purpose**

- Return movie detail, related movies, preview assets, and metadata

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Path parameters**

| Name | Required | Example | Notes |
| --- | --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` | JavDB internal movie id |

**Success response**

Top-level fields observed:

- `share_info`
- `show_vip_banner`
- `movie`

Observed detailed `movie` fields:

- `id`
- `type`
- `number`
- `number_letter`
- `title`
- `origin_title`
- `summary`
- `thumb_url`
- `cover_url`
- `has_cnsub`
- `has_preview_images`
- `duration`
- `score`
- `reviews_count`
- `comments_count`
- `want_watch_count`
- `watched_count`
- `magnets_count`
- `has_preview_video`
- `can_play`
- `play_subtitle`
- `play_sources`
- `release_date`
- `top_rankings`
- `maker_id`
- `maker_name`
- `director_id`
- `director_name`
- `publisher_id`
- `publisher_name`
- `series_id`
- `series_name`
- `tags`
- `actors`
- `review`
- `preview_video_url`
- `preview_images`
- `relative_movies`
- `actor_movies`

**Notes**

- `preview_video_url` already includes its own playback signing query items in observed responses
- For movies with `can_play: true`, authenticated requests can reveal non-empty `play_sources`
- Example: `GET /api/v4/movies/gyQbKZ` with a valid `authorization` header returned `play_sources: [{ "id": 4, "name": "播放源4" }]`

### 7. Movie magnets

**Endpoint**

```http
GET /api/v1/movies/{movie_id}/magnets
```

**Purpose**

- Return all known magnet resources for a movie

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |

**Success response**

- `magnets`

Each magnet item observed includes:

- `name`
- `hash`
- `size`
- `cnsub`
- `hd`
- `files_count`
- `created_at`
- `pikpak_url`

### 8. Movie reviews list

**Endpoint**

```http
GET /api/v1/movies/{movie_id}/reviews
```

**Purpose**

- Return review/comment list for a movie

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |

**Success response**

- `reviews`

Each review item observed includes:

- `id`
- `user_id`
- `username`
- `watched_count`
- `status`
- `status_title`
- `score`
- `content`
- `likes_count`
- `liked`
- `created_at`

**Known behavior**

- Some movie ids return an empty array
- No pagination or ordering parameter has been confirmed yet

### 9. Like a review

**Endpoint**

```http
POST /api/v1/movies/{movie_id}/reviews/{review_id}/like
```

**Purpose**

- Like a review

**Auth**

- Requires `jdsignature`
- `authorization` was not required in the current test round, but this should be treated as provisional

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |
| `review_id` | Yes | `222247168` |

**Body**

- No request body confirmed

**Success response**

```json
{
  "success": 1,
  "action": null,
  "message": null,
  "data": null
}
```

**Known behavior**

- `GET` on this route returned `404`
- `POST` succeeded in the current test round

### 10. Report a review

**Endpoint**

```http
POST /api/v1/movies/{movie_id}/reviews/{review_id}/report
```

**Purpose**

- Report a review

**Auth**

- Requires `jdsignature`
- Also appears to require `authorization`

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |
| `review_id` | Yes | `222247168` |

**Known behavior**

- `GET` on this route returned `404`
- `POST` without login returned `JWTVerificationError`

### 11. Delete a review

**Endpoint**

```http
DELETE /api/v1/movies/{movie_id}/reviews/{review_id}
```

**Purpose**

- Delete a review owned by the authenticated user, or invoke the review deletion flow used by the app

**Auth**

- Requires `jdsignature`
- Also appears to require `authorization`

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |
| `review_id` | Yes | `222247168` |

**Known behavior**

- `GET` on this route returned `404`
- `DELETE` without login returned `JWTVerificationError`

### 12. Keyword movie search

**Endpoint**

```http
GET /api/v2/search
```

**Purpose**

- Search movies by keyword, code, or free-text title input

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

| Name | Required | Example | Notes |
| --- | --- | --- | --- |
| `q` | Yes | `KV-316` | Search keyword |
| `page` | No | `2` | Pagination is accepted |

**Success response**

- `movies`
- `current_page`

Movie item shape is very close to `/api/v1/movies/latest`.

**Known behavior**

- Missing `q` -> `ParameterInvalid: q`
- `page=2` was accepted and returned `current_page: 2`

### 13. Magnet search

**Endpoint**

```http
GET /api/v1/search_magnet
```

**Purpose**

- Search magnet resources by keyword

**Auth**

- Requires `jdsignature`
- No `authorization` required in current tests

**Query parameters**

| Name | Required | Example |
| --- | --- | --- |
| `q` | Yes | `KV-316` |

**Success response**

- `magnets`
- `current_page`

Observed item fields:

- `id`
- `title`
- `hash`
- `size`
- `files_count`
- `created_at`

### 14. Image search

**Endpoint**

```http
/api/v2/search_image
```

**Purpose**

- Likely search movies by uploaded image

**Current status**

- `GET` returned `404`
- AOT evidence strongly suggests this is an upload-style route, not a plain query-only route

**Probable request style**

- multipart upload
- may combine form fields with file payload

**Documentation status**

- Not yet ready for a stable contract section

### 15. Login

**Endpoint**

```http
POST /api/v1/sessions
```

**Purpose**

- Authenticate a user and return a reusable token

**Auth**

- Requires `jdsignature`
- Does not require prior `authorization`

**Request body**

Observed or reconstructed multipart fields:

| Name | Required | Example | Notes |
| --- | --- | --- | --- |
| `username` | Yes | `demo_user` | Username or email |
| `password` | Yes | `demo_pass` | Plain submitted password string |
| `device_uuid` | Yes | UUID | Device identifier |
| `device_name` | Yes | `Android` | Device name |
| `device_model` | Yes | `Android` | Device model |
| `platform` | Yes | `android` | Fixed by current APK flow |
| `system_version` | Yes | `14` | Android version string |
| `app_channel` | Yes | `official` | Defaults to `official` in current analysis |
| `app_version` | Yes | `official` | Client channel version |
| `app_version_number` | Yes | `1.9.35` | Numeric app version |

**Transport**

- Flutter AOT shows `FormData.fromMap(...)`
- Multipart form submission is the safest currently confirmed request style

**Success response**

Observed top-level fields under `data`:

- `user`
- `following_tags`
- `token`

Observed `data.user` fields:

- `id`
- `username`
- `email`
- `is_vip`
- `vip_expired_at`
- `want_watch_count`
- `watched_count`
- `promote_users_count`
- `share_url`
- `promotion_code`
- `promotion_days`
- `checkin_days`
- `last_checkin_at`

**Token usage**

- Replay `data.token` directly as the `authorization` header value
- Keep `jdsignature` on subsequent requests
- No confirmed `Bearer ` prefix is required

### 16. Start play

**Endpoint**

```http
GET /api/v1/movies/{movie_id}/play
```

**Purpose**

- Start movie playback for a selected source

**Auth**

- Requires `jdsignature`
- Requires `authorization` according to current live tests

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |

**Static query fields inferred from AOT**

- `source_id`
- `from_rankings`
- `operation`

**Known behavior**

- Unauthenticated request returned `JWTVerificationError`
- `POST` on this route returned `404`

**Additional live verification with authenticated user context**

- `GET /api/v1/movies/gyQbKZ/play` with login but without `source_id` returned:
  - `success:0`
  - `message: "此影片不支持在線播放"`
- `GET /api/v1/movies/gyQbKZ/play?source_id=4` returned:
  - `success:0`
  - `action: "PermissionDeniedToPayment"`
  - `message: "需要VIP權限才能訪問此內容"`
- Adding `from_rankings=1&operation=play` did not change that VIP-gated result

**Current refined inference**

- Authentication is necessary but not always sufficient
- For playable titles, `source_id` is a meaningful parameter
- Non-VIP accounts can still hit a business-permission gate after auth succeeds
- When `source_id` is omitted, the route may fall back to a generic "not supported online" message even for titles whose detail payload has `can_play: true`

### 17. Resume play

**Endpoint**

```http
GET /api/v1/movies/{movie_id}/resume_play
```

**Purpose**

- Resume playback using source, episode, and resolution context

**Auth**

- Requires `jdsignature`
- Requires `authorization` according to current live tests

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `movie_id` | Yes | `bKOeYA` |

**Static query fields inferred from AOT**

- `source_id`
- `episode`
- `resolution`
- `platform=android`

**Known behavior**

- Unauthenticated request returned `JWTVerificationError`
- `POST` on this route returned `404`

**Additional live verification with authenticated user context**

- `GET /api/v1/movies/gyQbKZ/resume_play?source_id=4&episode=1&resolution=720p&platform=android`
  returned `HTTP 200` with:
  - `Content-Type: text/plain; charset=utf-8`
  - body: `ERROR:PermissionDeniedToPayment`

**Current refined inference**

- `/resume_play` is not guaranteed to return JSON
- The route can return plain-text playback or error payloads
- VIP permission checks happen after auth and query validation

## Inferred Interface Skeletons

These routes have not all been fully live-verified yet, but the APK strings and presenter names are strong enough to justify a lightweight documentation skeleton.

Each entry below is intentionally conservative:

- `Purpose` reflects the most likely app behavior
- `Auth` is a current best guess, not a final contract
- `Parameters` include only names that are already visible from route shape or AOT request construction
- `To verify next` highlights the most useful live checks

### 18. User profile

**Endpoint**

```http
GET /api/v1/users
```

**Purpose**

- Likely return the current authenticated user profile and account summary

**Auth**

- Probably requires both `jdsignature` and `authorization`

**Expected response areas**

- User identity
- VIP state
- Counts such as watched or want-watch totals
- Possibly wallet or promotion summary fields

**To verify next**

- Whether unauthenticated access returns `JWTVerificationError`
- Whether this route is `GET` only
- Whether response shape matches `data.user` from login

**Live verification update**

- Plain signed request without login returned `JWTVerificationError`
- This route should now be treated as authenticated
- Authenticated request returned:
  - `data.user`
  - `data.banner_type`
- Observed `data.user` fields matched the login payload fields:
  - `id`
  - `username`
  - `email`
  - `is_vip`
  - `vip_expired_at`
  - `want_watch_count`
  - `watched_count`
  - `promote_users_count`
  - `share_url`
  - `promotion_code`
  - `promotion_days`
  - `checkin_days`
  - `last_checkin_at`

**Planned response contract**

Top-level shape likely follows the common envelope:

```json
{
  "success": 1,
  "action": null,
  "message": null,
  "data": {}
}
```

Most likely useful `data` fields:

- `user`
- account counters
- VIP or subscription state
- promotion-related fields

If the route mirrors login payloads, `data.user` may include:

- `id`
- `username`
- `email`
- `is_vip`
- `vip_expired_at`
- `want_watch_count`
- `watched_count`
- `promote_users_count`
- `share_url`
- `promotion_code`
- `promotion_days`
- `checkin_days`
- `last_checkin_at`

### 19. Forgot password

**Endpoint**

```http
POST /api/v1/users/forgot_password
```

**Purpose**

- Start password reset flow

**Auth**

- Likely public, signed route

**Probable body**

- Email or username field submitted as form data

**To verify next**

- Exact required field name
- Whether the route expects multipart or urlencoded form data
- Error message returned for empty submission

**Draft request contract**

Likely request body candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `email` | High probability | Most common recovery field |
| `username` | Possible alternative | App login accepts username or email |

Most likely request style:

- `POST`
- `multipart/form-data` or `application/x-www-form-urlencoded`
- still signed with `jdsignature`

**Planned success semantics**

- `success:1` with no meaningful payload, or
- a message indicating recovery email/code was sent

### 20. Reset password

**Endpoint**

```http
POST /api/v1/users/reset_password
```

**Purpose**

- Complete password reset using a verification code or token

**Auth**

- Likely public, signed route

**Probable body**

- Account identifier
- Verification code
- New password

**To verify next**

- Required field names
- Whether the route is paired with `/users/forgot_password`

**Draft request contract**

Likely request body candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `email` or `username` | Possible | Depends on reset flow |
| `code` | High probability | Verification code |
| `password` | High probability | New password |
| `password_confirmation` | Possible | If the server expects confirmation |

**Planned success semantics**

- `success:1`
- possibly no payload beyond a success message

### 21. Change password

**Endpoint**

```http
POST /api/v1/users/change_password
```

**Purpose**

- Change password for the logged-in user

**Auth**

- Likely authenticated route

**Probable body**

- Current password
- New password

**To verify next**

- Whether the route is `POST` or another write method
- Exact body fields

**Draft request contract**

Likely authenticated form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `old_password` | High probability | Current password |
| `password` | High probability | New password |
| `password_confirmation` | Possible | Common server-side validation pair |

**Planned success semantics**

- `success:1`
- updated auth state may or may not require re-login

### 22. Change username

**Endpoint**

```http
POST /api/v1/users/change_username
```

**Purpose**

- Update current username

**Auth**

- Likely authenticated route

**Probable body**

- New username

**To verify next**

- Required field name
- Username validation behavior

**Draft request contract**

Likely form field:

| Name | Likely required | Notes |
| --- | --- | --- |
| `username` | High probability | New username |

Possible validation source:

- reuse of the login-page username pattern
- server-side uniqueness checks

### 23. Resend activation code

**Endpoint**

```http
POST /api/v1/users/resend_activation_code
```

**Purpose**

- Resend account activation or email verification code

**Auth**

- Likely public or semi-authenticated depending on registration state

**To verify next**

- Whether it accepts email, username, or token context
- Whether a partially logged-in account is required

**Draft request contract**

Most likely inputs:

- `email`
- or an activation token bound to current session/device state

**Planned success semantics**

- `success:1`
- delivery confirmation message

### 24. Activate registration

**Endpoint**

```http
POST /api/v1/users/activate_registration
```

**Purpose**

- Complete registration activation with a verification code

**Auth**

- Likely public, signed route

**Probable body**

- Verification code
- Possibly username or email

**To verify next**

- Exact required form fields

**Draft request contract**

Likely body candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `code` | High probability | Activation code |
| `email` or `username` | Possible | Registration identifier |

**Planned success semantics**

- account activation success
- possibly immediate login-like user payload, though not yet evidenced

### 25. Additional user info

**Endpoint**

```http
GET /api/v1/users/additional
```

**Purpose**

- Likely return supplementary account state not included in the main user profile

**Auth**

- Probably authenticated

**Expected response areas**

- Extra counters
- Feature flags
- Billing or promotion state

**To verify next**

- Whether it is a pure `GET`
- Whether it overlaps with `/api/v1/users`

**Planned response contract**

Most likely `data` areas:

- flags for optional app features
- additional counters not always needed on first login
- wallet, promotion, or notification summary blocks

### 26. Recent viewed

**Endpoint**

```http
GET /api/v1/users/recent_viewed
```

**Purpose**

- Return the viewing history or recently opened movie list for the current user

**Auth**

- Probably authenticated

**Probable query parameters**

- `page`

**Expected response**

- A paginated movie list

**To verify next**

- Pagination support
- Whether unauthenticated requests return `JWTVerificationError`

**Live verification update**

- Plain signed request without login returned `JWTVerificationError`
- This route should now be treated as authenticated
- Authenticated request returned `data.movies`
- The movie item shape matches the reusable movie list object already observed on public feeds
- No `current_page` field was confirmed in the authenticated sample captured during this round

**Planned query contract**

Likely query candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `page` | Low | Common for paginated feeds |

**Planned response contract**

Likely response fields:

- `movies`
- `current_page`

Movie item shape is likely close to:

- `/api/v1/movies/latest`
- `/api/v2/search`

### 27. Transaction logs

**Endpoint**

```http
GET /api/v1/users/transaction_logs
```

**Purpose**

- Return payment or account transaction history

**Auth**

- Almost certainly authenticated

**Expected response**

- Paginated billing entries

**To verify next**

- Supported filters and pagination

**Planned query contract**

Likely query candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `page` | Low | Expected for log pagination |
| `type` | Possible | Payment or transaction type filter |

**Planned response contract**

Expected entry shape:

- `id`
- amount or currency fields
- status
- created timestamp
- payment channel or transaction type

### 28. Unpaid tickets

**Endpoint**

```http
GET /api/v1/users/unpaid_tickets
```

**Purpose**

- Likely return unpaid orders or pending payment tickets

**Auth**

- Almost certainly authenticated

**To verify next**

- Whether result shape is order-oriented or plan-oriented

**Planned response contract**

Likely response fields:

- pending order id
- plan details
- payable amount
- payment deadline or status

### 29. Promotion logs

**Endpoint**

```http
GET /api/v1/users/promotion_logs
```

**Purpose**

- Return promotion or referral history

**Auth**

- Probably authenticated

**To verify next**

- Whether route is paginated
- Whether it references `promotion_code` data seen in login response

**Planned response contract**

Likely entry fields:

- `id`
- referred user or event description
- reward amount or days
- created timestamp

### 30. User feedback

**Endpoint**

```http
POST /api/v1/users/feedback
```

**Purpose**

- Submit user feedback to the service

**Auth**

- Probably authenticated, though anonymous feedback is also plausible

**Probable body**

- Feedback text
- Possibly contact metadata or category

**To verify next**

- Method and body fields
- Whether login is required

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `content` | High probability | Main feedback text |
| `contact` | Possible | Optional callback/contact string |
| `category` | Possible | Problem type |

**Planned success semantics**

- `success:1`
- possibly null `data`

### 31. Wallet summary

**Endpoint**

```http
GET /api/v1/wallets
```

**Purpose**

- Return wallet balance and withdrawal-related account data

**Auth**

- Authenticated

**Expected response areas**

- Balance
- Bind status
- Withdrawal options

**To verify next**

- Whether this route is the wallet homepage payload

**Live verification update**

- Plain signed request without login returned `JWTVerificationError`
- This route should now be treated as authenticated
- Authenticated request returned:
  - `minimum_withdraw_amount`
  - `maximum_withdraw_amount`
  - `withdraw_fee`
  - `current_level`
  - `current_rate`
  - `next_level`
  - `next_rate`
  - `yesterday_income`
  - `today_income`
  - `withdrawable`
  - `total_income`
  - `users_count`
  - `withdraw_methods`

**Planned response contract**

Likely `data` areas:

- current balance
- withdrawable balance
- verification state
- linked withdraw accounts
- available chain or payment options

### 32. Wallet email verification

**Endpoint**

```http
POST /api/v1/wallets/verify_email
```

**Purpose**

- Confirm wallet-related email verification step

**Auth**

- Probably authenticated

**To verify next**

- Required form fields
- Whether it accepts a verification code

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `code` | High probability | Email verification code |
| `email` | Possible | If not implied by session |

**Planned success semantics**

- verified wallet-email state updated

### 33. Send wallet verification email

**Endpoint**

```http
POST /api/v1/wallets/send_verification_email
```

**Purpose**

- Trigger a verification email for wallet actions

**Auth**

- Probably authenticated

**To verify next**

- Rate-limit behavior
- Whether any request body is required

**Draft request contract**

Most likely possibilities:

- empty authenticated `POST`
- or `POST` with `email` if verification target is editable

**Planned success semantics**

- delivery confirmation message
- rate-limit error if repeated too quickly

### 34. Bind withdraw account

**Endpoint**

```http
POST /api/v1/wallets/bind_withdraw_account
```

**Purpose**

- Bind a withdrawal account such as bank or crypto destination

**Auth**

- Authenticated

**Probable body**

- Account type
- Account identifier
- Verification data

**To verify next**

- Supported withdraw account types
- Validation fields

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `type` | High probability | Bank / USDT / wallet type |
| `account` | High probability | Destination identifier |
| `name` | Possible | Account holder |
| `bank_code` | Possible | Needed for bank withdrawals |
| `chain_type` | Possible | Needed for USDT routes |
| `code` | Possible | Verification code |

**Planned success semantics**

- new linked account returned, or
- null payload with success message

### 35. List binded withdraw accounts

**Endpoint**

```http
GET /api/v1/wallets/binded_withdraw_accounts
```

**Purpose**

- Return all currently linked withdrawal accounts

**Auth**

- Authenticated

**To verify next**

- Response item shape

**Planned response contract**

Expected item fields:

- internal account id
- account type
- masked account value
- display label
- default flag

### 36. Unbind withdraw account

**Endpoint**

```http
DELETE /api/v1/wallets/unbind_withdraw_account/{account_id}
```

**Purpose**

- Remove a linked withdrawal account

**Auth**

- Authenticated

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `account_id` | Yes | internal id | Exact id format unknown |

**To verify next**

- Required method
- Whether confirmation fields are required

**Planned success semantics**

- `success:1`
- account removed from subsequent binded-account listing

### 37. Withdraw logs

**Endpoint**

```http
GET /api/v1/wallets/withdraw_logs
```

**Purpose**

- Return withdrawal history

**Auth**

- Authenticated

**To verify next**

- Pagination
- Status fields

**Planned response contract**

Likely entry fields:

- `id`
- amount
- fee
- status
- created timestamp
- destination account summary

### 38. Rebate logs

**Endpoint**

```http
GET /api/v1/wallets/rebate_logs
```

**Purpose**

- Return rebate or referral earnings history

**Auth**

- Authenticated

**To verify next**

- Whether this overlaps with promotion logs or wallet commission records

**Planned response contract**

Likely entry fields:

- rebate source
- amount
- status
- created timestamp

### 39. USDT chain types

**Endpoint**

```http
GET /api/v1/wallets/usdt_chain_types
```

**Purpose**

- Return available USDT chain/network options for withdrawal or payment

**Auth**

- Probably authenticated

**To verify next**

- Response item fields such as chain code, label, and fee

**Live verification update**

- Plain signed request without login returned `JWTVerificationError`
- This route should now be treated as authenticated
- Authenticated request returned `data.chain_types`
- Observed value: `["TRC20"]`

**Planned response contract**

Expected item fields:

- chain identifier
- display name
- fee or rate metadata
- enabled flag

### 40. SFPay banks

**Endpoint**

```http
GET /api/v1/wallets/sfpay_banks
```

**Purpose**

- Return bank options for a payment processor named SFPay

**Auth**

- Probably authenticated

**To verify next**

- Whether route is tied to a specific region or payment channel

**Planned response contract**

Expected item fields:

- bank code
- bank name
- display order
- availability flag

### 41. Wallet withdraw

**Endpoint**

```http
POST /api/v2/wallets/withdraw
```

**Purpose**

- Submit a withdrawal request

**Auth**

- Authenticated

**Probable body**

- Amount
- Target account id
- Chain or bank selection
- Verification code

**To verify next**

- Minimum required fields
- Error behavior on invalid amount

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `amount` | High probability | Withdraw amount |
| `withdraw_account_id` | High probability | Bound destination id |
| `chain_type` or `bank_code` | Possible | Depends on withdraw method |
| `code` | Possible | Email/security verification |

**Planned success semantics**

- withdrawal request created
- log entry may become visible in `/api/v1/wallets/withdraw_logs`

### 42. V4 plans

**Endpoint**

```http
GET /api/v4/plans
```

**Purpose**

- Likely a newer subscription plan payload than `/api/v3/plans`

**Auth**

- Live verification shows this route also requires `authorization`

**To verify next**

- Whether it supersedes `/api/v3/plans`
- Differences in response structure

**Planned response contract**

Likely top-level areas:

- plan list
- platform/channel list
- upgrade messaging
- region- or device-specific availability flags

Expected overlap with `/api/v3/plans`:

- `plans`
- payment platform metadata
- plan intro copy

### 43. Payment order v2

**Endpoint**

```http
POST /api/v2/plans/payment_order
```

**Purpose**

- Create a payment order for a selected subscription plan

**Auth**

- Probably authenticated

**Probable body**

- Plan id
- Platform or channel id
- Payment method id

**To verify next**

- Exact required ids and whether anonymous order creation is allowed

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `plan_id` | High probability | Selected plan |
| `platform_id` | High probability | Payment platform |
| `channel_id` | Possible | Intermediate payment channel |
| `method_id` | High probability | Payment method |
| `price` or `price_id` | Possible | If limited-price options exist |

**Planned success semantics**

- payment order object created
- response may include payment URL, QR payload, or order number

### 44. Payment order v3

**Endpoint**

```http
POST /api/v3/plans/payment_order
```

**Purpose**

- Newer payment-order creation route

**Auth**

- Probably authenticated

**To verify next**

- Structural differences from `/api/v2/plans/payment_order`

**Draft request contract**

Likely same core identifiers as v2:

- `plan_id`
- `platform_id`
- `channel_id`
- `method_id`
- optional price selector fields

**Planned response contract**

Possible added fields versus v2:

- richer cashier metadata
- expiration timestamp
- redirect URL or QR code payload

### 45. Top movies

**Endpoint**

```http
GET /api/v1/movies/top
```

**Purpose**

- Return ranked or top movie feed

**Auth**

- Live verification shows this route requires `authorization`

**Probable query parameters**

- `page`
- ranking period or category filters

**Expected response**

- Paginated `movies` list similar to latest/search feeds

**To verify next**

- Ranking filters
- Whether result shape includes ranking metadata

**Live verification update**

- Plain signed request without login returned `JWTVerificationError`
- This route should currently be treated as authenticated
- Authenticated request without extra query parameters returned `success:1`
- Observed top-level response fields:
  - `generated_at`
  - `movies`
  - `current_page`
- Observed extra movie field compared with generic list routes:
  - `ranking`

**Planned query contract**

Likely query candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `page` | Low | Standard pagination |
| `period` | Possible | Daily / weekly / monthly style ranking window |
| `type` | Possible | Ranking category |

**Planned response contract**

Likely fields:

- `movies`
- `current_page`
- optional ranking meta such as current period or tab

Movie item shape will likely match the reusable `movie` object used by latest/search feeds.

### 46. Recommended movies

**Endpoint**

```http
GET /api/v1/movies/recommend
```

**Purpose**

- Return movie recommendations

**Auth**

- Live verified as public signed route

**To verify next**

- Whether recommendations are global or user-personalized

**Planned query contract**

Likely query candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `page` | Low | If recommendation feed is paginated |
| `period` | Possible | If tied to recommendation windows |

**Planned response contract**

Likely fields:

- `movies`
- `current_page`
- optional recommendation label or period metadata

### 47. Recommendation periods

**Endpoint**

```http
GET /api/v1/movies/recommend_periods
```

**Purpose**

- Return available recommendation time windows or ranking periods

**Auth**

- Probably public signed route

**To verify next**

- Whether this route feeds the UI filter for `/movies/recommend`

**Planned response contract**

Likely fields:

- period list entries
- current/default period

Possible entry shape:

- `id`
- `name`
- `value`
- `selected`

### 48. May also like

**Endpoint**

```http
GET /api/v1/movies/may_also_like
```

**Purpose**

- Return a related-recommendation list, likely tied to a movie or viewing context

**Auth**

- Unknown; could be public or personalized

**Probable query parameters**

- Some movie or context identifier

**To verify next**

- Required parameter names

**Planned query contract**

Likely query candidates:

| Name | Likely required | Notes |
| --- | --- | --- |
| `movie_id` | Medium | Most likely recommendation anchor |
| `page` | Low | If list can paginate |

**Planned response contract**

Likely fields:

- `movies`
- optional context label

### 49. Movie tags

**Endpoint**

```http
GET /api/v1/movies/tags
```

**Purpose**

- Return a filtered movie list for discovery pages that are driven by UI tag state
- Despite the route name, current APK call sites treat this as a `MovieEntity` list endpoint rather than a tag-catalog endpoint

**Auth**

- Public signed route

**Observed request patterns**

- This route requires `filter_by`
- Empty request returned `ParameterInvalid: filter_by`
- A naive guess such as `filter_by=can_play` returned `HTTP 500`
- AOT analysis shows `filter_by` is not a single global enum baked into the APK
- In category and maker-detail flows it is typically assembled as `<group>:<id1,id2,...>`
- Confirmed UI-derived filter tokens include:
  - `main:m`
  - `main:p`
- Bare `filter_by=main:m&page=1&limit=48` and `filter_by=main:p&page=1&limit=48` still returned `HTTP 500`
- The following combinations succeeded in live requests:
  - `/api/v1/movies/tags?filter_by=main:m&sort_by=release_date&order_by=desc&page=1&limit=48`
  - `/api/v1/movies/tags?filter_by=main:p&sort_by=release_date&order_by=desc&page=1&limit=48`

**Observed query parameter families**

- Home presenter:
  - `filter_by`
  - `sort_by`
  - `page`
  - `limit=48`
- Category presenter:
  - `filter_by`
  - `sort_by`
  - `order_by`
  - `page`
  - `limit=48`
- Actor detail presenter:
  - `filter_by`
  - `filter_by_tags`
  - `sort_by`
  - `order_by`
  - `page`
  - `limit=48`
- Maker detail presenter:
  - `filter_by`
  - `sort_by`
  - `order_by`
  - `page`
  - `limit=48`
- My list detail presenter:
  - `type=0`
  - `filter_by`
  - `sort_by`
  - `page`
  - `limit=48`

**Observed response contract**

- Successful responses returned:
  - `movies`
  - `has_collected`
  - `current_page`

**Static evidence**

- AOT call sites using this route:
  - `HomePagePresenter::getFocusMovies`
  - `CategoryListPresenter::getCategoryMovies`
  - `ActorInfoNewPagePresenter::getActorMovies`
  - `MakerDetailPagePresenter::getDetailMovies`
  - `MyListsDetailPagePresenter::getListsMovies`
- Category provider injects two known synthetic filter items:
  - `group="main", id="m", name="Download"`
  - `group="main", id="p", name="Playable"`
- Actor detail pages also pass `filter_by_tags=<comma-separated TagItem.id>` for secondary tag selection

### 50. Hot reviews

**Endpoint**

```http
GET /api/v1/reviews/hotly
```

**Purpose**

- Return trending or featured reviews

**Auth**

- Probably public signed route

**Expected response**

- Review list, possibly with movie preview context

**To verify next**

- Pagination
- Whether it includes nested movie info

**Planned query contract**

Live verified query behavior:

| Name | Likely required | Notes |
| --- | --- | --- |
| `period` | Yes | `week` is confirmed working |
| `page` | Low | Feed pagination not yet verified |

**Planned response contract**

Live observed fields:

- `reviews`

Each entry may include:

- review fields
- nested `movie` summary fields for navigation

**Live verification update**

- Missing `period` -> `ParameterInvalid: period`
- `period=week` -> `success:1`
- Observed review fields:
  - `id`
  - `user_id`
  - `username`
  - `watched_count`
  - `content`
  - `score`
  - `likes_count`
  - `liked`
  - `created_at`
- Observed nested `movie` fields:
  - `id`
  - `number`
  - `title`
  - `origin_title`
  - `score`
  - `thumb_url`
  - `release_date`

### 51. Actors list

**Endpoint**

```http
GET /api/v1/actors
```

**Purpose**

- Return actor listing or search results

**Auth**

- Probably public signed route

**Probable query parameters**

- `page`
- keyword or sort fields

**To verify next**

- Whether default listing works without query params

**Planned query contract**

Live verified query behavior:

| Name | Likely required | Notes |
| --- | --- | --- |
| `page` | Low | Pagination |
| `type` | Yes | `0` and `1` are both confirmed |
| `q` | Possible | Keyword search |
| `letter` | Possible | Initial-letter filter |
| `sort` | Possible | Popularity/recent sort |

**Planned response contract**

Live observed fields:

- `actors`
- `current_page`

Expected actor item shape:

- `id`
- `name`
- `avatar_url`
- movie count or popularity metadata
- collect/follow state

**Live verification update**

- Missing `type` -> `ParameterInvalid: type`
- `type=0` -> `success:1`
- `type=1` -> `success:1`
- Observed actor item fields:
  - `id`
  - `type`
  - `avatar_url`
  - `name`
  - `name_zht`
  - `other_name`
  - `uncensored`
  - `gender`
  - `videos_count`

Current inference:

- `type=0` appears to correspond to censored/mainstream actress listing
- `type=1` appears to correspond to uncensored actress listing

### 52. Actor detail

**Endpoint**

```http
GET /api/v1/actors/{actor_id}
```

**Purpose**

- Return actor profile and movie list

**Auth**

- Probably public signed route

**Path parameters**

| Name | Required | Example |
| --- | --- | --- |
| `actor_id` | Yes | internal id | Exact format not yet verified |

**Expected response areas**

- Actor profile
- Related movies
- Collect/follow state

**To verify next**

- Whether actor detail is paginated by movies

**Live verification update**

- `GET /api/v1/actors/kzx6` returned `success:1`
- Observed top-level fields:
  - `share_info`
  - `has_collected`
  - `actor`
  - `filter_tags`
  - `tags`
- Observed `actor` fields:
  - `id`
  - `type`
  - `avatar_url`
  - `name`
  - `name_zht`
  - `other_name`
  - `birthday`
  - `age`
  - `cons`
  - `blood_type`
  - `height`
  - `bust`
  - `cup`
  - `waist`
  - `hips`
  - `birthplace`
  - `twitter_id`
  - `instagram_id`
  - `videos_count`
- `filter_tags` entries currently expose:
  - `id`
  - `name`
- `tags` entries currently expose:
  - `id`
  - `name`
  - `videos_count`

Current inference:

- Actor detail is public
- This route returns profile and filter metadata first; a movie list may be loaded separately by the app

**Planned response contract**

Likely top-level fields:

- `actor`
- `movies`
- `current_page`

Possible `actor` fields:

- `id`
- `name`
- `avatar_url`
- `summary`
- ranking or popularity counters
- collect/follow state

### 53. Recommended actors

**Endpoint**

```http
GET /api/v1/actors/recommend
```

**Purpose**

- Return curated actor recommendations

**Auth**

- Probably public signed route

**Planned response contract**

Likely fields:

- `actors`
- optional `current_page`

**Live verification update**

- `GET /api/v1/rankings/actors?type=0` returned `success:1`
- `GET /api/v1/rankings/actors?type=1` returned `success:1`
- Observed top-level response field:
  - `actors`
- Observed actor ranking item fields:
  - `id`
  - `name`
  - `name_zht`
  - `other_name`
  - `avatar_url`

Current inference:

- Despite the route path, `/api/v1/rankings/actors` behaves like an actor ranking feed
- `type=0` and `type=1` likely split censored and uncensored actor rankings

### 54. Actor collect action

**Endpoint**

```http
POST /api/v1/actors/{actor_id}/collect_actions
```

**Purpose**

- Follow, collect, or uncollect an actor

**Auth**

- Probably authenticated

**To verify next**

- Whether the route uses `POST` only and toggles by body field
- Whether `DELETE` is also supported

**Draft request contract**

Likely behavior patterns:

1. Toggle-style `POST` with no body
2. `POST` with an action field such as `type=collect` / `type=uncollect`

Likely success response:

- null or small state payload
- updated collect flag/count

### 55. Batch actor uncollection

**Endpoint**

```http
POST /api/v1/actors/batch_uncollection
```

**Purpose**

- Remove multiple actor collections in one request

**Auth**

- Authenticated

**Probable body**

- Array-like actor ids

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `actor_ids` | High probability | Comma-separated or repeated ids |

**Planned success semantics**

- bulk removal completed
- null payload or updated count

### 56. Directors list/detail/collect

**Endpoints**

```http
GET /api/v1/directors
GET /api/v1/directors/{director_id}
POST /api/v1/directors/{director_id}/collect_actions
```

**Purpose**

- Director discovery, profile detail, and collect actions

**Auth**

- List/detail probably public
- Collect action probably authenticated

**To verify next**

- Whether response shape mirrors actor endpoints

**Planned contracts**

Likely mirrors actor APIs:

- list route returns `directors` plus pagination
- detail route returns `director` plus related `movies`
- collect route behaves like actor collect action

### 57. Makers list/detail/collect

**Endpoints**

```http
GET /api/v1/makers
GET /api/v1/makers/{maker_id}
POST /api/v1/makers/{maker_id}/collect_actions
```

**Purpose**

- Studio or maker discovery, detail, and collect actions

**Auth**

- List/detail probably public
- Collect action probably authenticated

**Planned contracts**

Likely response areas:

- maker profile
- maker-associated movies
- collect state

### 58. Publisher detail

**Endpoint**

```http
GET /api/v1/publishers/{publisher_id}
```

**Purpose**

- Return publisher detail and related movie list

**Auth**

- Probably public signed route

**Planned response contract**

Likely fields:

- `publisher`
- `movies`
- `current_page`

### 59. Series list/detail/letters/collect

**Endpoints**

```http
GET /api/v1/series
GET /api/v1/series/{series_id}
GET /api/v1/series/letters
POST /api/v1/series/{series_id}/collect_actions
```

**Purpose**

- Series discovery, detail, alphabet index, and collect actions

**Auth**

- Discovery/detail probably public
- Collect action probably authenticated

**Planned contracts**

Likely route behavior:

- `/series` returns paginated series listing
- `/series/{series_id}` returns series detail and related movies
- `/series/letters` returns alphabet filter data
- collect route mirrors actor/director/maker collect semantics

**Live verification update for `/api/v1/series`**

- Missing `type` -> `ParameterInvalid: type`
- `type=0` -> `success:1`
- `type=1` -> `success:1`
- Observed top-level response fields:
  - `series`
  - `current_page`
- Observed series item fields:
  - `id`
  - `type`
  - `name`
  - `videos_count`

Current inference:

- `type=0` and `type=1` likely split censored and uncensored series catalogs

**Live verification update for `/api/v1/series/{series_id}`**

- `GET /api/v1/series/rY2v` returned `success:1`
- Observed top-level fields:
  - `share_info`
  - `has_collected`
  - `series`
- Observed `series` fields:
  - `id`
  - `type`
  - `name`
  - `videos_count`

Current inference:

- Series detail is public
- Like actor detail, the primary payload is metadata-first and may rely on separate list-loading requests for member movies

### 60. Lists endpoints

**Endpoints**

```http
GET /api/v1/lists
GET /api/v1/lists/simple
GET /api/v1/lists/{list_id}
POST /api/v1/lists/{list_id}/collect_actions
POST /api/v1/lists/{list_id}/movie_actions
GET /api/v1/lists/related
```

**Purpose**

- User or curated list discovery, list detail, collect actions, and per-list movie management

**Auth**

- Browse routes may be public
- Collect and movie-action routes are authenticated

**Observed route roles**

- `/lists` for full list discovery
- `/lists/simple` for lightweight selector data
- `/lists/{list_id}` for detail and member movies
- `/lists/{list_id}/collect_actions` for subscribe or favorite behavior
- `/lists/{list_id}/movie_actions` for adding or removing movies in a user-managed list
- `/lists/related` for related lists from a movie or list context

**Observed mutation request bodies**

- `POST /api/v1/lists/{list_id}/collect_actions`
  - `multipart/form-data`
  - `name=collect` or `name=uncollect`
- `POST /api/v1/lists/{list_id}/movie_actions`
  - `multipart/form-data`
  - `movie_id=<movie_id>`
  - `name=add` or `name=remove`

**Static evidence for `movie_actions`**

- The movie detail "Save to list" flow opens `MovieListsPage`, reads the current list item's boolean selected state, and maps it client-side:
  - `false -> "add"`
  - `true -> "remove"`
- The request is then sent to `/api/v1/lists/%s/movie_actions`
- My list detail page deletion uses the same route with fixed `name=remove`
- No evidence was found for `toggle`, `movie title`, `operation`, or `status` fields on this route

**Planned response contract**

List detail likely includes:

- `list`
- `movies`
- `current_page`

### 61. Code detail and code collect

**Endpoints**

```http
GET /api/v1/codes/{code}
POST /api/v1/codes/{code}/collect_actions
```

**Purpose**

- Resolve a movie code to detail and optionally collect it

**Auth**

- Detail likely public
- Collect likely authenticated

**To verify next**

- Whether `{code}` is a raw title code like `KV-316`

**Planned response contract**

Most likely behavior:

- `/codes/{code}` resolves directly to movie detail-like data, or
- returns a list if a code maps to multiple versions

Collect action likely mirrors other collect endpoints with authenticated toggle semantics.

### 62. Following tags suite

**Endpoints**

```http
GET /api/v1/following_tags
GET /api/v1/following_tags/{tag_id}
POST /api/v1/following_tags/{tag_id}/sort
POST /api/v1/following_tags/batch_destroy
POST /api/v1/following_tags/batch_push
```

**Purpose**

- Manage followed tags, tag ordering, deletion, and push settings

**Auth**

- Authenticated

**To verify next**

- Whether `/following_tags/{tag_id}` is detail or action-oriented

**Planned contracts**

Likely route behavior:

- `/following_tags` returns current followed tag list
- `/following_tags/{tag_id}` returns tag detail or followed-tag movie feed
- `/{tag_id}/sort` changes order weight
- `/batch_destroy` removes multiple followed tags
- `/batch_push` toggles push/notification state for multiple tags

**Draft body ideas**

| Route | Likely fields |
| --- | --- |
| `/following_tags/{tag_id}/sort` | `sort`, `direction`, or ordinal index |
| `/following_tags/batch_destroy` | `tag_ids` |
| `/following_tags/batch_push` | `tag_ids`, maybe push flag |

### 63. Tags v2

**Endpoint**

```http
GET /api/v2/tags
```

**Purpose**

- Return tag catalog or search/filter tag list in a newer API version

**Auth**

- Probably public signed route

**Planned response contract**

Likely fields:

- `tags`
- `current_page` or grouped structures
- optional search/filter metadata

### 64. Ads and splash logging

**Endpoints**

```http
GET /api/v1/ads
POST /api/v1/ads/splash_log
```

**Purpose**

- Load ads and submit splash ad exposure/click logs

**Auth**

- Likely signed routes without full login for basic ad serving

**To verify next**

- Whether `splash_log` accepts ad id, action type, and timestamp-ish fields

**Planned contracts**

Likely behavior:

- `/ads` returns ad placements or currently active ads
- `/ads/splash_log` records impression, close, or click events

**Draft body ideas for `/ads/splash_log`**

| Name | Likely required | Notes |
| --- | --- | --- |
| `ad_id` | High probability | Target ad |
| `action` | High probability | Impression/click/close style event |
| `position` | Possible | Splash/banner placement |

### 65. Articles list and detail

**Endpoints**

```http
GET /api/v1/articles
GET /api/v1/articles/{article_id}
```

**Purpose**

- Return article/news list and article detail

**Auth**

- Probably public signed routes

**Planned response contract**

Likely list fields:

- `articles`
- `current_page`

Likely article item/detail fields:

- `id`
- `title`
- `cover_url`
- `summary`
- `content`
- `created_at`

**Live verification update**

- `GET /api/v1/articles` returned `success:1`
- Observed list response fields:
  - `articles`
  - `current_page`
- Observed article list item fields:
  - `id`
  - `title`
  - `cover_url`
  - `author`
  - `category`
  - `released_at`
- `GET /api/v1/articles/y4Dr` returned `success:1`
- Observed article detail fields:
  - `id`
  - `title`
  - `origin_name`
  - `origin_url`
  - `cover_url`
  - `author`
  - `category`
  - `image_domain`
  - `content`
  - `released_at`
  - `related_movies`

### 66. Rankings suite

**Endpoints**

```http
GET /api/v1/rankings
GET /api/v1/rankings/actors
GET /api/v1/rankings/playbackP
```

**Purpose**

- Return ranking overview, actor rankings, and playback-related rankings

**Auth**

- Probably public signed routes

**To verify next**

- Query parameters for ranking category or period
- Meaning of `playbackP`

**Planned contracts**

Likely route roles:

- `/rankings` for movie rankings or ranking tabs overview
- `/rankings/actors` for actor ranking list
- `/rankings/playbackP` for playback/popularity ranking variant

**Likely query candidates**

| Name | Likely required | Notes |
| --- | --- | --- |
| `period` | Possible | Ranking window |
| `page` | Low | Feed pagination |
| `type` | Possible | Ranking subtype |

**Live verification update**

- `GET /api/v1/rankings` without params returned `ParameterInvalid: type`
- `GET /api/v1/rankings?type=0` returned `ParameterInvalid: period`
- `GET /api/v1/rankings?type=1` returned `ParameterInvalid: period`
- `GET /api/v1/rankings/actors` without params returned `ParameterInvalid: type`
- `GET /api/v1/rankings/actors?type=0` returned `success:1`
- `GET /api/v1/rankings/actors?type=1` returned `success:1`
- `GET /api/v1/rankings/playbackP` currently returned `HTTP 404`

Current refined inference:

- `/api/v1/rankings` requires at least both `type` and `period`
- `/api/v1/rankings/actors` requires `type` but did not require `period` in current tests
- `/api/v1/rankings/playbackP` may be obsolete, host-specific, or incorrectly recovered from static strings

### 67. Magnet apps

**Endpoint**

```http
GET /api/v1/magnet_apps
```

**Purpose**

- Return recommended third-party magnet client apps

**Auth**

- Probably public signed route

**Planned response contract**

Likely fields:

- `apps`

Each app item may include:

- `name`
- `icon_url`
- `download_url`
- `description`

### 68. Movie played log

**Endpoint**

```http
POST /api/v1/logs/movie_played
```

**Purpose**

- Submit movie playback log or progress event

**Auth**

- Probably authenticated

**Probable body**

- Movie id
- Source id
- Episode or progress markers

**Draft request contract**

Likely form fields:

| Name | Likely required | Notes |
| --- | --- | --- |
| `movie_id` | High probability | Played movie |
| `source_id` | High probability | Playback source |
| `episode` | Possible | Episode index |
| `resolution` | Possible | Selected resolution |
| `progress` | Possible | Playback progress or seconds |
| `operation` | Possible | Start/end/update event type |

**Planned success semantics**

- analytics/log success with minimal payload

### 69. Activated log

**Endpoint**

```http
POST /api/v2/logs/activated
```

**Purpose**

- Submit app activation or install activation event

**Auth**

- Likely signed route, possibly without login

**To verify next**

- Required analytics fields

**Draft request contract**

Likely body areas:

- device identifiers
- app version fields
- activation/install tracking metadata

**Planned success semantics**

- `success:1`
- no meaningful response payload expected

## Bulk Verification Sweep

This section records the outcome of the broad route sweep performed after the first wave of endpoint notes.

The goal here was coverage first:

- confirm whether the route exists on the current host
- confirm whether it is public, auth-gated, or parameter-gated
- capture at least one useful response-shape clue

### Plans and payment

- `GET /api/v4/plans`
  Live verified as public signed route.
  Response shape is very close to `/api/v3/plans`, including:
  - `plan_message`
  - `unavailable_plan_message`
  - `intros`
  - `plans`
  - `platforms`
- `POST /api/v2/plans/payment_order`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: plan_id`.
- `POST /api/v3/plans/payment_order`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: plan_id`.

### Actor, director, maker, publisher, series families

- `GET /api/v1/actors/recommend`
  Live verified as public signed route.
  Observed top-level fields:
  - `new_actors`
  - `monthly_actors`
  - `recommend_actors`
  Observed actor item fields:
  - `id`
  - `type`
  - `avatar_url`
  - `name`
  - `other_name`
  - `gender`
  - `videos_count`
- `POST /api/v1/actors/{actor_id}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.
- `POST /api/v1/actors/batch_uncollection`
  Current host returned `HTTP 404`.
- `GET /api/v1/directors`
  Public signed route.
  Missing params returned `ParameterInvalid: type`.
  `type=0` returned `success:1` with:
  - `directors`
  - `current_page`
  `type=1` returned the same observed shape in the current round.
- `GET /api/v1/directors/{director_id}`
  Live verified as public signed route.
  Observed top-level fields:
  - `share_info`
  - `has_collected`
  - `director`
- `POST /api/v1/directors/{director_id}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.
- `GET /api/v1/makers`
  Public signed route.
  Missing params returned `ParameterInvalid: type`.
  `type=0` and `type=1` both returned `success:1` with:
  - `makers`
  - `current_page`
- `GET /api/v1/makers/{maker_id}`
  Live verified as public signed route.
  Observed top-level fields:
  - `share_info`
  - `has_collected`
  - `maker`
- `POST /api/v1/makers/{maker_id}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.
- `GET /api/v1/publishers/{publisher_id}`
  Requires a valid publisher id.
  Invalid sample id returned `ResourceNotFound`.
  Verified working example:
  - `GET /api/v1/publishers/9D2Qg`
  Observed top-level fields:
  - `share_info`
  - `publisher`
- `GET /api/v1/series/letters`
  Live verified as public signed route.
  Observed response field:
  - `letters`
  Observed letter item fields:
  - `id`
  - `letter`
  - `type`
  - `description`
  - `videos_count`
  - `views_count`
  Passing `type=0` or `type=1` did not change the observed response in this test round.
- `POST /api/v1/series/{series_id}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.

### Lists, codes, tags

- `GET /api/v1/lists`
  Without login returned `JWTVerificationError`.
  With login, the current sample request returned `HTTP 500`, so the route exists but still needs cautious retesting.
- `GET /api/v1/lists/simple`
  Without login returned `JWTVerificationError`.
  With login returned `success:1` with:
  - `lists`
  Observed list item fields:
  - `id`
  - `name`
  - `privacy`
  - `is_default`
  - `movies_count`
  - `has_movie`
- `GET /api/v1/lists/related`
  Missing params returned `ParameterInvalid: movie_id`.
  `movie_id=bKOeYA` returned `success:1` with:
  - `lists`
  - `current_page`
- `GET /api/v1/lists/{list_id}`
  Invalid or unauthorized sample ids can return `NoPermission`.
  Verified public example:
  - `GET /api/v1/lists/RDRz`
  Observed top-level fields:
  - `is_creator`
  - `share_info`
  - `has_collected`
  - `list`
- `POST /api/v1/lists/{list_id}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.
- `POST /api/v1/lists/{list_id}/movie_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: movie_id`.
  Supplying only `movie_id` then returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `movie_id=<movie_id>`
  - multipart form field `name=add`
  - or multipart form field `name=remove`
  Client-side "toggle" behavior is implemented before the request:
  - unselected list item -> `name=add`
  - selected list item -> `name=remove`
  No evidence was found for extra fields such as list name, movie title, `operation`, or `status`.
- `GET /api/v1/codes/{code}`
  Sample `KV-316` returned `ResourceNotFound`.
  This route likely needs a valid code resource or may not map 1:1 to all public movie numbers.
- `POST /api/v1/codes/{code}/collect_actions`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: name`.
  AOT analysis identified the real request body:
  - multipart form field `name=collect`
  - or multipart form field `name=uncollect`
  Live verification confirmed both requests succeed when sent as `multipart/form-data`.
- `GET /api/v1/following_tags`
  Current host returned `HTTP 404`.
- `GET /api/v1/following_tags/{tag_id}`
  Current host returned `HTTP 404`.
- `POST /api/v1/following_tags/{tag_id}/sort`
  Current host returned `HTTP 404`.
- `POST /api/v1/following_tags/batch_destroy`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: ids`.
- `POST /api/v1/following_tags/batch_push`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: tags`.

### Movies, rankings, related discovery

- `GET /api/v1/movies/recommend`
  Live verified as public signed route.
  Observed top-level fields:
  - `period`
  - `movies`
- `GET /api/v1/movies/recommend_periods`
  Live verified as public signed route.
  Observed top-level fields:
  - `periods`
  - `current_page`
- `GET /api/v1/movies/may_also_like`
  `movie_id=bKOeYA` with no auth returned `JWTVerificationError`.
  Current classification: authenticated route with at least one required context parameter.
- `GET /api/v1/movies/tags`
  Requires `filter_by`.
  Empty request returned `ParameterInvalid: filter_by`.
  A naive guess such as `filter_by=can_play` triggered `HTTP 500`.
  AOT call sites show this is a movie-list endpoint, not a tag-catalog endpoint.
  Live verification confirmed all of the following:
  - `filter_by=main:m&page=1&limit=48` returned `HTTP 500`
  - `filter_by=main:p&page=1&limit=48` returned `HTTP 500`
  - `filter_by=main:m&sort_by=release_date&order_by=desc&page=1&limit=48` succeeded
  - `filter_by=main:p&sort_by=release_date&order_by=desc&page=1&limit=48` succeeded
  Successful responses returned:
  - `movies`
  - `has_collected`
  - `current_page`
- `GET /api/v1/rankings`
  Missing params returned `ParameterInvalid: type`.
  `type=0` returned `ParameterInvalid: period`.
  `type=1` returned `success:1` with:
  - `periods`
  - `current_page`
  `type=1&period=552` returned `success:1` with:
  - `movies`
  `type=0&period=552` returned `JWTVerificationError`.
  Current inference:
  - some `type` values return ranking-period metadata rather than movie rows
  - `type=1` behaves as a public ranking feed once `period` is supplied
  - `type=0` appears to be an authenticated ranking branch
- `GET /api/v1/rankings/actors`
  Missing params returned `ParameterInvalid: type`.
  `type=0` returned `ParameterInvalid: filter_by`.
  `type=0&filter_by=views` returned `success:1` with:
  - `actors`
  `type=0&filter_by=movies_count` also returned `success:1` with:
  - `actors`
  `type=1&filter_by=views` also returned `success:1` with:
  - `actors`
  Current inference:
  - `filter_by` is required
  - both `type=0` and `type=1` are usable once `filter_by` is supplied
- `GET /api/v1/rankings/playbackP`
  Current host returned `HTTP 404`.

### Ads, articles, utility catalogs

- `GET /api/v1/ads`
  Live verified as public signed route.
  Observed top-level fields:
  - `enabled`
  - `ads`
  Observed placement groups include:
  - `magnets_top`
  - `web_magnets_top`
  - `index_top`
- `POST /api/v1/ads/splash_log`
  Route shape confirmed.
  Empty request returned `ParameterInvalid: id`.
- `GET /api/v1/magnet_apps`
  Live verified as public signed route.
  Observed top-level field:
  - `apps`
  Observed app item fields:
  - `name`
  - `description`
  - `recommended`
  - `links`
- `GET /api/v1/articles`
  Live verified as public signed route.
- `GET /api/v1/articles/{article_id}`
  Live verified as public signed route.

### User/account and wallet

- `POST /api/v1/users/forgot_password`
  Public route shape confirmed.
  Empty request returned `ParameterInvalid: email`.
- `POST /api/v1/users/reset_password`
  Public route shape confirmed.
  Empty request returned `ParameterInvalid: email`.
- `POST /api/v1/users/change_password`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: old_password`.
- `POST /api/v1/users/change_username`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: current_password`.
- `POST /api/v1/users/resend_activation_code`
  Public route shape confirmed.
  Empty request returned `ParameterInvalid: email`.
- `POST /api/v1/users/activate_registration`
  Public route shape confirmed.
  Empty request returned `ParameterInvalid: email`.
- `GET /api/v1/users/additional`
  Authenticated route verified.
  Observed fields:
  - `reports_count`
  - `deleted_comments_count`
  - `muted_count`
  - `max_muted_count`
  - `uncorrected_count`
  - `corrections_count`
- `GET /api/v1/users/transaction_logs`
  Authenticated route verified.
  Observed fields:
  - `notice`
  - `logs`
  - `current_page`
- `GET /api/v1/users/unpaid_tickets`
  Current host returned `HTTP 404`.
- `GET /api/v1/users/promotion_logs`
  Authenticated route verified.
  Observed fields:
  - `logs`
  - `current_page`
- `POST /api/v1/users/feedback`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: content`.
- `POST /api/v1/wallets/verify_email`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: code`.
- `POST /api/v1/wallets/send_verification_email`
  Authenticated route verified.
  Empty authenticated request returned `success:1`.
- `POST /api/v1/wallets/bind_withdraw_account`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: withdraw_type`.
- `GET /api/v1/wallets/binded_withdraw_accounts`
  Authenticated route verified.
  Observed field:
  - `accounts`
- `DELETE /api/v1/wallets/unbind_withdraw_account/{id}`
  Sample id `1` returned `ResourceNotFound`.
- `GET /api/v1/wallets/withdraw_logs`
  Authenticated route verified.
  Observed fields:
  - `logs`
  - `current_page`
- `GET /api/v1/wallets/rebate_logs`
  Authenticated route verified.
  Observed fields:
  - `logs`
  - `current_page`
- `GET /api/v1/wallets/sfpay_banks`
  Authenticated route verified.
  Observed top-level field:
  - `banks`
- `POST /api/v2/wallets/withdraw`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: withdraw_account_id`.

### Logging endpoints

- `POST /api/v1/logs/movie_played`
  Authenticated route shape confirmed.
  Empty request returned `ParameterInvalid: movie_id`.
- `POST /api/v2/logs/activated`
  Public signed route shape confirmed.
  Empty request returned `ParameterInvalid: device_uuid`.

## Pending Verification Inventory

These routes were extracted from the APK. Some are likely public, others probably require login or additional params. They have not all been re-verified here.

### Startup and config

- `/api/v1/startup`
- `/api/v1/about`
- `/api/v1/helps`
- `/api/v1/debug_logging`

### Auth and users

- `/api/v1/sessions`
- `/api/v1/users`
- `/api/v1/users/forgot_password`
- `/api/v1/users/reset_password`
- `/api/v1/users/change_password`
- `/api/v1/users/change_username`
- `/api/v1/users/resend_activation_code`
- `/api/v1/users/activate_registration`
- `/api/v1/users/additional`
- `/api/v1/users/recent_viewed`
- `/api/v1/users/transaction_logs`
- `/api/v1/users/unpaid_tickets`
- `/api/v1/users/promotion_logs`
- `/api/v1/users/feedback`

### Wallet and payments

- `/api/v1/wallets`
- `/api/v1/wallets/verify_email`
- `/api/v1/wallets/send_verification_email`
- `/api/v1/wallets/bind_withdraw_account`
- `/api/v1/wallets/binded_withdraw_accounts`
- `/api/v1/wallets/unbind_withdraw_account/%s`
- `/api/v1/wallets/withdraw_logs`
- `/api/v1/wallets/rebate_logs`
- `/api/v1/wallets/usdt_chain_types`
- `/api/v1/wallets/sfpay_banks`
- `/api/v2/wallets/withdraw`
- `/api/v3/plans`
- `/api/v4/plans`
- `/api/v2/plans/payment_order`
- `/api/v3/plans/payment_order`

### Movies and reviews

- `/api/v1/movies/top`
- `/api/v1/movies/latest`
- `/api/v1/movies/recommend`
- `/api/v1/movies/recommend_periods`
- `/api/v1/movies/may_also_like`
- `/api/v1/movies/tags`
- `/api/v4/movies/%s`
- `/api/v1/movies/%s/play`
- `/api/v1/movies/%s/resume_play`
- `/api/v1/movies/%s/magnets`
- `/api/v1/movies/%s/reviews`
- `/api/v1/movies/%s/reviews/%s`
- `/api/v1/movies/%s/reviews/%s/like`
- `/api/v1/movies/%s/reviews/%s/report`
- `/api/v1/reviews/hotly`

### Actors and related

- `/api/v1/actors`
- `/api/v1/actors/%s`
- `/api/v1/actors/recommend`
- `/api/v1/actors/%s/collect_actions`
- `/api/v1/actors/batch_uncollection`
- `/api/v1/directors`
- `/api/v1/directors/%s`
- `/api/v1/directors/%s/collect_actions`
- `/api/v1/makers`
- `/api/v1/makers/%s`
- `/api/v1/makers/%s/collect_actions`
- `/api/v1/publishers/%s`
- `/api/v1/series`
- `/api/v1/series/%s`
- `/api/v1/series/letters`
- `/api/v1/series/%s/collect_actions`

### Lists, codes, tags, search

- `/api/v1/lists`
- `/api/v1/lists/simple`
- `/api/v1/lists/%s`
- `/api/v1/lists/%s/collect_actions`
- `/api/v1/lists/%s/movie_actions`
- `/api/v1/lists/related`
- `/api/v1/codes/%s`
- `/api/v1/codes/%s/collect_actions`
- `/api/v1/following_tags`
- `/api/v1/following_tags/%s`
- `/api/v1/following_tags/%s/sort`
- `/api/v1/following_tags/batch_destroy`
- `/api/v1/following_tags/batch_push`
- `/api/v2/tags`
- `/api/v2/search`
- `/api/v2/search_image`
- `/api/v1/search_magnet`

### Other

- `/api/v1/ads`
- `/api/v1/ads/splash_log`
- `/api/v1/articles`
- `/api/v1/articles/%s`
- `/api/v1/rankings`
- `/api/v1/rankings/actors`
- `/api/v1/rankings/playbackP`
- `/api/v1/magnet_apps`
- `/api/v1/logs/movie_played`
- `/api/v2/logs/activated`

## Suggested Next Expansion

Best next targets for documentation:

1. Map which endpoints require `authorization` in addition to `jdsignature`
2. Reconstruct a proper upload client for `/api/v2/search_image`
3. Verify whether review pagination or ordering params exist
4. Test `/play` and `/resume_play` again with a real logged-in token and full query params
5. Add higher-level playback helpers to the Python script

## Documentation Backlog

Useful follow-up sections to add as verification continues:

1. A host compatibility table comparing `jdforrepam.com` and alternate candidate domains
2. Captured success and failure examples for each mutation endpoint
3. A parameter matrix for paginated feeds, ranking filters, and playback query fields
4. A dedicated multipart upload section once `/api/v2/search_image` is reproduced live
5. A user/account object reference once more authenticated endpoints are verified
