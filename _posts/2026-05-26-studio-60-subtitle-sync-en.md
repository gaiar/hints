---
layout: post
title: "Fixing subtitle sync for Studio 60: the full story"
date: 2026-05-26 21:30:00 +0300
categories: [media, subtitles]
tags: [ffsubsync, alass, opensubtitles, plex, jekyll, vad]
lang: en
permalink: /en/2026/studio-60-subtitle-sync/
---

*[Прочитать по-русски](/2026/studio-60-subtitle-sync/)*

A walk-through of one specific task and the tools we leaned on: **ffsubsync**, **alass**, the OpenSubtitles API, and the algorithms living inside these tools (FFT cross-correlation vs. dynamic programming).

<!--more-->

## The starting point

22 episodes of **Studio 60 on the Sunset Strip** (one season, 2006-2007, NBC) in 720p HDTV. Next to each video sat a Russian subtitle file in **windows-1251** encoding, drifting out of sync with the audio. The goals:

1. Add English subtitles.
2. Fix timing for both English and Russian.
3. Convert everything to UTF-8.
4. Make Plex correctly recognize each language.

---

## Tools we used

### 1. OpenSubtitles.com REST API (v1)

The source for English subtitles. Important quirk: search by **parent_imdb_id** (`485842` for Studio 60), not by text title — text search returned "Oats Studios" instead of our show.

- Anonymous API key gives 100 downloads per day.
- We preferred official Warner Bros releases (`DVD.NonHI.en.WB`) over fan rips (`HDTV.XviD-MiNT`, etc.) — they tend to be cleaner and have fewer typos.
- Auth is just the `Api-Key: <KEY>` header, no OAuth dance.

### 2. ffsubsync (smacke/ffsubsync, v0.4.31, November 2025)

Actively maintained, the industry default. It extracts audio from the video, runs VAD (voice activity detection), then uses FFT to find **one global offset** and **one framerate scaling factor**. Perfect for the "PAL 25fps vs. NTSC 23.976fps" case.

Supports several VADs: `webrtcvad`, `auditok`, `silero` (the last requires PyTorch).

**Weakness:** only one linear transform across the whole file. If your subtitle's commercial breaks are cut in different spots than the video, ffsubsync averages and leaves drift. The author himself writes in the README: *"Handling breaks and splits in the middle of video... is left to future work"* — [open issue #31](https://github.com/smacke/ffsubsync/issues/31) since 2019.

### 3. alass (kaegi/alass, v2.0.0, 2019, unmaintained)

We switched to it once ffsubsync gave "better, but still off." Algorithmically, alass **detects split-points** and applies different offsets to different segments of the file. On S01E03 it found **4 segments** with shifts of −14.26s → −17.40s → −20.79s → −26.37s — the classic drift pattern caused by missing commercial breaks. ffsubsync can't do this.

**Weakness:** project is abandoned, but the algorithm still works in 2026 — the v2.0.0 Linux binary runs without issues.

**Picking rule:** ffsubsync is the default. Reach for alass when ffsubsync fails and you see uneven drift (especially for broadcast content with commercial breaks).

### 4. ffmpeg / ffprobe

Both sync tools use them under the hood for audio extraction. ffprobe was also useful standalone — to confirm the video framerate (`r_frame_rate=24000/1001` = NTSC 23.976) and check the .mkv had no embedded subtitle streams.

### 5. iconv

For windows-1251 → UTF-8 conversion. Though it turned out alass already writes its output in UTF-8 regardless of the input encoding, so the step was a no-op.

### 6. uv / uvx

Python package manager by Astral. Installed ffsubsync, then added torch+torchaudio (for silero VAD) via `uv tool install --with torch --with torchaudio ffsubsync`.

Side note: the machine had a broken pyright from an old pipx install — it pointed at a removed `~/miniforge3/bin/python3.10`. Fixed via `uv tool install pyright`.

---

## The trap we didn't see coming

Warner Bros DVD subtitles are structured this way: **every two-line dialogue is split into two SRT entries with identical timestamps**:

```srt
3
00:00:03,804 --> 00:00:06,291
You're one of the highest-ranking

4
00:00:03,804 --> 00:00:06,291
female executives...
```

Many players (including Plex in some clients) render both blocks simultaneously — you get visual stacking, "the end of the phrase shows on top of the beginning." This is a **source problem**, not a sync problem.

Fix is a Python script that merges adjacent blocks with identical timestamps into one multi-line block:

```python
def merge_pairs(entries):
    merged = []
    for start, end, body in entries:
        if merged and merged[-1][0] == start and merged[-1][1] == end:
            ps, pe, pb = merged[-1]
            merged[-1] = (ps, pe, pb + "\n" + body)
        else:
            merged.append((start, end, body))
    return merged
```

