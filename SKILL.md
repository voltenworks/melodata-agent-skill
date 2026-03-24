---
name: melodata-api
description: Use the MeloData API to get audio features (BPM, key, energy, danceability, valence, etc.) and metadata for any music track by ISRC. Use this skill whenever you need to look up song audio features, build music recommendation logic, analyze playlists, detect BPM/key for DJ tools, or integrate music data into any application. Also use when the user mentions MeloData, audio features API, ISRC lookups, or needs a Spotify audio features replacement.
---

# MeloData API

Audio features and metadata for any track by ISRC. Returns BPM, key, energy, danceability, valence, acousticness, loudness, instrumentalness, speechiness, liveness, key confidence, and time signature.

**Base URL:** `https://melodata.voltenworks.com/api/v1`

**Auth:** `Authorization: Bearer melo_sk_...` (get a key at https://melodata.voltenworks.com/dashboard/keys)

---

## Quick Start

```bash
curl -H "Authorization: Bearer $MELODATA_API_KEY" \
  https://melodata.voltenworks.com/api/v1/tracks/USRC17607839/features
```

Response:
```json
{
  "data": {
    "isrc": "USRC17607839",
    "title": "Bohemian Rhapsody",
    "artist": "Queen",
    "features": {
      "bpm": 143.8,
      "key": "Bb",
      "energy": 0.72,
      "danceability": 0.39,
      "valence": 0.23,
      "acousticness": 0.28,
      "loudness": -7.2,
      "instrumentalness": 0.01,
      "speechiness": 0.05,
      "liveness": 0.24,
      "time_signature": 4
    }
  },
  "meta": { "request_id": "req_abc123", "quota": { "used": 1, "limit": 1000, "resets_at": "2026-05-01T00:00:00Z" } }
}
```

---

## Critical: Handling 202 Responses

When a track hasn't been analyzed yet, the API returns **202 Accepted** (not billed). You **must** handle this.

```javascript
async function getFeatures(isrc, apiKey) {
  const res = await fetch(
    `https://melodata.voltenworks.com/api/v1/tracks/${isrc}/features`,
    { headers: { Authorization: `Bearer ${apiKey}` } }
  );

  if (res.status === 202) {
    const wait = Number(res.headers.get("Retry-After") || 30);
    await new Promise(r => setTimeout(r, wait * 1000));
    return getFeatures(isrc, apiKey); // retry — will be a cache hit
  }

  if (!res.ok) throw new Error(`MeloData API error: ${res.status}`);
  return res.json();
}
```

**Python equivalent:**
```python
import time, requests

def get_features(isrc, api_key):
    res = requests.get(
        f"https://melodata.voltenworks.com/api/v1/tracks/{isrc}/features",
        headers={"Authorization": f"Bearer {api_key}"}
    )
    if res.status_code == 202:
        time.sleep(int(res.headers.get("Retry-After", 30)))
        return get_features(isrc, api_key)
    res.raise_for_status()
    return res.json()
```

---

## Endpoints Reference

### GET /v1/tracks/{isrc}/features
Audio features by ISRC. Returns 202 if not yet analyzed (not billed).

**Parameters:** `isrc` (path, required) — International Standard Recording Code

**Response fields:** `bpm`, `key`, `key_confidence`, `energy`, `danceability`, `valence`, `acousticness`, `loudness`, `instrumentalness`, `speechiness`, `liveness`, `time_signature`

### GET /v1/tracks/{isrc}/metadata
Track metadata: title, artist, album, release_date, duration_ms, genres.

### GET /v1/tracks/search?q={query}
Search by title or artist. Params: `q` (required, min 2 chars), `limit` (default 20, max 50), `offset`.

### GET /v1/artists/{id}
Artist data by MusicBrainz ID. Returns name, country, genres, track_count.

### GET /v1/recommendations
Similar tracks from seed ISRCs. Params: `seed` (1-5 ISRCs, repeatable), `limit` (max 50), `target_bpm`, `target_energy`, `target_danceability`, `target_valence`.

### POST /v1/tracks/batch/features
Batch lookup up to 50 ISRCs. Body: `{"isrcs": ["ISRC1", "ISRC2"]}`. Counts as N lookups.

### POST /v1/tracks/batch/analyze
Pre-queue ISRCs for analysis. **Free, no billed lookups.** Body: `{"isrcs": [...]}`. Returns `{queued, already_analyzed, already_queued, total}`.

### POST /v1/tracks/{isrc}/reanalyze
Re-analyze unavailable tracks. Pro+ plans only. 30-day cooldown.

---

## Response Envelope

**Success:**
```json
{"data": {...}, "meta": {"request_id": "req_...", "quota": {"used": N, "limit": N, "resets_at": "ISO"}}}
```

**Error:**
```json
{"error": {"message": "...", "status": 401}, "meta": {"request_id": "req_..."}}
```

## Billing Rules

| Status | Billed? | Why |
|--------|---------|-----|
| 2xx | Yes | Success |
| 202 | **No** | Track analyzing |
| 4xx (not 429) | Yes | Client error |
| 429 | **No** | Rate limited |
| 5xx | **No** | Server error |

## Rate Limits

| Plan | $/mo | Quota | Req/s | Req/min |
|------|------|-------|-------|---------|
| Free | $0 | 1K | 5 | 100 |
| Dev | $19 | 25K | 10 | 300 |
| Pro | $79 | 200K | 25 | 1K |
| Scale | $299 | 1M+SLA | 50 | 3K |

Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-Quota-Used`, `X-Quota-Limit`

---

## Common Patterns

For detailed integration patterns (batch workflows, playlist analysis, recommendation engines, error handling), read `references/integration-patterns.md`.

## Feature Definitions

For the exact meaning, range, and unit of each audio feature, read `references/feature-definitions.md`.
