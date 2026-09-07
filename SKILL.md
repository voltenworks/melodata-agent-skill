---
name: melodata-api
description: Integrate the MeloData music API for ISRC audio-feature lookups, title-and-artist resolution, saved bulk enrichment, playlist analysis, BPM/key tools, and recommendations. Use when the user mentions MeloData or needs music metadata or an alternative to Spotify audio features. Explain nullable fields and source differences instead of promising Spotify-identical results.
---

# MeloData API

Contract checked against API v1.4.0 on September 7, 2026.

**Base URL:** `https://melodata.voltenworks.com/api/v1`

**Authentication:** `Authorization: Bearer $MELODATA_API_KEY`. Read the key from the environment. Never print it, commit it, or include it in URLs. Create keys in the [developer portal](https://melodata.voltenworks.com/sign-in) under API Keys.

## Choose the workflow

- Have one ISRC: request `/tracks/{isrc}/features`.
- Have one title and artist: resolve `/tracks/resolve` first, then inspect confidence before requesting features.
- Have a list of titles and artists: submit a saved bulk job. It keeps progress when the caller disconnects and returns ordered per-track outcomes.
- Have ISRCs already: `/tracks/batch/features` reads up to 50 per request; `/tracks/batch/analyze` queues up to 50 for analysis.

Do not promise every feature for every song. Every numeric feature can be null. Live analysis uses short previews, and its values are not identical to Spotify's features. Read [feature definitions](references/feature-definitions.md) before interpreting results.

## Single-track lookup

```bash
curl --fail-with-body \
  -H "Authorization: Bearer $MELODATA_API_KEY" \
  https://melodata.voltenworks.com/api/v1/tracks/USUG11904206/features
```

A fully analyzed lookup includes `isrc`, `title`, `artist`, `source`, `analysis_version`, and `features`. A metadata-only **200** returns `status: "partial"` with nullable features and can omit `source` and `analysis_version`; treat those fields as optional and partial as a terminal result. Feature keys are `bpm`, `key`, `key_confidence`, `energy`, `danceability`, `valence`, `acousticness`, `loudness`, `instrumentalness`, `speechiness`, `liveness`, and `time_signature`.

A **202** means analysis is pending, not that features exist. Respect `Retry-After` (default 30 seconds), bound the number of polls, and use request timeouts. A later **200** can report `status: "unavailable"` with no features. Never recurse indefinitely or assume the next request will be ready. Runnable bounded examples are in [integration patterns](references/integration-patterns.md).

## Resolve a title and artist

`GET /tracks/resolve?title={title}&artist={artist}`

Both strings must contain 2-200 characters. Encode query parameters with `URLSearchParams` or the HTTP client's parameter option. Preserve live, remix, acoustic, and other recording qualifiers in the title.

Response data contains `query`, `isrc`, `matched`, `confidence`, `confidence_level`, `warning`, `source`, `cached`, `providers`, and `analysis_queued: false`. Resolving alone never queues analysis.

- `exact` or `high`: suitable for automated follow-up, subject to your application's requirements.
- `medium` or `low`: show the warning and ask the user to verify the recording. Do not silently substitute it.
- **404**: sources answered but no acceptable match was found.
- **503**: a dependency could not answer. Respect `Retry-After`; do not classify it as a missing song.

## Saved bulk jobs

Requires a direct MeloData key. RapidAPI requests receive **403**.

| Method | Path | Result |
| --- | --- | --- |
| POST | `/tracks/jobs` | Create or replay a submission; **202**, with `id`, `replayed`, `status_url` |
| GET | `/tracks/jobs` | Latest 50 unexpired jobs and monthly `allowance` |
| GET | `/tracks/jobs/{id}` | Ordered tracks, progress, counts and expiry |
| POST | `/tracks/jobs/{id}/retry` | Retry failed items after processing finishes |

```bash
curl --fail-with-body -X POST \
  https://melodata.voltenworks.com/api/v1/tracks/jobs \
  -H "Authorization: Bearer $MELODATA_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: library-import-2026-09-07-001" \
  -d '{"tracks":[{"title":"Blinding Lights","artist":"The Weeknd"}]}'
```

Persist the input list and submission key before sending. Reuse that key with identical inputs after a timeout or network error. Keys accept 8-128 letters, numbers, periods, colons, underscores or hyphens. Changed inputs with the same key return **409**. A genuinely new list needs a new key.

`status_url` already includes `/api/v1`; resolve it against `https://melodata.voltenworks.com`, not the API base with `/api/v1` appended again. Save the job ID, and poll about every 30 seconds. Work continues when polling stops.

Job detail contains `status` (`processing` or `finished`), `total`, `remaining`, `counts`, `retries_used`, `max_retries`, `created_at`, `expires_at`, and `tracks`. Each track has `position`, `query`, `status`, `isrc`, `matched`, `result`, `error`, `attempts`, and `next_attempt_at`. The input order and duplicate rows are preserved. Features, when present, are under `track.result.features`.

Active item states: `pending`, `retrying`, `analyzing`.
Terminal item states: `completed`, `partial`, `unavailable`, `not_found`, `needs_review`, `failed`.
A finished job means every row has an outcome, not that every row succeeded. Weak matches become `needs_review` and are not analyzed automatically. Partial results contain what is available; inspect fields individually.

For `/retry`, send an `Idempotency-Key` for the retry cycle and reuse it on a network retry. Only `failed` items reset; completed items remain unchanged. At most two cycles are allowed. Retrying while processing returns **409**. When nothing failed, the endpoint returns `retried: 0`. Do not keep retrying `needs_review`, `not_found`, or `unavailable` as though they were transient failures.

Each processing cycle has a 24-hour deadline. Inputs and results expire seven days after submission; retrieve and save them before `expires_at`. Expired jobs return **410**. Retries do not extend expiry. Minimal job metadata and submission keys remain for 90 days; monthly counters remain until account deletion. Account deletion removes bulk records.

### Separate included bulk allowance

| Plan | Rows/month | Rows/job | Active jobs |
| --- | ---: | ---: | ---: |
| Free | 25 | 10 | 1 |
| Dev | 1,000 | 250 | 3 |
| Pro | 10,000 | 250 | 3 |
| Scale | 50,000 | 250 | 3 |

Each accepted input row uses one bulk unit, including duplicates and unmatched rows. Submission replays, reads, and failed-item retries add no units. Bulk does not use lookup quota and has no overage charges. `GET /tracks/jobs` returns allowance fields `submitted`, `limit`, `per_job`, `active_jobs` (the plan's active-job limit), and `resets_at`.

Oversized jobs return **400**. Active-job or monthly allowance exhaustion returns **429**. Wait for an active job to finish, or for the monthly reset as appropriate; do not use an immediate retry loop. Existing results remain readable when allowance is exhausted.

## Other endpoints

Paths below are relative to the base URL.

| Method | Path | Inputs and behavior |
| --- | --- | --- |
| GET | `/tracks/{isrc}/metadata` | Title, artist, album, release date, duration, genres |
| GET | `/tracks/search` | `q`: 2-200 characters, no `%` or `_`; `limit`: 1-50, default 20; `offset`: 0-10000 |
| GET | `/artists/{id}` | MusicBrainz artist ID; name, country, genres, track count |
| GET | `/recommendations` | Repeat `seed` 1-5 times with ISRCs; `limit`: 1-50, default 10; optional `target_bpm`, `target_energy`, `target_danceability`, `target_valence` |
| POST | `/tracks/batch/features` | Body `{"isrcs":[...]}`, 1-50 ISRCs; inspect each row for features, `not_found`, or `unavailable` |
| POST | `/tracks/batch/analyze` | Body `{"isrcs":[...]}`, 1-50 ISRCs; returns queued/already-analyzed/already-queued counts, not finished features |
| POST | `/tracks/{isrc}/reanalyze` | Pro/Scale only, unavailable tracks, 30-day cooldown |

ISRCs are 12 characters: two letters, three alphanumeric characters, then seven digits. They are case-insensitive. Split longer ISRC lists into batches of at most 50 and respect account rate limits. A fixed sleep does not prove analysis completed.

## Errors, billing and limits

Responses have a `data` or `error` envelope, plus `meta.request_id`. Preserve the request ID for debugging; do not log authorization headers. Errors include `error.message` and `error.status`. Quota metadata is endpoint-dependent; bulk reports its own allowance.

For direct-key, non-bulk requests, the current implementation bills one lookup per **200/201 request**. A successful `/tracks/batch/features` request counts once, not once per ISRC. `/tracks/batch/analyze` also currently returns a billed 200; do not describe it as free. Some older API documentation describes different batch billing. Check the current account usage and published contract before making cost promises. This skill describes implemented behavior and does not change billing.

**202, 4xx and 5xx responses do not consume lookup quota.** Bulk submission's 202 uses its separate row allowance as described above.

| Plan | USD/month | Lookup quota/month | Requests/second | Requests/minute |
| --- | ---: | ---: | ---: | ---: |
| Free | 0 | 1,000 | 5 | 100 |
| Dev | 19 | 25,000 | 10 | 300 |
| Pro | 79 | 200,000 | 25 | 1,000 |
| Scale | 299 | 1,000,000 | 50 | 3,000 |

Rate-limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Unix seconds). Non-bulk responses can include `X-Quota-Used` and `X-Quota-Limit`. A 429 can indicate rate limiting, monthly quota exhaustion, or a bulk concurrency limit. Inspect its message and headers; monthly exhaustion is not fixed by rapid retries. Lookup overage is opt-in at $0.002 per lookup. Never enable billing options automatically.

For new integrations, recheck the [live API docs](https://melodata.voltenworks.com/docs). Treat fields that are absent or null as unknown, never as zero.
