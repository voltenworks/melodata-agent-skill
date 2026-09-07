# Integration patterns

Examples use Node.js 20+ with built-in `fetch`, or Python with `requests`. Set `MELODATA_API_KEY` in the environment. These helpers do not log it.

## JavaScript: bounded reads and saved bulk jobs

Keep these helpers together. The 30-second request timeout allows the resolver its 25-second server budget. Read errors preserve the response for the caller to distinguish rate limits from quota exhaustion. A timeout stops this caller's polling; saved bulk work continues on the server.

```javascript
const ORIGIN = "https://melodata.voltenworks.com";
const BASE = `${ORIGIN}/api/v1`;
const API_KEY = process.env.MELODATA_API_KEY;
if (!API_KEY) throw new Error("Set MELODATA_API_KEY");

const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

async function request(path, { method = "GET", body, key } = {}) {
  const response = await fetch(`${BASE}${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      ...(body ? { "Content-Type": "application/json" } : {}),
      ...(key ? { "Idempotency-Key": key } : {}),
    },
    ...(body ? { body: JSON.stringify(body) } : {}),
    signal: AbortSignal.timeout(30000),
  });
  const payload = await response.json();
  return { response, payload };
}

function failure(response, payload) {
  const error = new Error(payload.error?.message || `HTTP ${response.status}`);
  error.status = response.status;
  error.requestId = payload.meta?.request_id;
  return error;
}

function retryDelay(response) {
  const value = response.headers.get("Retry-After");
  const seconds = value === null ? 30 : Number(value);
  // A long cooldown needs a scheduled resume, not an early retry.
  if (Number.isFinite(seconds) && seconds > 60) {
    throw new Error(`Retry later, after ${seconds} seconds`);
  }
  return Number.isFinite(seconds) && seconds > 0 ? seconds * 1000 : 30000;
}

async function poll(path, pending, maxPolls = 20) {
  if (!Number.isInteger(maxPolls) || maxPolls < 1 || maxPolls > 120) {
    throw new Error("maxPolls must be an integer from 1 to 120");
  }
  for (let attempt = 0; attempt < maxPolls; attempt++) {
    const { response, payload } = await request(path);
    if (response.status === 503 || response.status === 202) {
      if (attempt + 1 < maxPolls) await sleep(retryDelay(response));
      continue;
    }
    // Includes 429: the caller must inspect rate, quota or active-job limits.
    if (!response.ok) throw failure(response, payload);
    if (!pending(payload.data)) return payload.data;
    if (attempt + 1 < maxPolls) await sleep(30000);
  }
  throw new Error("Polling limit reached; resume later using the saved identifier");
}

async function getFeatures(isrc, maxPolls = 20) {
  const data = await poll(`/tracks/${encodeURIComponent(isrc)}/features`, () => false, maxPolls);
  // A terminal unavailable result is returned intact, without invented features.
  return data;
}

async function resolveTrack(title, artist) {
  const query = new URLSearchParams({ title, artist });
  const data = await poll(`/tracks/resolve?${query}`, () => false);
  return { ...data, requiresReview: !["exact", "high"].includes(data.confidence_level) };
}

async function submitBulk(tracks, submissionKey) {
  if (!submissionKey) throw new Error("Provide the persisted submission key");
  const { response, payload } = await request("/tracks/jobs", {
    method: "POST", body: { tracks }, key: submissionKey,
  });
  if (response.status !== 202) throw failure(response, payload);
  return payload.data; // Persist id and status_url; a 202 is not completed work.
}

async function waitForBulk(jobId, maxPolls = 20) {
  return poll(`/tracks/jobs/${encodeURIComponent(jobId)}`, data => data.status !== "finished", maxPolls);
}

async function retryFailed(jobId, retryKey) {
  if (!retryKey) throw new Error("Provide the persisted retry-cycle key");
  const { response, payload } = await request(`/tracks/jobs/${encodeURIComponent(jobId)}/retry`, {
    method: "POST", key: retryKey,
  });
  if (!response.ok) throw failure(response, payload);
  return payload.data;
}

async function batchFeatures(isrcs) {
  if (!Array.isArray(isrcs) || isrcs.length < 1 || isrcs.length > 50) {
    throw new Error("Provide 1-50 ISRCs per batch");
  }
  const { response, payload } = await request("/tracks/batch/features", {
    method: "POST", body: { isrcs },
  });
  if (!response.ok) throw failure(response, payload);
  return payload.data.tracks;
}

