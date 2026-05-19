# myfuelcoach
Fuel Coach - reads what is planned on Intervals.icu and then recommends fueling 

# Daily Workout Fueling Email

An n8n workflow that reads your planned workout from intervals.icu each morning, computes pre- and post-workout macro targets, generates meal suggestions via Claude, and emails the result to you.

Companion file: [`intervals-fueling-workflow.json`](intervals-fueling-workflow.json)

## What it does

Every day at 5:00 AM Eastern:

1. Pulls today's planned workout from intervals.icu (name, type, duration, TSS, kJ, description)
2. Pulls the last 7 days of events + wellness data for weekly context (TSS load, sleep, HRV, resting HR, readiness)
3. Computes intensity bucket (easy / moderate / hard) and macro targets:
   - Pre-workout carbs scaled to body weight and intensity (0.6–1.5 g/kg)
   - During-workout fueling for sessions ≥90 min (30–90 g carbs/hr)
   - Post-workout 1.0 g/kg carbs + 0.3 g/kg protein within 60 minutes
   - Hydration and sodium targets
4. Calls Claude Haiku 4.5 with the workout context and targets, returns 2 pre-workout + 2 post-workout meal ideas
5. Renders an HTML email with workout summary, weekly context, macro table, and meal cards
6. Sends via Gmail to steven@salesignition.com

Rest days are handled — sends a recovery-focused email instead of skipping.

## Architecture

```
Schedule (5am ET cron)
  -> Init                  (profile + date ranges, JS Code)
  -> TodayEvents           (HTTP GET intervals.icu events for today)
  -> WeeklyEvents          (HTTP GET intervals.icu events, last 7 days)
  -> WeeklyWellness        (HTTP GET intervals.icu wellness, last 7 days)
  -> Macros                (compute intensity + macro targets, JS Code)
  -> Meals                 (HTTP POST Anthropic Messages API)
  -> RenderEmail           (parse LLM JSON + render HTML, JS Code)
  -> SendEmail             (Gmail send)
```

All deterministic logic (macro math, intensity bucketing, weekly aggregates, HTML render) lives in Code nodes so it's predictable and editable. The LLM is only responsible for the creative part — picking meals that hit the targets.

## Prerequisites

- Self-hosted n8n (tested on 2.16.1)
- intervals.icu account + API key (Settings → Developer)
- Anthropic API key (console.anthropic.com)
- Gmail account connected to n8n via OAuth2

## Setup

### 1. Create credentials in n8n

| Credential type     | Display name (your choice) | Required fields                                              |
| ------------------- | -------------------------- | ------------------------------------------------------------ |
| **HTTP Basic Auth** | `intervals.icu`            | **User**: `API_KEY` (literal string)  ·  **Password**: your intervals.icu API key |
| **HTTP Header Auth**| `Anthropic`                | **Name**: `x-api-key`  ·  **Value**: your Anthropic API key  |
| **Gmail OAuth2**    | `Gmail`                    | OAuth flow to steven@salesignition.com                       |

Notes on credentials:
- For intervals.icu Basic Auth, the username is the **literal string `API_KEY`**, not your email or athlete ID. This is intervals.icu's specific convention.
- For Anthropic Header Auth, the field labeled **Name** is the HTTP header name (`x-api-key`), not the credential's display name.

### 2. Import the workflow

n8n → Workflows → "..." menu → **Import from File** → pick `intervals-fueling-workflow.json`.

### 3. Attach credentials to each node

Credential IDs don't survive import (they're instance-specific), so n8n won't auto-link them. Open each of these nodes and select the right credential from the dropdown:

| Node            | Credential type    | Pick                |
| --------------- | ------------------ | ------------------- |
| `TodayEvents`   | HTTP Basic Auth    | `intervals.icu`     |
| `WeeklyEvents`  | HTTP Basic Auth    | `intervals.icu`     |
| `WeeklyWellness`| HTTP Basic Auth    | `intervals.icu`     |
| `Meals`         | HTTP Header Auth   | `Anthropic`         |
| `SendEmail`     | Gmail OAuth2       | `Gmail`             |

