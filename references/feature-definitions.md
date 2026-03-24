# Audio Feature Definitions

## Numeric Features (0.0 to 1.0)

| Feature | Range | Description |
|---------|-------|-------------|
| **energy** | 0.0 - 1.0 | Perceptual intensity and activity. High energy = loud, fast, noisy. Low energy = soft, slow, ambient. Derived from loudness, spectral energy, and onset rate. |
| **danceability** | 0.0 - 1.0 | How suitable a track is for dancing. Based on tempo stability, rhythm strength, beat regularity, and overall regularity. |
| **valence** | 0.0 - 1.0 | Musical positivity. High valence = happy, cheerful, euphoric. Low valence = sad, depressed, angry. Derived from spectral and harmonic content. |
| **acousticness** | 0.0 - 1.0 | Confidence that the track is acoustic (non-electronic). 1.0 = highly acoustic, 0.0 = highly electronic. |
| **instrumentalness** | 0.0 - 1.0 | Predicts whether a track contains vocals. Values above 0.5 suggest instrumental. "Ooh" and "aah" sounds are treated as instrumental. |
| **speechiness** | 0.0 - 1.0 | Presence of spoken words. High values = spoken word, podcast, rap. Mid values = music with lyrics. Low values = instrumental. |
| **liveness** | 0.0 - 1.0 | Probability of a live audience. Values above 0.8 strongly suggest live recording. Detects audience noise, reverb patterns. |
| **key_confidence** | 0.0 - 1.0 | Confidence of the key detection algorithm. |
| **tempo_confidence** | 0.0 - 1.0 | Confidence of the BPM detection algorithm. |

## Non-Normalized Features

| Feature | Unit | Description |
|---------|------|-------------|
| **bpm** | beats/min | Tempo in beats per minute. Typical range: 60-200. Ballads ~70, pop ~120, EDM ~128, drum & bass ~170. |
| **key** | text | Musical key in short notation. Examples: `C`, `Am`, `F#m`, `Bb`. Major keys have no suffix, minor keys end in `m`. |
| **loudness** | dB | Overall loudness. Typical range: -60 to 0 dB. Values closer to 0 = louder. |
| **time_signature** | integer | Estimated time signature (beats per bar). Common values: 3 (waltz), 4 (most music), 5, 6, 7. |

## Feature Sources

- **AcousticBrainz archive** (`source: "acousticbrainz_archive"`): Features extracted by the AcousticBrainz project using Essentia. `liveness` is `null` for these tracks.
- **On-demand Essentia** (`source: "essentia"`): Analyzed from 30-second iTunes/Deezer previews. `valence`, `acousticness`, `instrumentalness`, `speechiness` may be `null` (requires ML models not in base Essentia).
- **Unavailable** (`analysis_version: "unavailable"`): No audio source found. Track may be unreleased, region-locked, or have an incorrect ISRC.
