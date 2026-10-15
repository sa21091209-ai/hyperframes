---
name: video-ad-editor
description: Turn a talking-head video into a 9:16 reel or ad — cut the silences, add word-timed Arabic captions, and layer motion graphics in the brand's colors; mobile-first, runs from a phone chat. Trigger on "منتج هذا المقطع", "سو من هذا الفيديو إعلان", "شيّل السكتات", "حوّل الفيديو لريل", "ركّب كابشن عربي", "اقص الصمت", or a talking-head video attached with an edit request. Not for carousels or video covers alone. Plain captions only → /embedded-captions; graphic overlays only → /talking-head-recut. Unclear → /hyperframes.
---

# Video Ad Editor

Talking-head video in, one publish-ready 9:16 MP4 out (plus `.srt` and a transcript `.txt`). The edit is all code: `ffmpeg` cuts, `whisper` transcribes with word timing, and HyperFrames renders the captions and motion graphics over the footage.

> **The front door is `/hyperframes`.** This skill owns the pipeline for one talking-head clip → reel. Silence cutting is the part the sibling skills do not cover; captions and overlays are delegated to them (table below). Any other intent → `/hyperframes` first.

## Mobile-first

The primary path is a phone: the user attaches the video in the Claude app, the work runs in the cloud session, and the phone can close.

1. User attaches the video and states the request in one sentence.
2. Work runs in the background. Tell them: "النتيجة بتنتظرك بنفس المحادثة."
3. Deliver the MP4 in the conversation with a short summary (what changed, what was replaced, what was not checked).
4. Notes like "القص يخرب الشعر" are fixed without re-shooting.

Phone and cloud constraints:

- All interaction is in chat. There is no browser or terminal on the user's side: ask style questions (colors, font) in one chat question, never hand over commands.
- Previews are one contact sheet of 6 frames (3 columns × 2 rows) per stage, never single images.
- Keep the deliverable under 30 MB. Work on a compressed copy for previews if the source is large.
- The container is ephemeral: keep anything that matters (result, `fixes.json`, theme) in the conversation or the repo.
- Cloud is slower than a local machine: estimate time before heavy stages (transcription, render).

Substitutes when a Mac-only tool is missing — try one sentence of verification before building on it, and tell the user in one line what you used:

| Step            | Preferred    | Fallback                                                |
| --------------- | ------------ | ------------------------------------------------------- |
| Transcription   | `whisper`    | `faster-whisper`                                        |
| Person cut-out  | Vision (Mac) | `rembg` (`u2net_human_seg`) or MODNet; check hair/hands |
| Face tracking   | Vision (Mac) | MediaPipe or OpenCV; else center crop with a warning    |
| Frame rendering | HyperFrames  | headless Chromium + `ffmpeg`                            |

Never tell the user "this can't run on your phone."

Desktop (later): same stages and rules. Differences are local `whisper`, `npx hyperframes preview` instead of contact sheets, and faster renders. Enable when the user says "أبي أعدّل بنفسي".

## Who does what

| Stage                        | Tool                                                |
| ---------------------------- | --------------------------------------------------- |
| Silence cut, zoom, audio mux | `ffmpeg` / `ffprobe`                                |
| Word-timed transcript        | `whisper` or `/media-use`                           |
| Captions over the footage    | `/embedded-captions`, `/captions-overlay`           |
| Titles, cards, motion        | `/talking-head-recut`, `/hyperframes-animation`     |
| Seams between scenes         | `/motion-doctrine`, `/cut-the-curve`, `/seam-craft` |
| Sound effects, loudness      | `/hyperframes-audio`, `/media-use`                  |
| Gate and render              | `npx hyperframes lint`, then `check`, then `render` |

Read `/hyperframes-core` before writing any composition HTML.

## How to work with the user

- You prepare, they do not. Missing a tool? "ناقصني أداتين أجهّزهم لك الحين، ياخذ دقيقتين — أبدأ؟"
- No jargon: "أداة ترسم الشرائح", not "puppeteer".
- Say where you are and how long it takes: "شلت السكتات — نصف الفيديو راح. الحين أفرّغ كلامك، 3 دقايق."
- One thing at a time; show a frame or summary after each major stage.
- Do not ask "which mode?" — say in one sentence what you understood and continue.

## Stages

1. **Receive.** 4K is better (zoom crops the source) but 1080p works; never ask for a re-shoot. Check duration with `ffprobe`. Landscape source (interview, podcast): same stages, crop tall around the face instead of stretching.
2. **Dropped frames.** Browser or non-native camera recordings drop frames. List gaps over 0.07 s with `ffprobe -select_streams v -show_entries frame=pts_time`; if any, repair with `ffmpeg -vf minterpolate=fps=30:mi_mode=mci:mc_mode=aobmc:me_mode=bidir:vsbmc=1` before cutting, and tell the user.
3. **Cut plan.** Detect silences and plan the cuts. Report: "شلت 52 ثانية سكوت — الفيديو صار 46 بدل 98."
4. **Audio cleanup (optional).** Echo or noise, or "حسّن الصوت": do it before transcription so whisper hears clean audio. Keep the original.
5. **Transcribe.**

   ```bash
   ffmpeg -v error -i src.mov -vn -ac 1 -ar 16000 -y a.wav
   python3 -m whisper a.wav --language ar --model medium --word_timestamps True \
     --output_format json --output_dir . --fp16 False --verbose False > whisper.log 2>&1
   ```

   Takes minutes: run in the background and say how long.