async function filterByBpm(isrcs, minBpm, maxBpm) {
  const tracks = await batchFeatures(isrcs);
  return tracks.filter(track => Number.isFinite(track.features?.bpm)
    && track.features.bpm >= minBpm && track.features.bpm <= maxBpm);
}

async function getSimilar(seedIsrcs, { energy, danceability } = {}) {
  const params = new URLSearchParams();
  seedIsrcs.forEach(isrc => params.append("seed", isrc));
  if (energy !== undefined) params.set("target_energy", String(energy));
  if (danceability !== undefined) params.set("target_danceability", String(danceability));
  params.set("limit", "20");
  const data = await poll(`/recommendations?${params}`, () => false);
  return data.recommendations;
}
```

For a bulk import:

1. Save the ordered titles/artists and a unique submission key in your application's durable storage.
2. Call `submitBulk(tracks, savedKey)`. On an uncertain network result, call it again with the same saved inputs/key. Do not generate a fresh key for each attempt.
3. Persist the returned job ID. Call `waitForBulk(jobId)`, or resume with that ID in a later process.
4. Save every row in order. `completed` and `partial` rows may contain `result.features`; inspect nullable fields. Route `needs_review` to a user decision. Preserve unavailable/missing/failed outcomes.
5. If the job has `failed` items and a retry cycle remains, persist a new retry-cycle key, call `retryFailed`, and resume polling. Reuse that retry key after an uncertain response. Do not resubmit the entire list.

Split title lists according to the allowance response's `per_job`, and respect its `active_jobs` limit. Do not fire all chunks concurrently. Check the monthly remaining allowance before submitting. Duplicate rows count individually even when their work is cached.

## Python: bounded feature and bulk polling

```python
import os
import time
import requests

BASE = "https://melodata.voltenworks.com/api/v1"


class MeloDataClient:
    def __init__(self, api_key):
        self.session = requests.Session()
        self.session.headers["Authorization"] = f"Bearer {api_key}"

    def close(self):
        self.session.close()

    def _poll(self, path, pending, max_polls=20):
        if not isinstance(max_polls, int) or not 1 <= max_polls <= 120:
            raise ValueError("max_polls must be an integer from 1 to 120")
        for attempt in range(max_polls):
            response = self.session.get(f"{BASE}{path}", timeout=30)
            if response.status_code in (202, 503):
                try:
                    delay = float(response.headers.get("Retry-After", "30"))
                except ValueError:
                    delay = 30
                if not 0 < delay <= 60:
                    raise RuntimeError("Retry later; response has an unsupported cooldown")
            else:
                # Includes 429: inspect the message before deciding when to resume.
                response.raise_for_status()
                data = response.json()["data"]
                if not pending(data):
                    return data
                delay = 30
            if attempt + 1 < max_polls:
                time.sleep(delay)
        raise TimeoutError("Polling limit reached; resume with the saved identifier")

    def get_features(self, isrc, max_polls=20):
        return self._poll(f"/tracks/{requests.utils.quote(isrc, safe='')}/features",
                          lambda data: False, max_polls)

    def submit_bulk(self, tracks, submission_key):
        if not submission_key:
            raise ValueError("Provide the persisted submission key")
        response = self.session.post(
            f"{BASE}/tracks/jobs", json={"tracks": tracks},
            headers={"Idempotency-Key": submission_key}, timeout=30,
        )
        response.raise_for_status()
        if response.status_code != 202:
            raise RuntimeError("Expected a bulk submission acknowledgment")
        return response.json()["data"]

    def wait_for_bulk(self, job_id, max_polls=20):
        return self._poll(f"/tracks/jobs/{requests.utils.quote(job_id, safe='')}",
                          lambda data: data["status"] != "finished", max_polls)


# Construction only. Submitting an import is an explicit application action.
# client = MeloDataClient(os.environ["MELODATA_API_KEY"])
# Close the session in a finally block when finished.
```

A 200 feature response with `status: "unavailable"` is a terminal result. Check for `features` before reading BPM or key. Bounded polling does not guarantee completion within that window. Do not repeat a POST with a new key just because a caller timed out.

For ISRC pre-analysis, a queued count only confirms admission. Inspect later feature responses instead of sleeping a fixed 45 seconds and assuming completion. Both successful batch endpoints currently use ordinary direct-key request billing; bulk uses its separate included row allowance.
