# TrialFinder

**Find clinical trials near you, explained in plain language, in your own language.**

TrialFinder helps adults with depression and anxiety find recruiting clinical trials near them. It pulls live studies from ClinicalTrials.gov, compares each one to what the patient tells us, rewrites the dense listing at a 6th-grade reading level, and reads it aloud in seven languages. Nothing reaches a study team until the patient says so, and a human coordinator reviews every request.

**Live demo: https://trialfinder-ebon.vercel.app/**

Built in 2 hours at the Claude Build Day hackathon (2026-10-08).

> **TrialFinder is software, not a doctor.** It does not diagnose, recommend treatment, or decide eligibility. Only a study team can do that. **All patient data in this demo is synthetic.** This is a prototype, not a medical product, and is not meant for real patients (see [Limitations](#limitations)).

## Why

Trial listings are written for researchers. Eligibility criteria are long, jargon-heavy, and English-only, so the people who might benefit most often can't tell whether a study is worth a phone call. TrialFinder answers three questions a patient actually has: *Is this for someone like me? What would taking part involve? What should I ask?*

## What it does

1. **Safety first.** Before any search, whatever the patient wrote is checked for signs of crisis. If it looks like one, the search stops and the screen shows the 988 Suicide & Crisis Lifeline (call or text). No trials are returned.
2. **Live search.** Recruiting studies within 5, 10, 25 or 50 miles of the patient's NYC ZIP code, from the ClinicalTrials.gov v2 API. The nearest recruiting site and its distance are shown for each, and remote or virtual studies are flagged.
3. **Fit analysis.** Claude compares the patient's profile with each trial and returns a fit of *likely*, *possible* or *unlikely*, with reasons for and against, what the listing doesn't say, a plain-language summary, and three questions to ask the study team. The prompt forbids saying a patient is or isn't eligible. Output uses "may" and "seems", and gaps go under *unknowns* instead of being guessed.
4. **Listen.** Any summary can be translated and read aloud in English, Spanish, Mandarin, Russian, Korean, Arabic or Hindi, with a read-along transcript. Every translation starts with a "not medical advice" disclaimer.
5. **Human in the loop.** The patient can ask a study team to get in touch, but only after ticking a consent box. The request goes to a **coordinator queue** (`/coordinator.html`), where a person approves or declines it. The patient sees the status update on their side.

## Quick start

To try it without installing anything, use the [live demo](https://trialfinder-ebon.vercel.app/). To run it locally you need Node.js 18 or newer.

```bash
git clone https://github.com/soumyaadubey/claudebuildday10-08.git trialfinder
cd trialfinder
npm install
cp .env.example .env     # PowerShell: copy .env.example .env
# add your keys to .env
npm start
```

Then open:

| URL | What |
|---|---|
| http://localhost:3000 | Patient app |
| http://localhost:3000/coordinator.html | Coordinator queue |
| http://localhost:3000/deck.html | Presentation deck (also exported as `TrialFinder-slides.pdf`) |

You can pass keys through the shell instead of `.env`:

```powershell
$env:ANTHROPIC_API_KEY="sk-ant-..."; $env:ELEVENLABS_API_KEY="..."; npm start
```

```bash
ANTHROPIC_API_KEY=sk-ant-... ELEVENLABS_API_KEY=... npm start
```

Neither key is required to run the app. Without them it degrades gracefully (see below), but you need `ANTHROPIC_API_KEY` for the real fit analysis and translation.

To check what's running, open `/api/health`. It reports which keys are set and which models are active.

### Trying the demo

- **Sample patients:** the intake screen has a menu of synthetic personas (for example a 42-year-old in Queens with depression who speaks Spanish, or a 29-year-old in Manhattan with anxiety who is pregnant). Pick one to fill the form, then open a trial and press **Listen**.
- **Your own search:** pick a condition such as "Depression" or "Generalized anxiety disorder" and enter an NYC ZIP such as `10001`.
- **Safety stop:** one sample is a synthetic sentence that triggers the 988 screen.
- **Bad wifi:** start with `FORCE_SAMPLE=1` to skip ClinicalTrials.gov entirely.

## Configuration

Set these in `.env` or the environment. All are optional except where noted.

| Variable | Default | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | none | Crisis check, fit analysis, translation |
| `ELEVENLABS_API_KEY` | none | Spoken audio for Listen |
| `PORT` | `3000` | Server port |
| `FORCE_SAMPLE` | off | Set to `1` to always use `data/sample_trials.json` |
| `MATCH_EFFORT` | `low` | Claude effort for matching: `low`, `medium` or `high`. Higher is more careful but slower |
| `CRISIS_MODEL` | `claude-sonnet-5-5` | Safety classifier |
| `MATCH_MODEL` | `claude-sonnet-5-5` | Eligibility and fit analysis |
| `TRANSLATE_MODEL` | `claude-haiku-5-5` | Translation for Listen |

**Model choices:** Sonnet where judgment and safety matter (crisis check, matching), and Haiku where the task is fast and mechanical (translation).

## When things fail

The demo is built so one broken dependency doesn't break the whole flow.

| Missing or failing | What happens |
|---|---|
| ClinicalTrials.gov down, slow (over 8s), or no results | Falls back to `data/sample_trials.json` and shows a "Showing sample data" badge |
| `ANTHROPIC_API_KEY` missing or Claude errors | The crisis check falls back to a keyword list. Matching marks the trial "Possible fit" with "Could not analyze automatically". Translation falls back to English |
| `ELEVENLABS_API_KEY` missing or TTS errors | Listen uses the browser's built-in voice and still shows the read-along text |
| ZIP not in the lookup table | Searches near Midtown Manhattan and says so |
| Crisis classifier output unparseable | Falls back to the keyword check (it errs toward showing the crisis screen) |

## How it works

```
Patient browser ──► Express server (server.js) ──► Anthropic API      crisis check, matching, translation
                     │                         ├─► ClinicalTrials.gov  live recruiting studies
                     │                         └─► ElevenLabs          text-to-speech
                     └─► in-memory coordinator queue ◄── Coordinator browser
```

All external calls happen on the server, so API keys never reach the browser.

| Endpoint | Purpose |
|---|---|
| `POST /api/search` | Crisis check, then trial search. Never returns trials on crisis |
| `POST /api/match` | Fit analysis for one trial (the browser runs up to 5 in parallel, and the server caps at 5) |
| `POST /api/speak` | Translate, then synthesize audio |
| `POST /api/contact` | Consent-gated request to the coordinator queue |
| `GET /api/requests` | Coordinator: list requests |
| `POST /api/requests/:id/decision` | Coordinator: `approve` or `decline` |
| `GET /api/requests/status` | Patient: poll request status |
| `GET /api/health` | Keys present, models in use |

Details worth knowing:

- Patient free text and trial listings are passed to the models as data, and the prompts tell the models to ignore any instructions inside them.
- Match results, translations and audio are cached in memory. Duplicate in-flight match requests are merged, so a double click doesn't pay twice.
- A trial that isn't in memory (for example on a fresh serverless instance) is re-fetched by NCT ID instead of failing.

## Project layout

```
server.js                 Express server: crisis check, search, matching, translation, TTS, coordinator queue
public/index.html         Patient app (landing, step-by-step intake, crisis screen, dashboard, Listen, contact review)
public/coordinator.html   Coordinator queue
public/deck.html          Presentation deck
data/nyc_zips.json        NYC ZIP → lat/long lookup
data/sample_trials.json   12 invented sample trials, all labeled as samples
TrialFinder-slides.pdf    Exported slides
```

## Limitations

This was a 2-hour build. Before it could go near real patients it would need:

- **Persistence and privacy.** State (requests, caches) is in memory and resets on restart. Patient details and free text are sent to Anthropic, and translated text to ElevenLabs. There is no encryption at rest, retention policy, or HIPAA review.
- **Authentication.** The coordinator page and `/api/requests*` endpoints are open. Anyone who can reach the server can read the queue and approve or decline requests.
- **Coverage.** ZIP lookup covers NYC only, and the patient flow targets depression and anxiety. Other ZIPs fall back to Midtown Manhattan.
- **Evaluation.** The fit analysis and crisis classifier have not been clinically validated or tested against a labeled set. Language models can be wrong, and a keyword fallback is a weak safety net. Translations haven't been reviewed by native speakers.
- **Accessibility and testing.** There is no automated test suite, and the UI hasn't had a full accessibility audit.

## Data sources and credits

- Trial data: [ClinicalTrials.gov API v2](https://clinicaltrials.gov/data-api/api), U.S. National Library of Medicine
- Language models: [Claude](https://www.anthropic.com/claude) (Sonnet 5.5 and Haiku 5.5)
- Voice: [ElevenLabs](https://elevenlabs.io)
- Crisis support shown in the app: [988 Suicide & Crisis Lifeline](https://988lifeline.org)

## License

[MIT](LICENSE) © 2026 Soumya Dubey
