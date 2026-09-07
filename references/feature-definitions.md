# Audio feature definitions and availability

Checked against API v1.4.0 on September 7, 2026. Values can be null. Do not invent missing values, infer zero, or promise equivalence to Spotify's models.

## Sources

- `essentia`: live analysis of a short preview, typically 30 seconds. Current analysis version is `1.2`. Valence, acousticness, instrumentalness, and liveness are null. Speechiness is a computed proxy and can be present.
- `acousticbrainz_archive`: archived full-track community analyses. Valence, acousticness and instrumentalness can be available. Loudness and time signature are unavailable in the current archive mapping. Liveness is null.
- `status: "unavailable"`: no usable audio source was found. A 200 response can carry this terminal outcome without a features object.

No source supplies all feature fields. Liveness is currently null across both sources. Individual and bulk results expose source/version information when available; the batch feature response does not currently include those fields. Inspect the actual response shape.

## Feature meanings

| Field | Unit/range | Interpretation |
| --- | --- | --- |
| `bpm` | Beats/minute | Estimated tempo, not limited to 60-200; half/double-time ambiguity is possible |
| `key` | Text | Short notation such as `C`, `Am`, `F#m`, `Bb` |
| `key_confidence` | 0-1 | Confidence of key estimation |
| `energy` | 0-1 | Live analysis uses a loudness-derived proxy; do not interpret as Spotify model output |
| `danceability` | 0-1 | Normalized rhythmic measure; live analysis scales Essentia's underlying measure |
| `valence` | 0-1 or null | Archive-derived musical positivity estimate; unavailable from live analysis |
| `acousticness` | 0-1 or null | Archive classifier estimate of acoustic character; unavailable from live analysis |
| `instrumentalness` | 0-1 or null | Archive classifier estimate of instrumental character; unavailable from live analysis |
| `speechiness` | 0-1 or null | Live preview-based proxy for speech-like content; not a transcript or vocal-presence guarantee |
| `liveness` | null | Currently unavailable; do not use it to detect concert recordings |
| `loudness` | LUFS, typically negative | Live preview integrated loudness with a fallback estimate; not full-master loudness |
| `time_signature` | Integer or null | Estimated beats per bar; unavailable in the archive mapping |

`tempo_confidence` is not a field in the current public feature projection. Do not generate code requiring it.

Preview estimates depend on the excerpt available. They are unsuitable for claiming a full-track mastering or compliance measurement. Use the available values for relative comparisons and explain source limitations. See the [live data-source notes](https://melodata.voltenworks.com/docs#data-sources).
