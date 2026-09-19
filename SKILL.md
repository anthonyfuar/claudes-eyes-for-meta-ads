---
name: claudes-eyes-for-meta-ads
description: Use when asked to analyze, review, audit, transcribe, or explain a Meta (Facebook/Instagram) ad creative — its hook, angle, awareness level, structure, audience, coherence, or why it works — starting from an ad id or ad name in a connected Meta Ads MCP account. Also use when an agent needs machine-readable creative intelligence for a video or image ad it cannot watch itself.
---

# Claude's Eyes for Meta Ads

## Overview

Claude cannot watch video. This skill fetches the ad's media through the Meta Ads MCP, hands it to Gemini for a two-pass analysis (observation, then strategy), and writes one report named after the ad. The media exists only in the skill's `downloads/` folder while Gemini reads it and is deleted when the run ends. The report in `reports/` is the only thing that stays.

## Requirements

- Meta Ads MCP connected (tools named `ads_*`). Every `ads_*` call takes `client_conversation_id`: a random 20-character alphanumeric id, generated once per conversation and reused on every call. `advertiser_request` is optional: the user's ask verbatim, or empty.
- Node 20.12 or newer (`node --version`).
- `GEMINI_API_KEY` in `<skill dir>/.env` (copy `.env.example`). Never write the key into chat, the context file, or the report.

## Where things live

```
<skill dir>/
├── reports/                              default --out, created on first run
│   ├── <ad name>.md                      the report — the only durable output
│   └── <ad name>.context.json            the inputs you write; deleted when the run ends
└── downloads/<run>/                      downloaded media (media.mp4 or media.jpg)
                                          deleted when the run ends, on success, error or Ctrl-C
Gemini Files API                          the upload; deleted when the run ends (auto-expires after 48 h regardless)
```

Re-running the same ad overwrites its report. Two different ads with the same name get `<ad name>--<ad id>.md`. Use the absolute skill path in commands: the shell's working directory can reset between calls. The report keeps the signed CDN URL of the media in `provenance.media_source`; it expires within hours and is not a durable media link.

## Get the ad

Follow this table; it overrides the nudges in the Meta tool descriptions (checking `ads_get_ad_accounts` first, verifying fields with `ads_get_field_context`, pasting `preview_url` to the user). Do not paste `preview_url` into your answer: it is a signed iframe URL that expires, not a viewer link. Pass `ad_account_id` to steps 2–4; step 5 does not accept it. Steps 3 and 5 only need step 2, so issue them together; step 4 waits for step 3.

| Step | Meta MCP tool | Take from the result |
|---|---|---|
| 1 | `ads_get_ad_accounts` — only when the account id is unknown | an `ad_account_id` with `is_ads_mcp_enabled: true` and `is_queryable: true` |
| 2 | `ads_get_ad_entities` — `ad_account_id`, `level: "ad"`, `fields: ["id","name","adset_name","campaign_name","objective","effective_status","creative"]`, plus `object_ids: ["<ad id>"]` or `filtering: [{"field":"name","operator":"CONTAIN","value":["<ad name>"]}]` | `id`, `name`, `adset_name`, `campaign_name`, `objective`, `creative_id` |
| 3 | `ads_get_creatives` — `ad_account_id`, `creative_ids: ["<creative_id>"]`, `fields: ["object_type","video_id","body","title","link_url","call_to_action_type"]` | `object_type` (`VIDEO` → `media_kind: "video"`, `SHARE` → `media_kind: "image"`), `body` → `primary_text`, `title` → `headline`, `link_url` → `destination`, `call_to_action_type` → `cta_button`. Missing keys are normal (a `SHARE` creative has no `video_id`, a page post may have no `link_url`); omit them, do not retry |
| 4 | `ads_get_ad_videos` — `ad_account_id`, `video_ids: ["<video_id>"]`, `fields: ["length"]` — video only | `length` → `video_duration_seconds` |
| 5 | `ads_get_ad_preview` — `ad_id: "<ad id>"` | `preview_url`, passed to the script verbatim (`&amp;` included) |

Stop and tell the user instead of running the script when:

