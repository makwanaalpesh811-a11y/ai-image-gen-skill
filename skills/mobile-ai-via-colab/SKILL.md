---
name: mobile-ai-via-colab
description: Run mobile AI via Colab on Termux — no PyTorch, no GPU.
---

# Mobile AI via Colab

Termux cannot run PyTorch on Python 3.14 (no wheels for `aarch64` + `3.14`), cannot fit >1 GB models in ~1 GB available RAM, and has no GPU. Voice cloning, talking-avatar, and heavy TTS inference are all Colab-only tasks. This skill captures the standard pattern: prepare inputs locally, hand the user a one-click Colab notebook link, and process the downloaded output with local ffmpeg/edge-tts.

## When to use
- User wants voice cloning, talking-head, lip-sync, style transfer, or any model >500 MB on Termux.
- User asks for a workflow that cannot run locally (PyTorch, Diffusers, HuggingFace Transformers >1 B params).
- User asks about "mobile", "phone", "Colab", or names a specific AI service that requires browser auth.

## Never claim these work locally
Do not say "let me install PyTorch on Termux" or "I can run RVC/VoxCPM/SadTalker here". PyTorch wheels are absent for the current Python version and CPU-only inference of large TTS models exceeds available RAM. If the user pushes back, the honest answer is: use Colab, or use a smaller local model if one exists for their exact task.

## Standard pattern

1. **Prepare inputs locally** — reference audio, image, text, script. Save to `/sdcard/Hermes_Almari/...` so the user has them in one place.
2. **Pick the right Colab notebook** from `references/colab-notebooks.md`. Prefer ones with a working "Open in Colab" badge and a maintained README.
3. **Give the user the exact URL** and a 5-step checklist:
   - Open the notebook URL in Chrome/Firefox
   - `Runtime → Change runtime type → Hardware accelerator → T4 GPU → Save`
   - Run the "Mount Google Drive" cell first (caches model weights, avoids re-download each session)
   - Upload reference audio + image to Drive, not via notebook widget (widget is flaky)
   - Run cells in order; download output MP3/MP4 from Drive
4. **Never promise to run it for the user.** Browser auth + Google account are out of reach. Phrase it as "open this URL, follow the 5 steps, drop the output back here."
5. **Post-processing** (ffmpeg combine, edge-tts, screenshot collages, thumbnail generation) is local — do that part fully.

## Voice cloning tools (from lightest to heaviest)

| Tool | Colab link | Best for | Ref audio | Languages |
|------|------------|----------|-----------|-----------|
| VoxCPM2 | `skalaliya/voice-clone` | Multilingual incl. Hindi | 30 s | 30+ incl. Hindi |
| Qwen3-TTS | `Myuniqous/qwen3-tts-voice-cloning-google-collab` | Fast, high quality, English/Asian | 3–10 s | 10 (no Hindi) |
| RVC | various Colab forks | Music/singing, one-voice | 1–2 min | Any (per model) |
| SadTalker | `camenduru/wav2lip-colab` sibling | Talking head from 1 image + audio | image + audio | Any |
| Wav2Lip | `camenduru/wav2lip-colab` | Anime face lip-sync | image + audio | Any |
| SoulX FlashHead | See `references/colab-notebooks.md` | Newest, fast | image + audio | Any |

Full links and per-tool setup notes: `references/colab-notebooks.md`.

## Voice profiles on edge-tts (local, no Colab)

These are the confirmed parameters for anime-girl-style English narration. Save a chosen profile as JSON next to the caller script — never hardcode the tuple.

| Character vibe | Voice | Rate | Pitch |
|---|---|---|---|
| Soft anime female (Hinata-style) | `en-US-JennyNeural` | `-20%` | `+25Hz` |
| Softer, more anime | `en-US-AriaNeural` | `-20%` | `+25Hz` |
| Max soft (very shy) | `en-US-AriaNeural` | `-25%` | `+30Hz` |
| Japanese-accented English (Hinata JP) | `ja-JP-NanamiNeural` | `-15%` | `+15Hz` |
| Soft Hindi female | `hi-IN-SwaraNeural` | `-20%` | `+25Hz` |
| Soft Hindi male | `hi-IN-MadhurNeural` | `-15%` | `+10Hz` |

Full parameter recipes and pitfalls: `references/voice-profiles.md`.

## edge-tts pitfalls

- **Language/voice mismatch silently returns empty files.** Feeding Hindi text to `ja-JP-NanamiNeural` produces 0-byte MP3s with no error. Test with the matching language; if you need Japanese voice + Hindi text, they simply don't mix — use Swara for Hindi.
- **Percent signs in shell**: `--rate -20%` inside a Python f-string or shell command needs proper escaping. Build the command as a list and pass via `subprocess.run`, not a shell string, so `%` never hits bash.
- **/tmp is not writable on some Termux setups.** Write audio/video to `/sdcard/Hermes_Almari/...` or `$PREFIX/tmp`, never `/tmp`.
- **edge-tts list-voices**: `edge-tts --list-voices | grep <lang-code>` gives every voice for a language. Hindi has only 2 (Swara female, Madhur male). Don't assume more.

## Gemini API (local, works from Termux)

- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={KEY}`
- **Working model as of now: `gemini-2.5-flash`.** `gemini-2.5-pro`, `gemini-2.0-flash`, `gemini-1.5-*` all return HTTP 404 on the current API version. Always try 2.5-flash first, fall back only if it 404s.
- Store the key in `~/.ig_env` (chmod 600) — do not print it in tool output. Load via `export $(grep KEY ~/.ig_env)` or read the file.
- For script generation, `temperature: 0.7-0.8` and `maxOutputTokens: 500-1024` give better results than default. For JSON output, wrap prompt with explicit `Return as JSON:` — Gemini often wraps JSON in markdown fences, so strip them.
- Free tier rate limit is per-minute, not daily. Sleep 1-2 s between bursts.

Full model matrix, prompt patterns, and error codes: `references/gemini-api-notes.md`.

## Avatar generation (needs Colab, always)

Talking avatar from a still image + audio is Colab-only. Never promise local generation. Standard recipe:
1. Prepare: `hinata_image.jpg` (clean face, no occlusion) + `voice.mp3` (any language, 60-120 s ideal)
2. Wav2Lip Colab → upload both → run → download `output.mp4` (~4 MB, 30 fps)
3. SadTalker if you want head motion + blinking (slower, prettier)
4. Post-process locally with ffmpeg: `-vf "scale=1080:1920" -c:v libx264 -preset fast -crf 23 -pix_fmt yuv420p`

For anime-style characters, Wav2Lip gives more consistent lip-sync than SadTalker (SadTalker was trained on real faces). See `references/colab-notebooks.md` for links.

## Legal note (anime characters)

Voice cloning or avatarizing a copyrighted anime character (Hinata, Goku, Naruto, etc.) is:
- **Personal non-monetized channel:** low risk
- **Monetized YouTube/Instagram:** medium risk (DMCA possible)
- **Commercial / paid content:** high risk

Always disclose this to the user before they start. Recommend a non-monetized fan channel and credit the source character in description. Never advise the user to monetize a fan character voice without warning.

## Do not save
- Colab session state, model download URLs, Drive folder IDs — those are user-specific and rotate.
- Specific prompt templates that only work for one user's niche — put those in the caller's project files, not here.
- Failed attempts with "try X again" fixes — those belong in memory if they're session-specific, not in a skill.
