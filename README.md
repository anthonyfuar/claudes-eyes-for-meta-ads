<p align="center">
  <img src="https://em-content.zobj.net/source/apple/453/eye_1f441-fe0f.png" width="80" alt="Claude's Eyes for Meta Ads logo" />
</p>

<h1 align="center">Claude's Eyes for Meta Ads</h1>

<p align="center">
  Claude can already read your Meta ad account: spend, results, what is on and off. It cannot see the ads themselves. This skill lets it.
  <br /><br />
  Ask Claude to analyze an ad and you get back a short report that says what the ad is arguing, who it is talking to, what the first seconds do, where the message falls apart, and what to test next. The report is saved as a text file named after the ad.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/node-%E2%89%A520.12-339933?logo=node.js&logoColor=white" alt="Node 20.12+" />
  <img src="https://img.shields.io/badge/dependencies-0-blue" alt="Zero dependencies" />
  <img src="https://img.shields.io/badge/license-MIT-green" alt="License MIT" />
</p>

## What you get

Say you ask:

> Analyze the ad "Cold Brew Kit — UGC 30s".

Claude answers with a summary like this (the brand and the findings here are made up):

- **Hook:** a problem statement — "Steeping overnight is a chore", spoken in the first second
- **Angle:** convenience and time saving, with a live demo
- **Awareness:** the ad assumes the viewer already makes cold brew and hates the wait, and leaves them knowing the product
- **Structure:** hook → demo → how it works → offer → call to action
- **Audience:** adults 25–34, early career, cold traffic — pain: wanting cold brew now, not tomorrow
- **Offer:** free shipping this week, single clear call to action
- **Problem found:** the "how it works" part is only spoken. Muted viewers never get it.
- **Worth testing:** burn captions into that section and watch completion rate.

The full report goes deeper: the funnel stage the ad fits (TOFU, MOFU, BOFU) and the customer awareness level it opens at, the copy framework it follows with an explanation of what that framework does and a beat-by-beat check that the ad really follows it, the word-for-word transcript with start and end times, and a shot-by-shot table of what is on screen, what is said and what text appears in each shot. For an image ad the transcript and timeline are replaced by the full on-screen text and a description of the image. All of it as tables, no JSON.

## Setup

You need three things: a Meta ad account with at least one ad, the Meta Ads connector in Claude, and a free Gemini key from Google.

**1. Install the skill**

```bash
git clone https://github.com/anthonyfuar/claudes-eyes-for-meta-ads ~/.claude/skills/claudes-eyes-for-meta-ads
```

**2. Add your Gemini key**

Get a key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey). In the skill folder, copy `.env.example` to `.env` and paste the key after `GEMINI_API_KEY=`. The key stays on your computer.

**3. Connect Meta Ads to Claude**

- Open **Customize → Connectors**.
- Click **+**, then **Add custom connector**.
- Paste `https://mcp.facebook.com/ads` and save.
- Log in with the Meta Business Manager account that has access to the ad account.

**4. Ask**

> Analyze the ad "Cold Brew Kit — UGC 30s".

That is it. Claude finds the ad, runs the analysis, and answers with the summary.

The skill only reads from your account. It never creates, edits, pauses or spends anything.

## Where things end up

Reports are saved inside the skill folder, in `reports/`, one file per ad, named after the ad. Running the same ad again replaces its report.

The ad's video or image is downloaded into the skill's `downloads/` folder only for the minutes Gemini needs to look at it, then deleted — from your computer and from Google. The small context file Claude writes for the run is deleted too, whether the run succeeds or not. Nothing else is kept.

## Cost

