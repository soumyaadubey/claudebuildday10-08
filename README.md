# TrialFinder (hackathon prototype)

Helps patients find recruiting clinical trials near them and explains each one in plain language.
Built for a demo. **TrialFinder is software, not a doctor. It does not diagnose or decide eligibility. All patient data in this demo is synthetic.**

## Run

```powershell
cd C:\Users\soumy\trialfinder
npm install
copy .env.example .env      # then paste your keys into .env
npm start
```

Open http://localhost:3000 (patient) and http://localhost:3000/coordinator.html (coordinator).

You can also set the keys in the shell instead of `.env`:

```powershell
$env:ANTHROPIC_API_KEY="sk-ant-..."; $env:ELEVENLABS_API_KEY="..."; npm start
```

Optional settings: `PORT=3001`, `FORCE_SAMPLE=1` (skip ClinicalTrials.gov and use the sample trials, handy on bad wifi), `MATCH_EFFORT=medium` (more careful matching, but slower).

## How it degrades (the demo shouldn't break)

| Missing or failing | What happens |
|---|---|
| ClinicalTrials.gov down, slow (>8s), or 0 results | Loads `data/sample_trials.json` and shows a "Showing sample data" badge |
| `ANTHROPIC_API_KEY` / Opus error | Crisis check falls back to a keyword list. Matching marks the trial "Possible fit" with "Could not analyze automatically" |
| `ELEVENLABS_API_KEY` / TTS error | Listen uses the browser's built-in voice and still shows the read-along text |
| Unknown ZIP | Searches near Midtown Manhattan and says so |

## Files

- `server.js`: Express server with the crisis check, trial search, Opus matching, translation, TTS, and the coordinator queue
- `public/index.html`: the patient app (intake, crisis screen, dashboard, listen, contact review)
- `public/coordinator.html`: the coordinator queue
- `data/nyc_zips.json`: NYC ZIP to lat/long lookup
- `data/sample_trials.json`: 12 invented sample trials, clearly labeled
