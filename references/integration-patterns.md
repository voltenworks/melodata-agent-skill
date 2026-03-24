# MeloData Integration Patterns

## Playlist Analysis

Analyze an entire playlist by pre-warming the cache, then fetching features:

```javascript
const API_KEY = process.env.MELODATA_API_KEY;
const BASE = "https://melodata.voltenworks.com/api/v1";
const headers = { Authorization: `Bearer ${API_KEY}`, "Content-Type": "application/json" };

async function analyzePlaylist(isrcs) {
  // Step 1: Pre-warm cache (free, no quota cost)
  await fetch(`${BASE}/tracks/batch/analyze`, {
    method: "POST",
    headers,
    body: JSON.stringify({ isrcs }),
  });

  // Step 2: Wait for analysis
  await new Promise(r => setTimeout(r, 45000));

  // Step 3: Batch fetch features (billed)
  const res = await fetch(`${BASE}/tracks/batch/features`, {
    method: "POST",
    headers,
    body: JSON.stringify({ isrcs }),
  });

  const { data } = await res.json();
  return data.tracks;
}
```

## Building a Recommendation Engine

Use seed tracks and target features to find similar music:

```javascript
async function getSimilar(seedIsrcs, { energy, danceability } = {}) {
  const params = new URLSearchParams();
  seedIsrcs.forEach(i => params.append("seed", i));
  if (energy) params.set("target_energy", energy);
  if (danceability) params.set("target_danceability", danceability);
  params.set("limit", "20");

  const res = await fetch(`${BASE}/recommendations?${params}`, { headers });
  const { data } = await res.json();
  return data.recommendations; // [{isrc, title, artist, match_score, features}]
}
```

## BPM Range Filter

Find tracks within a BPM range (useful for DJ tools):

```javascript
async function filterByBpm(isrcs, minBpm, maxBpm) {
  const res = await fetch(`${BASE}/tracks/batch/features`, {
    method: "POST",
    headers,
    body: JSON.stringify({ isrcs }),
  });

  const { data } = await res.json();
  return data.tracks.filter(t =>
    t.features && t.features.bpm >= minBpm && t.features.bpm <= maxBpm
  );
}
```

## Key-Compatible Track Finder

Find tracks in compatible musical keys:

```javascript
const COMPATIBLE_KEYS = {
  "C": ["C", "Am", "F", "G"],
  "Am": ["Am", "C", "Dm", "Em"],
  "G": ["G", "Em", "C", "D"],
  // ... (Camelot wheel logic)
};

async function findKeyCompatible(seedIsrc, candidateIsrcs) {
  // Get seed track's key
  const seedRes = await fetch(`${BASE}/tracks/${seedIsrc}/features`, { headers });
  const seedKey = (await seedRes.json()).data.features.key;

  // Get candidate features
  const batchRes = await fetch(`${BASE}/tracks/batch/features`, {
    method: "POST",
    headers,
    body: JSON.stringify({ isrcs: candidateIsrcs }),
  });

  const { data } = await batchRes.json();
  const compatible = COMPATIBLE_KEYS[seedKey] || [seedKey];

  return data.tracks.filter(t =>
    t.features && compatible.includes(t.features.key)
  );
}
```

## Error Handling Best Practices

```javascript
async function robustFetch(isrc) {
  const res = await fetch(`${BASE}/tracks/${isrc}/features`, { headers });

  switch (res.status) {
    case 200: return (await res.json()).data;
    case 202: {
      // Not billed — track is being analyzed
      const wait = Number(res.headers.get("Retry-After") || 30);
      await new Promise(r => setTimeout(r, wait * 1000));
      return robustFetch(isrc);
    }
    case 401: throw new Error("Invalid API key");
    case 429: {
      // Rate limited (not billed) — respect Retry-After
      const reset = Number(res.headers.get("X-RateLimit-Reset") || 0);
      const waitMs = Math.max(0, reset * 1000 - Date.now());
      await new Promise(r => setTimeout(r, waitMs + 100));
      return robustFetch(isrc);
    }
    default: {
      const err = await res.json().catch(() => ({}));
      throw new Error(err.error?.message || `API error: ${res.status}`);
    }
  }
}
```

## Python: Full Integration Example

```python
import os
import time
import requests
from typing import Optional

API_KEY = os.environ["MELODATA_API_KEY"]
BASE = "https://melodata.voltenworks.com/api/v1"
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


class MeloDataClient:
    def __init__(self, api_key: str):
        self.session = requests.Session()
        self.session.headers["Authorization"] = f"Bearer {api_key}"

    def get_features(self, isrc: str) -> dict:
        """Get audio features, handling 202 retry automatically."""
        res = self.session.get(f"{BASE}/tracks/{isrc}/features")
        if res.status_code == 202:
            time.sleep(int(res.headers.get("Retry-After", 30)))
            return self.get_features(isrc)
        res.raise_for_status()
        return res.json()["data"]

    def get_metadata(self, isrc: str) -> dict:
        res = self.session.get(f"{BASE}/tracks/{isrc}/metadata")
        res.raise_for_status()
        return res.json()["data"]

    def search(self, query: str, limit: int = 20) -> list:
        res = self.session.get(f"{BASE}/tracks/search", params={"q": query, "limit": limit})
        res.raise_for_status()
        return res.json()["data"]["results"]

    def batch_features(self, isrcs: list[str]) -> list:
        res = self.session.post(
            f"{BASE}/tracks/batch/features",
            json={"isrcs": isrcs}
        )
        res.raise_for_status()
        return res.json()["data"]["tracks"]

    def pre_analyze(self, isrcs: list[str]) -> dict:
        """Queue ISRCs for analysis (free, not billed)."""
        res = self.session.post(
            f"{BASE}/tracks/batch/analyze",
            json={"isrcs": isrcs}
        )
        res.raise_for_status()
        return res.json()["data"]

    def recommendations(self, seeds: list[str], limit: int = 10, **targets) -> list:
        params = [("seed", s) for s in seeds] + [("limit", str(limit))]
        for k, v in targets.items():
            params.append((f"target_{k}", str(v)))
        res = self.session.get(f"{BASE}/recommendations", params=params)
        res.raise_for_status()
        return res.json()["data"]["recommendations"]
```
