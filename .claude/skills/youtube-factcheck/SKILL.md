---
name: youtube-factcheck
description: Transcribe a YouTube video and fact-check the claims made in it, producing a mobile-friendly HTML report published as a Claude Artifact. Use when the user pastes a YouTube link and asks to transcribe it, fact-check it, or both — e.g. "fact check this youtube video", "transcribe and fact check <url>", "/youtube-factcheck <url>". Also trigger on youtu.be or youtube.com/watch or /shorts links accompanied by a request to check claims, verify facts, or summarize+verify content.
---

# YouTube Transcribe + Fact-Check

Turn a YouTube video into a transcript with its factual claims checked against
the web, delivered as a single self-contained HTML report the user can open
in their phone's browser (published via the `Artifact` tool).

## Inputs

- A YouTube URL (`youtube.com/watch?v=...`, `youtu.be/...`, or
  `youtube.com/shorts/...`). If the user's message doesn't contain one, ask
  for it before doing anything else.

## Steps

### 1. Parse the video ID

Extract the 11-character video ID from whichever URL form was given:
- `youtube.com/watch?v=<id>`
- `youtu.be/<id>`
- `youtube.com/shorts/<id>`

### 2. Fetch metadata and transcript

1. WebFetch the watch page (`https://www.youtube.com/watch?v=<id>`) to get
   the video title, channel name, and publish date, and to confirm caption
   tracks exist (look for `captionTracks` in the page data).
2. Fetch the English caption track from YouTube's public timedtext endpoint:
   `https://video.google.com/timedtext?lang=en&v=<id>`
   (if that 404s or is empty, retry with `&kind=asrs` for auto-generated
   captions, and fall back to whatever caption track language the watch page
   metadata advertises if no English track exists).
3. Parse the returned XML into an ordered list of `{timestamp, text}`
   segments, then flatten into a continuous transcript. Keep the timestamps
   — they're needed later to anchor each claim.
4. **If no caption track exists at all**: stop here, tell the user this
   specific video has no captions available so it can't be transcribed via
   the public captions endpoint (this skill does not do audio transcription),
   and don't attempt to fabricate a transcript.

### 3. Extract checkable claims

Read through the transcript and pull out discrete, checkable factual
assertions — statistics, historical/scientific claims, quotes attributed to
someone, named-event claims, causal claims ("X caused Y"). Skip pure opinion,
predictions, and rhetorical filler. Note the approximate timestamp each claim
occurs at.

Don't try to fact-check every sentence — aim for the claims that are
substantive and actually checkable, typically somewhere between 5 and 20
depending on video length and density.

### 4. Fact-check each claim

For each extracted claim, use WebSearch (and WebFetch on the most promising
results) to check it against independent sources. Assign:

- **Verdict**: one of `True`, `False`, `Misleading`, `Needs Context`,
  `Unverifiable`
- **Explanation**: 1–3 sentences on what the evidence actually shows
- **Sources**: 1–3 links to the sources used

Prefer primary sources, reputable news organizations, official statistics
bodies, and encyclopedic references over blogs/forums. If sources disagree,
say so in the explanation rather than forcing a verdict.

### 5. Build the HTML report

Load the `artifact-design` skill before writing the HTML (required before
any Artifact publish) and design a single self-contained page with:

- **Header**: video title, channel, a link back to the original video,
  generation date.
- **Summary bar**: counts by verdict (e.g. "8 True · 2 False · 3 Needs
  Context · 1 Unverifiable").
- **Fact-check cards**, one per claim: the quoted claim + its transcript
  timestamp, a color-coded verdict badge, the explanation, and linked
  sources.
- **Full transcript** in a collapsible section at the bottom, with
  timestamps, for reference.
- Mobile-first responsive layout (this is read on an iPhone in Safari) —
  single column, comfortable tap targets and line length, light/dark theme
  tokens per the Artifact rules. No external CDN, font, or image
  dependencies — everything inline, since Artifacts run under a strict CSP.

### 6. Publish

Publish with the `Artifact` tool: a short descriptive title (the video's
subject, not literally "Fact Check"), a fitting favicon emoji (e.g. 🔍 or
🎬), and a one-sentence `description` naming the video. Give the user the
resulting link. Each video produces its own fresh Artifact — don't try to
append multiple videos onto one running page.

## Notes / limitations

- This relies on YouTube's public caption tracks. Videos with no captions
  (auto-generated or manual) in any language can't be transcribed by this
  skill — say so plainly rather than guessing at content.
- This is a per-request workflow run by a live Claude Code session, not a
  standalone installable app — no API keys to manage, but it requires an
  active session to invoke it.
- **Network policy**: some sessions/environments have a restrictive egress
  policy that blocks `WebFetch` to arbitrary domains (including
  `youtube.com`, `video.google.com`, and even sites like Wikipedia), while
  `WebSearch` still works. If step 2 or step 4 gets an `EGRESS_BLOCKED` (or
  similar) error from WebFetch, do not silently give up or fabricate a
  transcript/fact-check from guesswork: tell the user plainly that this
  session's network policy is blocking direct fetches to the domain in
  question, note that `WebSearch` results may still be usable for
  fact-checking (they come back as summarized snippets even when WebFetch
  is blocked), and suggest running this skill from a session/environment
  whose egress policy allows reaching general web domains if a full
  transcript is required.