Gemini charges per token. On the default model, a single image costs about 14 thousand tokens and a 37-second video about 59 thousand; check [Google's pricing](https://ai.google.dev/pricing) for what that is in money. A video takes about a minute to analyze, an image under one.

You can choose the model when you ask Claude:

| Flag | Model |
|---|---|
| `--cheap` | `gemini-3.5-flash-lite` — bulk triage of hundreds of ads |
| `--standard` | `gemini-3.6-flash` — the default |
| `--expensive` | `gemini-3.1-pro-preview` — decisions where a wrong label is costly |
| `--model <name>` | Pin any Gemini model |

Every report records exactly how many tokens it used.

## Privacy

- Your ad's video or image and its copy are sent to Google's Gemini and nowhere else. Meta login stays inside Claude's Meta connector.
- On Google's free tier, requests may be used to improve their products. Use a paid key for client accounts.
- The Gemini key lives in a file on your computer and is never written into reports or shared.
- Downloaded media is deleted when the analysis ends, whether it succeeded, failed, or was interrupted.

## What it does not do

- Only single-image and single-video ads. Carousels and catalog ads are not supported.
- Ads that Meta has blocked cannot be analyzed; Claude will tell you.
- The transcript and quotes come from an AI model. They are good evidence, not a legal record.
- Tested on macOS. Nothing in it is Mac-specific; reports from Windows and Linux are welcome.

## Technical notes

Everything above is what a user sees. The rest is for people who want to know what happens underneath or run the script by hand.

### How it works

```
Meta Ads MCP → ads_get_ad_preview → preview_url
                                        ↓
                              scripts/analyze-ad.js

  1. fetch the preview page, lift the real CDN media URL
  2. download to downloads/, hash it
  3. upload to the Gemini Files API, wait until ACTIVE
  4. pass 1 - observation: transcript, timeline, on-screen text
  5. pass 2 - strategy: hook, angle, awareness, audience, flags
  6. write reports/<ad name>.md, delete media, context, upload
```

Two passes because the two response schemas together are about 25 KB and Gemini would not take them as one `responseSchema`. Pass 2 gets pass 1 as its notes.

`ads_get_ad_preview` returns an iframe page, not a file. The script fixes the HTML-encoded `&amp;` in `preview_url`, fetches the page, finds the media URL embedded as a JSON-escaped string (`https:\/\/video…` on `fbcdn.net` for video, the `"imageURI"` containing `t45.1600-4` for images; `t39.30808-1` is the page's profile picture and is skipped), cuts it at the first `"` not preceded by a backslash, and decodes it with `JSON.parse('"' + text + '"')`. Cutting anywhere else returns `403 Bad URL hash` because the CDN signature covers the whole URL. If the context says video but the preview only has a still, the still is analyzed as `POSTER_FRAME` and the report is marked `degraded: true`.

### Files

```
claudes-eyes-for-meta-ads/
├── reports/
│   ├── <ad name>.md                      the report
│   └── <ad name>.context.json            the inputs Claude writes; deleted when the run ends
├── downloads/<run>/                      media.mp4 or media.jpg, deleted when the run ends
└── scripts/analyze-ad.js
Gemini Files API                          the upload, deleted when the run ends
```

Two ads with the same name get `<ad name>--<ad id>.md`.

### Output

Reports default to `reports/` inside the skill folder; `--out` overrides. stdout is one line, the report path. Progress and errors go to stderr. Exit 0 on success, 1 on error, 130 on Ctrl-C. If Gemini leaks or overruns a verbatim field, the script repairs it and adds a `Warning:` line under the report header. `--help` lists everything.

| Error | Meaning |
|---|---|
| `Node 20.12 or newer is required` | Upgrade Node |
| `GEMINI_API_KEY is not set` | No `.env` next to `scripts/` |
| `No downloadable media found in the preview` | Blocked creative or placeholder preview |
| `HTTP 403 … Bad URL hash` | Signed CDN URL expired; get a fresh `preview_url` |
| `HTTP 429` / `5xx` | Retried three times (2 s, 4 s, 8 s), then gave up |
| `finishReason SAFETY` | Gemini declined the creative; not retried |

The report is Markdown only. Sections, in order: at a glance, business context, hook, angle, customer awareness and funnel stage, market sophistication, structure and frameworks (with a beat-by-beat framework check), creative attributes, branding, offer and call to action, proof and claims, persuasion, target audience, coherence diagnostics, strengths, weaknesses, testable hypotheses, transcript, visual timeline, provenance (media role, SHA-256, size, runtime, model, token usage, warnings). Prompts and both Gemini response schemas are embedded in `scripts/analyze-ad.js`.

## License

MIT
