# Shorts Studio — handoff notes

## What this is
A single-file, no-backend web tool that turns a topic into a ready-to-upload 30-second YouTube Short. Same pattern as ResuTailor and SportsWeek: vanilla HTML/CSS/JS, hosted on GitHub Pages, user brings their own API key (stored in localStorage), zero server cost.

File: `shorts-generator.html` (attached alongside this doc).

## Pipeline
1. **Topic → script** — Calls `gemini-2.5-flash` (`generateContent`) to write a ~30s script: 68–78 spoken words, split into 5–7 caption lines, plus a title and 5 hashtags. Returns structured JSON (`responseMimeType: application/json`).
2. **Script → voiceover** — Calls `gemini-2.5-flash-preview-tts` with the full script text, `responseModalities: ['AUDIO']`, and a `speechConfig.voiceConfig.prebuiltVoiceConfig.voiceName` (one of Kore / Puck / Charon / Fenrir). Response comes back as base64 PCM audio in `inlineData.data`, with sample rate embedded in `inlineData.mimeType` (parsed via regex `rate=(\d+)`). The code wraps the raw PCM16 bytes into a WAV header (`pcm16ToWavBlob`) and decodes it via `AudioContext.decodeAudioData`.
3. **Caption timing** — Line durations are apportioned across the audio's total duration by word count per line (`computeLineTimings`), not by real speech alignment.
4. **Canvas render** — 1080×1920 canvas draws an animated gradient + drifting particles + the active caption line (word-wrapped, faded in/out) + a progress bar, driven by `drawFrame(t)`.
5. **Preview** — Plays audio through `AudioContext.destination` while looping `drawFrame` via `requestAnimationFrame`, synced off `audioCtx.currentTime`.
6. **Render/export** — Combines `canvas.captureStream(30)` video tracks with an `AudioContext.createMediaStreamDestination()` audio track into one `MediaStream`, fed into `MediaRecorder` (`video/webm;codecs=vp9,opus` if supported). On stop, produces a downloadable `.webm` blob.

## Known risk / likely first bug
The Gemini TTS request/response shape in step 2 (model name, `responseModalities`, where sample rate lives in `mimeType`, exact `inlineData` structure) was written from training knowledge, **not verified against a live call** — I had no network access to test it. This is the single most likely thing to break first. When testing with a real key:
- Log the raw response from the `gemini-2.5-flash-preview-tts` call to confirm the actual JSON shape.
- Confirm audio encoding (should be PCM16 mono) and sample rate parsing.
- Adjust `pcm16ToWavBlob` / the mimeType regex if the field names differ.

## Next steps, roughly in priority order
1. Fix any TTS response mismatches found in live testing (see above).
2. Word-by-word caption reveal as an animation option (line-by-line is the current default) — word-level tends to perform better on Shorts.
3. Guard against script length drift (word count check post-generation; regenerate or trim if the audio duration lands far outside ~28–32s).
4. Batch/daily mode: generate N days of shorts (topic + script + voiceover + render) in one pass, since the goal is a daily 30s output rather than one-offs.
5. Optional: swap the plain gradient/particle background for a slightly more distinctive visual treatment once the pipeline itself is solid.

## Context for whoever picks this up
Portfolio project #3 in a set of zero-cost, no-backend web tools (alongside ResuTailor, an AI resume tailoring tool, and SportsWeek, a multi-sport schedule aggregator), built by a solo developer establishing an AI-as-a-Service freelance offering. Design direction: dark "editing suite" aesthetic — charcoal background, cyan/teal accent for timeline and active states, red accent for record/live states, condensed display type for headers, monospace for timecodes.