Across 22 files this removed 500-650 such duplicates per episode.

---

## The final pipeline

1. Search for subtitles via the OpenSubtitles API by `parent_imdb_id` + season + episode + language.
2. Download (POST `/download` with `file_id`).
3. Sync: ffsubsync first; if the result is "close, but not quite" — alass with `--speed-optimization 0 --interval 1` (max accuracy).
4. Post-process: merge entries with duplicate timestamps (for DVD sources).
5. Naming convention for Plex: `<video_basename>.en.srt`, `<video_basename>.ru.srt` (ISO 639-1 two-letter code — Plex auto-detects).
6. Originals into `_backup/` (Plex does not scan subdirectories for sidecar subtitles).

---

## The algorithms inside

### ffsubsync — FFT cross-correlation

**Step 1: discretize into binary sequences.**
Splits the video's audio track into 10 ms windows. For each window, VAD outputs **0** (silence/music) or **1** (speech). You get a binary string of length N (around 252 000 bits for a 42-minute episode).

Same with the subtitle: on the 10 ms grid, **1** where there should be text per the timestamps, **0** otherwise.

**Step 2: cross-correlation via FFT.**
To find the optimal shift between two sequences `a` (video) and `b` (subtitle), you need:

```
corr(τ) = Σ a[i] · b[i + τ]   for all τ from -max_offset to +max_offset
```

Brute force is O(N²) — billions of ops. Via FFT: `corr = IFFT(FFT(a) · conj(FFT(b)))` — O(N log N). For a typical episode ~50 million ops, seconds of CPU time.

**The peak of the correlation function is the optimal shift.** That's the "offset seconds: -8.250" in ffsubsync output.

**Step 3: framerate scaling (optional).**
Tries a handful of reasonable ratios (1.0, 23.976/25, 25/23.976, 24/23.976, etc.), recomputes the subtitle for each, picks the best cross-correlation. With `--gss` it uses **golden-section search** — a numerical method for finding the extremum of a unimodal function, converging to the optimum in log₁.₆₁₈(N) iterations without exhaustive search.

**VAD options:**

- `webrtcvad` (default) — Google's WebRTC, uses a **GMM** (Gaussian Mixture Model) trained on telephony speech. Fast, decent.
- `auditok` — energy-based detector: RMS energy above threshold = speech. Sensitive to background music (often flags it as speech).
- `silero` — a **neural net** (LSTM over MFCC features, ~1 MB of weights from the Silero company). Significantly more accurate, but requires PyTorch and has ~3 sec cold start.

**What ffsubsync structurally cannot do:** find the optimum is to find *one* τ maximizing correlation. By construction it applies that one τ to the entire file. Different τ for different sections requires a different algorithm.

### alass — dynamic programming with a split penalty

**Step 1: "rated intervals."**
Video → binary VAD sequence (like ffsubsync, but with **1 ms** intervals by default, not 10 ms). Subtitle → sequence of "has text / no text" intervals.

**Step 2: optimization problem.**
Let the subtitle have N lines. For each line `i` we choose a shift `δᵢ`. The optimal solution maximizes:

```
J = Σᵢ score(line_i, δᵢ) − P · (number of split-points)
```

where `score` measures how well the shifted line falls on speech in the video (overlap with the VAD mask), and **P** is `--split-penalty` (default 7). A "split-point" is a place where `δᵢ ≠ δᵢ₊₁`.

**Step 3: dynamic programming.**
Solved **bottom-up** via a table `DP[i][δ]` = "best total score for the first i lines if the last one is shifted by δ." The recurrence:

```
DP[i][δ] = score(i, δ) + max over δ' of (DP[i-1][δ'] − P · [δ ≠ δ'])
```

Classic DP with memory O(N · D), where D = number of candidate shifts (D = `max_offset / interval`). At `--interval 1ms` and `max_offset` of a couple minutes, D ≈ 120 000. N for a 42-minute episode is ~1300 lines. That's ~150M cells. With `--speed-optimization 1` (default) the space is compressed; with `--speed-optimization 0` (what we used) — exact search, slower but no accuracy loss.

**Step 4: recovering segments.**
After filling the table — backtrace via `argmax` gives the points where the optimal `δ` changes. Those are the "shifted block of 435 subtitles by -14.263s; shifted block of 249 subtitles by -17.400s..." lines — each block is a segment between split-points.

**Why `--split-penalty`:**

