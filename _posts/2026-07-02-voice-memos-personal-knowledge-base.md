---
title: "I Got Tired of My Best Notes Being Trapped as Audio Files"
date: 2026-07-02
tags: [ai, automation, macos, whisper, claude-code, personal-knowledge-base, vibe-coding]
excerpt: "I record voice memos on my Apple Watch throughout the day. They used to just sit there as audio. Here's the local pipeline I built to turn them into searchable text, and the unglamorous macOS problems that turned out to be the real engineering."
image: /assets/images/voice-memos-hero.png
---

![Voice memo on an Apple Watch turning into transcribed notes on a laptop](/assets/images/voice-memos-hero.png)

I talk to my Apple Watch more than I'd like to admit. Walking to a meeting, driving, in the middle of a trip-planning spiral at 11pm, I tap the mic and just say the thing before I lose it. It syncs to Apple's native Voice Memos app on my Mac a few seconds later, and that's where it used to end. A growing pile of `.m4a` files named "Recording 47," "Recording 48," write-only, unsearchable, and mostly forgotten by the time I'd need them again.

The problem was never capturing the thought. Apple solved that years ago. The problem was everything after: remembering the memo existed, finding the right one three weeks later, and turning unstructured audio into something I could actually search or act on.

So I built a local macOS automation that watches my Voice Memos, transcribes them, and summarizes them, without touching a single Apple feature or asking me to export anything by hand. The code is [open source on GitHub](https://github.com/jonzarecki/macos-voice-memos-transcriber). This is what building it actually taught me.

## Why not just use what's already there

Before writing a line of code I checked the obvious options, because most of my memos are in Hebrew and that constraint kills more tools than you'd expect.

**Apple's own on-device transcription** in Voice Memos is genuinely good, but it ships with a language whitelist: English, Spanish, Portuguese, French, German, Italian, Japanese, Korean, Chinese, Arabic, Russian, Turkish. Hebrew isn't on it. Recordings in an unsupported language simply never get a Transcript tab. Dead end, immediately.

**Cloud transcription apps** (Otter, Notta, and the rest) support far more languages, but the audio leaves the device to get there. For notes that include planning conversations, work reflections, and other things I'd rather not have leave the device at all, that's a trade I wasn't willing to make.

**Generic Whisper**, run locally, was closer. But the stock multilingual Whisper checkpoints hallucinate badly on Hebrew audio, they'll confidently generate fluent, completely wrong sentences instead of failing loudly. Not usable for anything I'd want to trust later.

What actually worked: `mlx_whisper` running on Apple Silicon, using `ivrit-ai`'s Hebrew fine-tune of Whisper large-v3. Local, fast enough on an M-series Mac, and dramatically more accurate on Hebrew than anything in the general-purpose set.

Transcription solved maybe 20% of the actual problem.

## From "transcribe this file" to "watch my life"

The first version of this was just me, manually. Export a memo, run `mlx_whisper` against it, read the text. That's fine for one file. It's useless as a habit, because the moment something requires a manual step, I stop doing it after about a week.

So the real project became: detect new memos automatically, transcribe them, summarize them, and tell me when it's done, all without me thinking about it.

<img src="/assets/images/architecture.svg" alt="Architecture diagram: Apple Watch to iCloud to Voice Memos to CloudRecordings.db detection to wrapper app to MLX Whisper and Claude Code to state and notification" />

The pipeline, end to end:

- A `launchd` LaunchAgent runs every 60 seconds and on any change to the Voice Memos recordings folder.
- It reads Apple's own metadata database (`CloudRecordings.db`), not the folder contents directly.
- New or unprocessed recordings get transcribed locally with MLX Whisper. Audio never leaves the machine.
- The transcript, not the audio, is summarized with `claude -p` into a short Markdown file: summary, action items, notable details. That's the one step where text can leave the machine, depending on how Claude Code is configured.
- A durable local state file tracks what's been processed, so nothing gets transcribed twice.
- I get a macOS notification when something is deferred, fails, or finishes.

None of that is exotic. What made it work reliably was the boring 80% underneath.

## The part that was actually hard: it wasn't the model

Every one of the interesting problems here turned out to be a macOS problem, not an AI problem.

**Folder watching lies to you.** Apple's Voice Memos folder contains more than recordings, waveform sidecars, composition folders, and files that exist but have no usable audio stream. The reliable signal is the `CloudRecordings.db` database, specifically fields like `ZPATH`, `ZDATE`, `ZDURATION`, and `ZUNIQUEID`. Query the database, don't guess from file names.

**`.qta` is a format you'll meet eventually.** Some native recordings aren't `.m4a`, they're QuickTime Audio (`.qta`). A few of those have no readable audio stream at all and need to be marked unavailable rather than retried forever.

**Terminal access and `launchd` access are different macOS privacy worlds.** A command that reads Voice Memos data perfectly well from Terminal can fail with `authorization denied` the moment it runs under a LaunchAgent. Full Disk Access is granted to an identity, and a raw Python interpreter invoked by `launchd` isn't a stable identity you'd want to grant broad disk access to anyway. The fix was a tiny compiled wrapper app, `VoiceMemoTranscriberRunner.app`, that `launchd` opens via `open -g -W`. Grant Full Disk Access to that one app bundle, and everything downstream inherits it cleanly.

**Claude auth under `launchd` isn't the same as Claude auth in your shell.** Background jobs don't inherit your interactive shell's environment, so Vertex credentials live in an isolated env file the worker loads explicitly. When auth fails, the worker backs off for 12 hours and sends one notification instead of retrying every 60 seconds and spamming me.

**Background AI work needs to respect battery, or "useful automation" becomes "why is my laptop hot."** Detection always runs. Transcription and summarization, the expensive parts, only run on AC power, except for short recordings when the battery is comfortably charged. Otherwise the job is deferred and retried once I plug in.

None of these are hard problems individually. They're just the kind of thing that never shows up in a demo, and that collectively decide whether an automation is something you trust or something you eventually disable.

## What it's actually good for

A few months in, the categories that get the most value aren't glamorous: trip-planning notes that turn into a route and logistics summary instead of forty minutes of audio I'd never replay, meeting and planning conversations that become searchable text with action items already pulled out, and the backlog of old "Recording 42"-style memos that finally became useful once they had names and content instead of just numbers.

The quieter win is that I stopped losing thoughts. A memo isn't fully captured for me anymore the moment I record it, it's captured the moment it becomes text I can search.

## The repo is open

I extracted the automation into a standalone project: [`macos-voice-memos-transcriber`](https://github.com/jonzarecki/macos-voice-memos-transcriber). It's genericized so it doesn't assume Hebrew, swap the model and language flag and it works for whatever you speak. Docs cover the install path, the Full Disk Access setup, and the failure modes above in more detail than most people will ever need.

It's not a startup idea and it's not trying to be a product. It's the kind of automation I actually trust: not a new surface area to manage, just a quiet bridge between two tools I was already using every day.
