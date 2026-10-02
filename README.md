# Batch Audio Transcriber — Design Spec

A build spec you can hand to a coding agent (or follow yourself) to transcribe a
whole library of audio — a local share, a NAS, or a cloud bucket — accurately,
resumably, and with structured output you can actually build on.

It distills the design and the non-obvious lessons from a community
transcription pipeline that processed **~2,300 hours** of long-form talk audio.
The goal isn't "run Whisper" — that's one line. It's everything *around* Whisper
that makes a large batch finish cleanly, read accurately, and produce output a
downstream tool can use.

> **Stack-agnostic.** A reference implementation exists in Python (single file,
> MIT), but the ideas port directly to `whisper.cpp` (great for a C++/FFmpeg
> codebase), faster-whisper, or any Whisper binding. Build it however fits.

---

## What you get per file

```
my-recording.mp3
   │
   ▼  (transcribe)
   ├── my-recording.txt            plain transcript
   ├── my-recording.srt            standard subtitles (segment-level)
   └── my-recording.segments.json  the useful one ▼
```

```jsonc
{
  "title": "my-recording",
  "engine": "whisper-large-v3",
  "word_count": 8421,
  "segments": [
    {
      "start": 12.34, "end": 15.90,
      "text": "welcome everybody to the show",
      "words": [                        // per-word timing → clip/caption trimming
        { "w": "welcome",   "start": 12.34, "end": 12.80 },
        { "w": "everybody", "start": 12.81, "end": 13.40 }
      ],
      "avg_logprob": -0.21,             // confidence → flag likely-wrong spans
      "no_speech_prob": 0.01,
      "compression_ratio": 1.34
    }
  ]
}
```

Three artifacts, one pass. The JSON is what makes this more than a subtitle
generator: **per-word timings** let you cut clips or karaoke captions to the
exact word, and **per-segment confidence** lets you later flag the handful of
shaky segments for review instead of re-reading the whole thing.

---

## The pipeline