- At P → ∞ the algorithm degenerates to a single segment (behaves like ffsubsync — one global shift).
- At P → 0 it allows a different shift for each line — overfitting, lines "snap" to the nearest speech with no logic.
- Default 7 is a practical compromise. On S01E03 we got 4 segments (typical for an episode with 3-4 commercial breaks); on S01E07 — 1 segment (commercials were cut in the same places in both the subtitle source and the video).

**More:**

- `--disable-fps-guessing` turns off the built-in framerate ratio search. By default alass tries 24/23.976, 23.976/25 and a few others.
- alass uses its own VAD — an energy-based detector built on **STFT** (short-time Fourier transform), no neural nets.

### Complexity comparison

|                          | ffsubsync                | alass                                       |
|--------------------------|--------------------------|---------------------------------------------|
| **Task**                 | argmax over 1D           | argmax over a sequence, with regularization |
| **Method**               | FFT cross-correlation    | Dynamic programming                         |
| **Output parameters**    | 2 (offset + scale)       | 2N (one shift per line)                     |
| **Complexity**           | O(N log N)               | O(N · D)                                    |
| **Time per 42-min ep**   | 10-30 sec                | 10-60 sec                                   |

### Why silero VAD is worth mentioning

Sync quality is bottlenecked on VAD quality. If VAD picks up background music as "speech" — e.g., the Studio 60 musical opener "I Am the Very Model of a Modern Network TV Show" — but the subtitle is silent there, cross-correlation gets a false peak. silero is trained to distinguish speech from music and background noise, which matters for drama with a soundtrack. We didn't need it here (alass handled it), but for cases like "syncing subs to a concert recording" silero is critical.

If your friends want the academic side — the **kaegi/alass** repo explains the DP recurrence in more depth, and ffsubsync points to the classic Lewis (1995) *Fast Normalized Cross-Correlation* paper for its FFT part.

---

## Claude Code prompt (English)

```text
Sync subtitles for the video files in this directory using the best-quality
pipeline:

1. Identify the show via filename. Resolve its IMDB parent_id and find video
   files (mkv/mp4/avi) that need subtitles.

2. For each episode that lacks a subtitle in the requested language, fetch one
   from the OpenSubtitles.com REST API (https://api.opensubtitles.com/api/v1).
   Auth: header "Api-Key: <KEY>". Search /subtitles with parent_imdb_id,
   season_number, episode_number, languages=<lang>. Prefer official releases
   (e.g., "DVD.NonHI.<lang>.WB") over fan rips; fall back to the non-HI sub
   with highest download_count. Download via POST /download with file_id
   (anonymous, 100/day quota). Save as <video_base>.<lang>.unsynced.srt.

3. Detect encoding with chardet or `file`; if not UTF-8, transcode to UTF-8.

4. Sync with ffsubsync first (single-offset model, actively maintained):
       ffsubsync <video.mkv> -i <sub.unsynced.srt> -o <out.srt> --gss
   If you suspect commercial-break drift (typical for HDTV/NBC airings of older
   shows) OR the user reports the result is "better but still off", re-run with
   alass (split-aware, downloadable binary from
   github.com/kaegi/alass/releases/latest):
       alass <video.mkv> <sub.unsynced.srt> <out.srt> \
             --speed-optimization 0 --interval 1
   alass reports "shifted block of N subtitles by Xs" per segment. Multiple
   segments mean it found split-points ffsubsync would have averaged out.

5. Post-process the output:
   - If the source SRT splits multi-line dialogue into separate entries with
     IDENTICAL timestamps (common for Warner Bros DVD subs), merge consecutive
     entries that share start/end timestamps into one multi-line block.
     Otherwise Plex/VLC may stack them visually.
   - Re-number entries 1..N.
   - Ensure UTF-8 output and \n line endings.

6. Name files for Plex auto-language detection: <video_base>.<iso639-1>.srt
   (e.g., .en.srt, .ru.srt). Stash original sources in a _backup/ subdirectory
   - Plex does not scan subdirectories for sidecar subtitles, so backups won't
   show up as phantom tracks.

7. After processing, each video should have exactly one synced .srt per
   language alongside it - no .unsynced.srt, .tmp, or duplicate-suffix files
   left behind, since Plex would surface them as additional tracks.

8. Verify by reading (not just parsing) a sample of the output: check first
   entry starts at real dialogue time, scan for adjacent entries with
   identical timestamps, confirm encoding renders correctly.

Tools to install if missing: uv tool install ffsubsync (add --with torch
--with torchaudio for silero VAD); download alass-linux64 from its GitHub
releases page and chmod +x. Use ffprobe to confirm video framerate and audio
language streams before syncing.
```

---

## The takeaway

One tool solves 90% of cases; for the remaining 10% you need the right second tool. And always read the output yourself — even when both synchronizers report "success," it can turn out that the source was broken.