### 4. Update your profile

Open the `Init` node. Edit the values at the top of the JS:

```js
const profile = {
  email: 'steven@salesignition.com',
  bodyWeightKg: 81,                            // update as needed
  sex: 'M',
  preferences: ['high-protein', 'mediterranean'],
  dislikes: [],                                // e.g. ['cilantro', 'olives']
  vegetarian: false,
  allergies: [],
  cuisinePreference: 'flexible',
  timezone: 'America/New_York'
};
```

### 5. Test

- Click **Execute Workflow** (the Schedule trigger is inactive by default — manual execution still works)
- Click through each node to inspect output. Expected:
  - `TodayEvents`: array with today's planned workout
  - `Macros`: structured context + macro targets
  - `Meals`: Anthropic response with JSON meals
  - `RenderEmail`: HTML body + subject
  - `SendEmail`: confirmation, email arrives in your inbox

### 6. Activate

Once the test email looks right, toggle the workflow to **Active** (top-right of the editor). It will run daily at 5:00 AM Eastern.

## Customizing

### Schedule time

`Schedule` node → cron expression. Examples:
- `0 5 * * *` — 5:00 AM (default)
- `30 5 * * *` — 5:30 AM
- `0 6 * * *` — 6:00 AM
- `0 5 * * 1-5` — 5:00 AM weekdays only

### Macro formulas

All macro math is in the `Macros` node. The intensity buckets and multipliers:
- `hard`: TSS ≥ 70 or duration ≥ 90 min → 1.5 g/kg carbs pre
- `moderate`: TSS ≥ 40 or duration ≥ 60 min → 1.0 g/kg carbs pre
- `easy`: otherwise → 0.6 g/kg carbs pre
- Post (all sessions): 1.0 g/kg carbs + 0.3 g/kg protein
- During (≥90 min): 60–90 g/hr (hard) or 30–60 g/hr (moderate)

Edit those numbers directly in the `buildTargets()` function.

### LLM prompt / meal style

`Meals` node → `jsonBody` field → the `system` prompt controls the meal-suggestion style and output format. Tweak there to bias toward specific cuisines, prep times, or ingredient lists.

### Email layout

`RenderEmail` node → the HTML template is inline. Edit the `html` template literal to change colors, sections, or layout.

### Switch LLM model

`Meals` node → `jsonBody` → change `"model": "claude-haiku-4-5"` to e.g. `claude-sonnet-4-6` for richer suggestions (still cheap at one call/day).

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Credentials not found` on an HTTP node | Credential not selected on the node after import | Open the node → pick the credential from the dropdown |
| `403 Forbidden` on `TodayEvents` / `WeeklyEvents` / `WeeklyWellness` | intervals.icu credential has wrong username | Username must be the literal string `API_KEY`, not your email |
| `403 Forbidden` and URL contains `/athlete/i2694494/` | Web URL form used instead of API form | Strip the `i` — use `/athlete/2694494/` |
| `Header name must be a valid HTTP token` on `Meals` | Anthropic credential's **Name** field has spaces/dashes | Set Name to `x-api-key` (the HTTP header, not a friendly label) |
| Empty meal sections in email | Claude returned non-JSON or wrapped output in markdown fences | Re-run; if persistent, inspect `Meals` raw output and tighten the system prompt |
| Workflow runs but no email | Gmail credential not authorized for `gmail.send` scope | Re-authenticate the Gmail credential |
| No workout in email | Rest day (no event planned in intervals.icu for today) | Expected — email shows recovery-day guidance instead |

## Cost

- intervals.icu API: free
- Anthropic Claude Haiku 4.5: ~$0.003 per run with current prompt size (~1500 input + 1500 output tokens). ~$1/year.
- Gmail: free

## Files

- [`intervals-fueling-workflow.json`](intervals-fueling-workflow.json) — importable n8n workflow
- [`intervals-fueling-workflow.README.md`](intervals-fueling-workflow.README.md) — this file