```
          ┌─────────────────────── BATCH LOOP (model loaded once) ───────────────────────┐
          │                                                                               │
  folder/ │   for each media file:                                                        │
  bucket ─┼─▶ already done? ──yes──▶ skip (resumable)                                      │
  (mount) │        │ no                                                                    │
          │        ▼                                                                       │
          │   ┌─────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────────┐   │
          │   │ FFmpeg  │──▶│   Whisper     │──▶│  glossary +    │──▶│ write txt / srt / │  │
          │   │ decode  │   │ (tuned flags) │   │  QC gate       │   │ segments.json     │  │
          │   │ 16k mono│   │ word+conf     │   │  (optional)    │   │                   │  │
          │   └─────────┘   └──────────────┘   └───────────────┘   └──────────────────┘   │
          │        │ on error: log + continue (one bad file can't kill the run)            │
          └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Transcribe one file well

**Engine.** Whisper `large-v3` for quality.

| Platform | Recommended backend | Notes |
|---|---|---|
| Apple Silicon | `mlx-whisper` | ~8× faster than CPU; the sweet spot on a Mac |
| NVIDIA GPU | faster-whisper (CTranslate2) or openai-whisper + CUDA | big speedup |
| CPU-only / C++ | `whisper.cpp` with a quantized model | no Python, runs anywhere |
| CPU-only / Python | openai-whisper `turbo` | fine default, slower |

**Decode with FFmpeg.** Whisper wants 16 kHz mono. Let the library handle it, or
normalize up front: `ffmpeg -i in -ac 1 -ar 16000 out.wav`. FFmpeg also means you
accept anything — mp3, wav, m4a, flac, opus, and video containers.

**Four settings that matter for long-form audio:**

1. **`condition_on_previous_text = False`.** With it on, each chunk is prompted
   with the previous chunk's text, so a single accidental repeated phrase
   self-reinforces into *minutes* of looping output. Off = no loops. This is the
   single most important flag for talks, sermons, podcasts, meetings.
2. **Clamp the temperature fallback to ~`(0.0, 0.2, 0.4)`.** The default ladder
   climbs to 1.0, where retries on noisy audio emit foreign-script and nonsense
   tokens rather than a usable transcript. Capping it keeps failures graceful.
3. **`word_timestamps = True`.** Cheap, and the only way to get the per-word
   timing that makes clipping and precise captions possible.
4. **Pin the language** (e.g. `language="en"`) when you know it — prevents
   spurious mid-file language switches.

---

## 2. Structured output

Write all three artifacts next to each other, named off the source file's stem
(see the JSON shape above). Keep the sidecar data (words, confidence) **index-
aligned** to the segments so anything downstream can re-join them without
guessing. One self-describing JSON per file beats a pile of parallel files.

---

## 3. Batch: point it at a folder, make it resumable

```
transcribe --batch /mnt/recordings --recursive --out ./transcripts
```

- **Scan** the directory (optionally recursive) for audio/video extensions.
- **Load the model once**, then loop. Model load is the slow part — never pay it
  per file.
- **Resumability is non-negotiable at scale.** Before transcribing, skip any file
  whose output already exists. A 500-file run *will* be interrupted — a reboot, a
  closed laptop, a Ctrl-C. Make stop-and-rerun free and idempotent.
- **Isolate failures.** Wrap each file in try/except and continue; one corrupt or
  zero-length file shouldn't end a 12-hour run. Tally `done / skipped / failed`
  at the end and log which files failed.
- **Handle SIGINT cleanly** so Ctrl-C stops between files and a rerun resumes.

---

## 4. Cloud buckets — mount, don't integrate

Skip per-provider SDKs. **Mount the bucket as a filesystem** and point the batch
scanner at the mountpoint:

| Provider | Mount with |
|---|---|
| Amazon S3 | `s3fs` or `rclone mount` |
| Google Cloud Storage | `gcsfuse` or `rclone mount` |
| Azure Blob | `blobfuse2` or `rclone mount` |
| Anything else | `rclone mount` |

```
gcsfuse my-church-recordings /mnt/recordings
transcribe --batch /mnt/recordings --recursive --out ./transcripts
```

This keeps the tool storage-agnostic and dependency-free; writes stream back to
the bucket if your output dir is on the mount too. (If you prefer to pull objects
directly, a thin "list + download to temp dir" adapter works — but mounting is
simpler and handles streaming and retries for you.)

---

## 5. Accuracy knobs worth adding

- **Domain glossary.** ASR reliably garbles domain-specific terms — product
  names, place names, people's names, jargon. A small find/replace pass against a
  glossary of your known terms (correct spelling + the common mis-hearings)
  dramatically improves perceived accuracy for almost no effort. Keep personal
  names in a private list if the output is published.
- **QC gate instead of blind trust.** Score each transcript with cheap heuristics
  — words-per-minute in a sane band, subtitle coverage vs. audio duration, runs
  of repeated characters, foreign-script ratio — and flag `suspect` / `fail` for
  a human glance. This catches the ~2% that went wrong without reviewing the 98%
  that didn't.
- **Multi-engine cross-check (advanced).** Run two *different* engines on the same
  file and compare; where they disagree is a likely error worth flagging, and a
  term-fidelity vote can pick the better word. Worth it only with **diverse**
  engines — the same engine run twice mostly agrees with itself, mistakes
  included. If you do this, decide the winner *before* overwriting anything.

---

## 6. Optional Phase 2 — distributed ("let more people help")

A single machine chewing a folder overnight covers most needs. Add this only
when volunteers genuinely want to pitch in — it's a real step up in complexity.
The minimal shape that works:

```
          ┌──────────────┐        claim (lease)        ┌─────────────────────┐
          │              │ ◀───────────────────────────│  Volunteer worker A │
          │   Job queue  │                             │  download→transcribe│
          │  + tiny API  │ ────────── submit ─────────▶│  →submit, loop      │
          │              │                             └─────────────────────┘
          │ status:      │        claim (lease)        ┌─────────────────────┐
          │ available    │ ◀───────────────────────────│  Volunteer worker B │
          │ checked_out  │ ────────── submit ─────────▶└─────────────────────┘
          │ done | failed│
          └──────────────┘   lease expires → back to "available" (nobody strands a file)
```

- **Job queue:** a DB table (even SQLite), one row per file, with
  `status (available | checked_out | done | failed)`, `lease_expires_at`, and an
  `attempts` counter.
- **Two API endpoints:** `claim` (atomically hand out the oldest `available` job
  + a short lease) and `submit` (accept the result, run the QC gate, mark
  `done`). The server never touches the audio or the CPU — volunteers do the work
  locally and send back only the transcript.
- **A tiny worker** each volunteer runs: `claim → download → transcribe →
  submit`, loop. Version-gate claims so you can hold back outdated installs when
  you ship a quality fix.
- **Leases + requeue:** if a volunteer quits mid-file, its lease expires and the
  job returns to `available`. Cap `attempts` so a poison file fails out instead
  of looping forever.
- **Consent gate:** only process recordings that are cleared to publish. Gate on
  an explicit "is this allowed" flag at claim time — not on a filename guess —
  so a private recording can never slip into a public transcript.

---

## Build order

1. **Phase 1** — one-file transcribe with the tuned flags + the three artifacts.
2. **Batch** — folder scan, load-once, resumable skip, failure isolation.
3. **Mount** a bucket and point Phase 2's scanner at it — done for most use cases.
4. **Phase 2** (queue + workers) only when there's real demand for distribution.

Start small, get Phase 1 right, and most of the value is already yours.

---

*MIT licensed — use it, fork it, build on it.*
