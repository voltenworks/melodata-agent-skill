# MeloData Agent Skill

Agent skill for the [MeloData API](https://melodata.voltenworks.com) — audio features and metadata for any music track by ISRC.

## What This Is

A skill file that gives AI coding agents (Claude Code, Cursor, Windsurf, etc.) full context about the MeloData API: endpoints, authentication, 202 retry handling, billing rules, rate limits, and integration patterns.

## Quick Install

### Claude Code

```bash
# Add to your project
cp -r melodata-api/ your-project/.claude/skills/
```

Or reference it in your CLAUDE.md:
```
See .claude/skills/melodata-api/SKILL.md for MeloData API reference.
```

### Cursor / Windsurf

Copy `SKILL.md` into your project's AI rules or context directory.

## What's Included

```
SKILL.md                              # Core API reference (all endpoints, auth, billing)
references/
  integration-patterns.md             # Playlist analysis, recommendations, error handling, Python client
  feature-definitions.md              # What each audio feature means, ranges, units
evals/
  evals.json                          # Test cases for skill validation
```

## API Overview

MeloData returns 12 audio features for any track by ISRC:

| Feature | Range | Description |
|---------|-------|-------------|
| bpm | 60-200 | Tempo in beats per minute |
| key | text | Musical key (e.g. "Bb", "C#m") |
| energy | 0-1 | Perceptual intensity |
| danceability | 0-1 | Rhythm suitability for dancing |
| valence | 0-1 | Musical positivity/happiness |
| acousticness | 0-1 | Acoustic vs electronic |
| loudness | dB | Overall loudness |
| instrumentalness | 0-1 | Vocal presence |
| speechiness | 0-1 | Spoken word presence |
| liveness | 0-1 | Live audience probability |

**Base URL:** `https://melodata.voltenworks.com/api/v1`

**Get a free API key:** https://melodata.voltenworks.com/dashboard/keys

## Links

- [API Documentation](https://melodata.voltenworks.com/docs)
- [Dashboard](https://melodata.voltenworks.com/dashboard)
- [Pricing](https://melodata.voltenworks.com/#pricing)
