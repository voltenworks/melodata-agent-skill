# MeloData Agent Skill

API integration guidance for [MeloData](https://melodata.voltenworks.com): music metadata, audio features, title-and-artist resolution, and saved bulk jobs.

Updated for API v1.4.0, September 7, 2026. The skill explains nullable features and recording-match confidence, so generated integrations can handle incomplete results.

## Install or update

Clone into a directory your coding agent loads as a skill:

```bash
git clone https://github.com/voltenworks/melodata-agent-skill.git your-project/.claude/skills/melodata-api
```

For an existing Git installation, review the incoming changes and update with:

```bash
git -C your-project/.claude/skills/melodata-api pull --ff-only
```

If you copied the files manually, replace `SKILL.md` and the entire `references/` directory together. The skill links to those references; copying only the main file leaves examples unavailable. Load the directory through your agent's supported skill/context mechanism.

## Included

- [SKILL.md](SKILL.md): endpoints, authentication, limits, billing, bulk states and retention.
- [Integration patterns](references/integration-patterns.md): bounded JavaScript/Python polling and resumable bulk submission.
- [Feature definitions](references/feature-definitions.md): units, nullable fields, preview and archive limitations.
- [Evaluation cases](evals/evals.json): prompts and expected behaviors for checking an agent using the skill. These are evaluation specifications, not recorded passing test results.

## Changes in this update

- Resolve a title and artist to an ISRC, with recording-confidence checks.
- Submit saved bulk jobs, resume polling, preserve ordered results and retry failed items.
- Explain the separate included bulk allowances on Free, Dev, Pro and Scale.
- Replace unbounded recursion and fixed-delay completion assumptions with bounded polling.
- Correct error billing, batch billing descriptions and feature availability.

No API keys are included. Read `MELODATA_API_KEY` from your environment and create a key in the [developer portal](https://melodata.voltenworks.com/sign-in) under API Keys.

## Sources and maintenance

- [Live API documentation](https://melodata.voltenworks.com/docs)

When an API release changes endpoints, billing, limits, response shapes or feature availability, update the skill, affected references and evaluation cases together. Check billing descriptions against actual implementation when older documentation disagrees. Record the API version and verification date in `SKILL.md`.