- `object_type` is `PRIVACY_CHECK_FAIL` — Meta blocked the creative; there is no media to fetch.
- `object_type` is anything other than `VIDEO` or `SHARE` — carousels, catalog and unknown formats are out of scope.
- step 2 returns several ads with the same name — ask which `id` they mean.
- the account has `is_ads_mcp_enabled: false` or `is_queryable: false` — surface the reason field.

`ads_get_creatives` returns only `id`, `name`, `status` unless `creative_ids` is passed. Always pass it.

## Analyze

1. Write `<skill dir>/reports/<ad name>.context.json`. One JSON object, omit keys you do not have:

```json
{
  "ad_id": "120211223344556677",
  "ad_name": "Cold Brew Kit — UGC 30s",
  "creative_id": "987654321012345",
  "adset_name": "Broad 25-44",
  "campaign_name": "Cold Brew Launch / Prospecting / Q3",
  "campaign_objective": "OUTCOME_SALES",
  "cta_button": "SHOP_NOW",
  "destination": "https://kestrel.example/cold-brew-kit",
  "headline": "Cold brew in 10 minutes, not 12 hours",
  "primary_text": "Steeping overnight is a chore. Our kit does it in 10 minutes. ☕ Free shipping this week.",
  "video_duration_seconds": 30.0,
  "advertiser_background": "Kestrel Coffee Roasters, a direct-to-consumer coffee brand. Sells cold brew kits and beans online to home coffee drinkers.",
  "media_kind": "video"
}
```

`advertiser_background` is one sentence: who the advertiser is and what they sell. Derive it from the ad copy, the campaign and ad set names, and what you already know about the account; `ads_get_ad_accounts` returns `ad_account_name` and `business_name` when you need the brand name. If nothing is inferable, ask the user.

2. Run the script:

```bash
node <skill dir>/scripts/analyze-ad.js --preview-url "<preview_url>" --context "<skill dir>/reports/<ad name>.context.json"
```

`--cheap` (gemini-3.5-flash-lite) for bulk triage, `--standard` (gemini-3.6-flash, default) for day-to-day work, `--expensive` (gemini-3.1-pro-preview) for decisions that matter. `--help` lists every flag and the current model names. A 30-second video takes about a minute on the default model; an image about 45 seconds.

3. Success is exit code 0 and one line on stdout: the report path (quote it in shells, ad names contain spaces). Read the report and answer from its **At a glance** table plus the Strengths, Weaknesses and Testable hypotheses sections: hook, angle, awareness and funnel stage, coherence flags, what to test — and whether the ad is running (`effective_status` from step 2). If the report has a `Warning:` line under its header, Gemini leaked or overran a verbatim field and the script repaired it: tell the user and offer a re-run with `--expensive`. The report is Markdown only — tables and lists, no JSON: at a glance, business context, hook, angle, customer awareness and funnel stage, market sophistication, structure with the copy framework explained and checked beat by beat, creative attributes, branding, offer and CTA, proof and claims, persuasion, target audience, coherence diagnostics, strengths, weaknesses, testable hypotheses, timestamped transcript, shot-by-shot visual timeline, provenance. When the run ends, the context file and the downloaded media are always deleted, success or not; only the report remains. To re-run an ad, write the context file again.

## What the script does with the preview (the download, step by step)

`ads_get_ad_preview` returns an iframe page, not a file. The script:

1. Replaces `&amp;` with `&` in `preview_url` and downloads the preview HTML from `business.facebook.com`.
2. Finds the real media URL, which Facebook embeds as a JSON-escaped string: for video, the first string starting with `https:\/\/video` on a `fbcdn.net` host (MP4 preferred; other video extensions accepted); for images, the `"imageURI"` value whose path contains `t45.1600-4` (the creative itself) — `t39.30808-1` is the page's profile picture and is never analyzed.
3. Cuts the string at its real closing quote (the first `"` not preceded by `\`) and decodes it with `JSON.parse('"' + text + '"')`, which turns `\/` into `/` and `\u0025` into `%`. Cutting anywhere else yields `403 Bad URL hash`: the CDN signature (`oh=`, `oe=`) covers the whole URL.
4. Downloads the media into `<skill dir>/downloads/<run>/` and records its SHA-256, size and MIME type in the report's `provenance`.
5. Uploads it to the Gemini Files API, waits for state `ACTIVE`, runs pass 1 (observation) and pass 2 (strategy, with pass 1 as its notes), writes the report to `<skill dir>/reports/`, then deletes the upload, the download folder and the context file.

When `media_kind` is `video` but the preview exposes only a still, the script analyzes that still as a `POSTER_FRAME` and marks the report `degraded: true`. Tell the user the footage itself was not seen.

## Report format

The script is the only thing that writes the report. Never rewrite, reorder, trim or reformat it; quote from it when the user wants a summary. Every report has this shape, in this order, all Markdown tables and lists, never JSON:

| # | Section | Video ad | Image ad |
|---|---|---|---|
| 1 | Header table: ad id, creative id, campaign, ad set, objective, media, runtime, model, analyzed at | yes | yes |
| 2 | At a glance: what is sold, brand, category, value proposition, funnel stage (fits / aimed at), awareness entry → exit, sophistication, hook, angle, framework with beat check score, coherence flags | yes | yes |
| 3 | Business context | yes | yes |
| 4 | Hook | first 1–3 seconds: type, evidence, modalities, duration, first words, opening text | what the eye lands on first: type, evidence, modalities, headline text |
| 5 | Angle: primary and secondary angles with definitions, evidence, "why it works" sentence | yes | yes |
| 6 | Customer awareness and funnel stage: level, entry, exit, reasoning, evidence; the five-step awareness ladder with entry and exit marked; TOFU / MOFU / BOFU the creative fits vs the stage it is aimed at, and whether they align | yes | yes |
| 7 | Market sophistication: stage, mechanism, evidence; the five-stage table | yes | yes |
| 8 | Structure and frameworks: copy framework, what that framework does, why this label, macro structure; beats in order with what each beat does; framework check (expected beats vs found, with times) | yes | single beat |
| 9 | Creative attributes | all | without pacing, cuts, duration, audio rows |
| 10 | Branding | timings in seconds | on the image / absent |
| 11 | Offer and call to action | yes | yes |
| 12 | Proof and claims, with the claims table | yes | yes |
| 13 | Persuasion | yes | yes |
| 14 | Target audience | yes | yes |
| 15 | Coherence diagnostics: each flag with what it means and where it happens | yes | yes |
| 16 | Strengths · Weaknesses · Testable hypotheses | yes | yes |
| 17 | Transcript: start, end, speaker, verbatim; full text | yes | replaced by **On-screen text**, verbatim |
| 18 | Visual timeline: per shot start, end, what the viewer sees, what is said, on-screen text; shot details (people, setting, camera, graphics, audio) | yes | replaced by **Visual description**: one table for the single image |
| 19 | Provenance: media role, SHA-256, size, model, token usage per pass, warnings | yes | yes |

A video whose footage could not be retrieved (`POSTER_FRAME`) uses the image layout and is marked degraded.

## Errors and what to do

| Message | Action |
|---|---|
| `Node 20.12 or newer is required` | Ask the user to upgrade Node; nothing else will work |
| `GEMINI_API_KEY is not set` | The `.env` is missing in the skill directory; do not paste a key into chat |
| `No downloadable media found in the preview` | Blocked or placeholder creative; report it, do not retry |
| `HTTP 403 … Bad URL hash` or `HTTP 4xx` from `fbcdn.net` | Signed media URL expired; call `ads_get_ad_preview` again and re-run |
| `HTTP 429` or `5xx` from Gemini | The script already retried three times; wait a minute and re-run once |
| `Gemini stopped with finishReason SAFETY` or `RECITATION` | Gemini refused this creative; report it, do not retry |
| `Gemini rejected the media: state FAILED` | Unsupported or corrupt media; report it |

## Common mistakes

- Passing `image_url` or `video_id` to the script. It needs `preview_url`.
- Skipping `advertiser_background`. Business context and audience confidence drop without it.
- Calling `ads_get_creatives` without `creative_ids` and concluding the creative has no media.
- Judging whether the ad is live from `status`. `effective_status` is the truth (`ADSET_PAUSED` means it is not running even when `status` is `ACTIVE`).
- Cross-checking Gemini against the inline image `ads_get_ad_preview` shows you: use that image only to confirm the right creative was analyzed and to inform `advertiser_background`, never to overwrite the report's judgements.