6. **Correct.** Whisper misses Gulf dialect. Fix every sentence in `fixes.json` (`fix` = word-for-word replacements, `hot` = words to emphasize). The corrected word count per sentence must equal whisper's for that sentence or the timings drift. Never invent words the speaker did not say; when unsure, generalize.
7. **Script review (mandatory).** Print the full transcript in chat, numbered with timestamps, and ask "شنو تبي أشيل؟" Detect repeated sentences (the first is the mistake, the second the fix) and offer to drop the first. Do this before designing any scene: deleting later shifts every time. After any deletion, redo the cut and zoom, and re-render fully.
8. **Cut and zoom.** Apply the plan. When cards are on screen, crop from above the head with a fixed crop for the whole video: start just above the highest point the head reaches (1.5% margin). Never cut the head, never leave a big empty ceiling. Verify with a close-up frame.
9. **Design scenes** from the meaning of the speech, not from a template.

   | Moment                   | Layout    | Shape                                                      |
   | ------------------------ | --------- | ---------------------------------------------------------- |
   | Talking, no graphic      | `R_FULL`  | Face full screen, caption above y 1460                     |
   | Any graphic or motion    | `R_DOWN`  | Graphic on top (y 280-520), caption, face below full width |
   | B-roll or large panel    | `R_LOWER` | Big card on top, caption, small face below                 |
   | Full explainer (numbers) | `R_OFF`   | No face, voice continues                                   |

   Captions never sit on the speaker's face. Safe zones (Instagram UI): no text at y 0-150; none at y 1620-1920 (caution band 1500-1620); none at x 950-1080 between y 1100 and 1750. The first caption appears before 0.5 s. Preview one contact sheet of 6 frames before the full render.

10. **Sound effects.** Only on meaningful moments (a number drops, something breaks, a scene changes), max 15 events per minute, peaks under -18 dBFS. Show the user where they land before export.
11. **Render and mux.** Render frames, mux with the audio. Rendering resumes where it stopped; after a scene edit re-render only that window.
12. **Loudness.** Master to -14 LUFS, true peak -1.5 dBTP or lower. A background audio file, if given, is mixed in and ducked under speech.
13. **Subtitles.** `.srt` for YouTube and LinkedIn; `.txt` with the full speech for the post caption.

## Fixed rules

1. Western digits (0-9) everywhere.
2. Colors come from the theme only; no hard-coded color in code.
3. Zero color treatment on the speaker's footage: no grade, tint, LUT or blur unless explicitly asked. Colors apply to cards and text only.
4. iPhone HDR: tone-map to SDR properly (`zscale` with `tonemap=hable`); naive conversion fades the image and reads as a grade.
5. No publishing or scheduling — the deliverable is a file.
6. Call it "ملف صوتي بالخلفية", never "موسيقى"; its content is the user's call.
7. Invent new scenes every time. Examples are a pattern library, not a template.
8. Images, screen recordings and logos keep their original aspect ratio: scale only, no squashing or cropping unless asked.
9. Quran text is copied verbatim and programmatically from a source (`api.quran.com` → `text_imlaei`, or `api.alquran.cloud`), checked against a second source before render. Prefer the vowelled imlaei text over Uthmani; Uthmani font: `Amiri Quran`. Anything that is not a verse ("قال الله تعالى", a surah name) uses the normal theme font.
10. Zoom-out is only a return from a zoom-in; it never turns the frame into a card or goes smaller than the original.
11. The hook has motion in the first second (a fast soft zoom-in or elements rising from behind). A static first frame reads as a scroll.
12. A popped-in word stays until its sentence ends (0.6 s minimum), never a half-second flash.
13. Off by default unless asked by name: letter-by-letter typing, glow, bounce, gradient text, purple-to-blue backgrounds. If glow is asked, apply it full-screen — a bounded rectangle shows edges.
14. A deliberately calm speaker (Quran, sermon, reading): ask "أخلي السكتات مثل ما هي؟" and if yes, do not cut silences — cutting here produces jumps.

## Verify before delivery

1. Before the full render: a frame from the middle of every scene. Look for clipped text, overlaps, numbers not from the facts list, crop edges around the person.
2. Safe zones and first-caption timing pass.
3. Sync: transcribe the output and compare sentence starts to the captions; difference under 0.1 s.
4. Audio: about -14 LUFS, peak at or under -1.5 dBTP.
5. Eyes: one 6-frame contact sheet from the final file, actually looked at.
6. Size under 30 MB.
7. `npx hyperframes lint` then `npx hyperframes check` pass.

## Delivery

Show the file with: duration, silence removed, scene count, final loudness. Attach the `.srt` and `.txt`, and say you did not publish anything.

The cover is the first frame of every video:

- **Safe square.** The reel is 9:16 but Instagram shows 4:5 (sometimes 1:1) on the profile. Everything important — face, text, logos — goes inside a 1080×1080 square at the center (y 420-1500). The square is imaginary: no border or line.
- **Hook text only:** two lines, 5 words max, about 9 characters per line, on the chest or just below.
- **Phone-readable sizes.** The grid shrinks the cover to about 12%: title at least 170 px, any other text at least 110 px, logo at least 200 px.
- **Person photo.** If they have a cut-out with transparent background, put it on a background matching the video's; otherwise use the clearest source frame.
- **Two-logo comparison** (✕ old, ✓ new) only when the story is "from X to Y": a row at the top of the square with the head below.
- Put the cover in the first 0.1 s of the video; that file is the deliverable.
- Show a preview at 9:16, 4:5 and 1:1 side by side and confirm nothing is cropped at 1:1.
