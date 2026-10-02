# Spec: Batch audio transcriber for a local share / cloud bucket

A build spec you can hand to a coding agent. It captures the design + the
non-obvious lessons from Office Hours Global's community transcription pipeline
(which transcribed ~2,300 hour-long episodes). Goal: transcribe every audio file
in a folder/bucket accurately, resumably, with structured output. Distributed
("more folks help") is an optional Phase 2 at the end.

Reference implementation (MIT-licensed, Python, ~700 lines, single file):
https://ohgplatform.cochran.cloud/worker/community-whisper-worker.py — read
`--local` / `--batch` and the `_write_transcripts` core. Clone the *ideas*; your
stack (C++/whisper.cpp, or Python) is your call.

## 1. Core: transcribe one file well

- **Engine:** Whisper large-v3 for quality. On Apple Silicon use `mlx-whisper`
  (~8× faster than CPU); on NVIDIA use CUDA; `whisper.cpp` is a great C++ fit
  given your stack and runs well CPU-only with quantized models. openai-whisper
  `turbo` is a good CPU default if you stay in Python.
- **Decode with FFmpeg** (you already ship it). Whisper wants 16 kHz mono; let
  the lib handle resample, or pre-convert: `ffmpeg -i in -ac 1 -ar 16000 out.wav`.
- **Two settings that matter a lot for long-form audio (learned the hard way):**
  - `condition_on_previous_text=False` — with it ON, Whisper prompts each chunk
    with the previous chunk's text, so one accidental repeated phrase
    self-reinforces into minutes of looping garbage. Off = no loops.
  - Clamp the temperature fallback ladder to ~`(0.0, 0.2, 0.4)`, NOT the default
    up to 1.0 — at high temp, retries on noisy audio emit foreign-script and
    nonsense tokens instead of a usable transcript.
  - `word_timestamps=True` — gives per-word start/end. Essential if you'll ever
    cut clips/highlights or do karaoke-style captions; cheap to enable.
- **Language:** pin `language="en"` if your content is English — stops spurious
  language auto-switches mid-file.

## 2. Output: three artifacts per file

Write next to each other, named off the source stem:
- `<name>.txt` — plain transcript.
- `<name>.srt` — standard subtitles (segment-level).
- `<name>.segments.json` — the useful one: `{title, engine, word_count,
  segments:[{start, end, text, words:[{w,start,end}], avg_logprob,
  no_speech_prob, compression_ratio}]}`. Per-segment **confidence** (Whisper's
  `avg_logprob` etc.) lets you later flag likely-wrong segments for review
  instead of re-reading everything; per-word timings enable clip trimming.

## 3. Batch: point at a folder, be resumable

- Scan a directory (optionally recursive) for audio/video extensions
  (`.mp3 .wav .m4a .flac .aac .ogg .opus .mp4 .mov .mkv …`).
- **Load the model ONCE**, loop files. (Model load is the slow part — don't pay
  it per file.)
- **Resumability is non-negotiable at scale:** before transcribing, skip any
  file whose `<name>.segments.json` already exists in the output dir. A 500-file
  run WILL get interrupted; make stop-and-rerun free. (OHG's queue uses a
  `done` status for the same reason.)
- Catch per-file exceptions and continue — one corrupt file shouldn't kill a
  12-hour batch. Log what failed, tally done/skipped/failed at the end.
- Handle SIGINT cleanly (finish or abandon the current file, exit; rerun resumes).

## 4. Cloud buckets — don't bake in SDKs

For S3/GCS/Azure, **mount the bucket as a filesystem** (`s3fs`, `gcsfuse`,
`rclone mount`) and point the batch scanner at the mountpoint. Keeps the tool
dependency-free and storage-agnostic; no per-provider code. (If you'd rather
pull objects directly, a thin adapter that lists + downloads to a temp dir works
too, but mounting is simpler and streams.)

## 5. Quality knobs worth stealing (OHG specifics, optional)

- **Glossary / term-fixups:** ASR reliably garbles domain terms (OHG's classic:
  "ATEM" → "ATM" thousands of times). A post-pass find/replace against a small
  domain glossary (church/ministry names, place names, product names) massively
  improves perceived accuracy for near-zero effort. Keep person-name fixes in a
  private list.
- **QC gate instead of trust-everything:** score each transcript (words-per-
  minute in a sane band, SRT coverage vs. duration, runs of repeated chars,
  foreign-script ratio) and flag `suspect`/`fail` for a human glance. Cheap,
  catches the ~2% that went wrong.
- **"SETI" multi-pass (advanced):** run two *different* engines on the same file
  and compare — where they disagree is a likely error to flag; a term-fidelity
  vote picks the better word. Only worth it with DIVERSE engines (same engine
  twice mostly agrees, including on its mistakes). If you do this, pick the
  winner BEFORE you overwrite anything — don't let a second pass blindly replace
  the first.

## 6. Phase 2 (optional): distributed "let the congregation help"

Only if volunteers actually want in — it's a real step up in complexity. Minimal
shape that works (OHG runs exactly this):
- A small **job queue** (a DB table or even a Postgres/SQLite row-per-file with a
  status: `available | checked_out | done | failed`, a `lease_expires_at`, and
  an `attempts` counter).
- An **HTTP API** with two endpoints: `claim` (atomically hand a volunteer the
  oldest `available` job + a short lease) and `submit` (accept the transcript,
  run the QC gate, mark `done`). Everything else (download, transcribe) happens
  on the volunteer's machine — the server never sees their CPU, just results.
- A **tiny worker** each volunteer runs: `claim → download → transcribe →
  submit`, loop. Gate claims on a worker-version so you can hold back outdated
  installs when you ship a quality fix.
- **Leases + requeue** so a volunteer who quits mid-file doesn't strand it:
  expired lease → back to `available`. Cap `attempts` so a poison file fails out
  instead of looping forever.
- **Consent/privacy gate:** never process a recording someone didn't consent to
  publish. If any source is non-public, gate on an explicit "is public" flag at
  the claim step — not on a filename guess. (OHG learned this one the hard way.)

Start with Phase 1 (batch over a mounted share). Add Phase 2 only when there's
real demand — a single machine chewing a folder overnight covers most needs.
