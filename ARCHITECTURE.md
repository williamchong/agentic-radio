# Agentic Radio: Architecture

How the system is put together. For goals, programming, research findings and
build order, see [PLAN.md](PLAN.md).

Status: design only. Nothing is built yet. Last updated 2026-10-02.

## Overview

A station is a folder of config files run by stateless containers. The same
images run under Docker Compose on any machine, or on a scale-to-zero container
platform, with no code changes. Someone clones the repo, edits the YAML, adds
API keys and runs `docker compose up`.

```
        station/ (YAML config, music, jingles)
                      |
   clock --/tick-->  app (stateless HTTP handlers)
                      |   planner -> producers -> HLS chunks
                      |
          +-----------+-----------+
          |                       |
      database              audio storage
      (Postgres)               (S3 API)
                                  |
   listeners <-- CDN <-- /live.m3u8 + chunks
                                  |
                 YouTube relay (optional, always on)
                 reads the HLS stream, pushes RTMP
```

Nothing runs between requests. The live stream is an HLS playlist computed from
the schedule and the clock, so there is no always-on player. The only
always-on part is the optional relay that YouTube Live requires.

## Services

Each role has a Compose form and a serverless form behind a standard interface,
and the app does not know which one it is talking to.

| Role | In Docker Compose | On serverless |
|---|---|---|
| App (stateless HTTP handlers) | One container | Scale-to-zero container service (Cloud Run, Fly Machines, Lambda container images) |
| Audio storage (S3 API) | MinIO container | R2 or S3 |
| Database (Postgres) | Postgres container | Neon or Supabase |
| Clock | Cron container that calls `/tick` every minute | Cloud scheduler calling the same URL |
| YouTube relay (optional) | ffmpeg container, always on | Stays on a VPS or always-on machine |

The target is serverless containers, not Workers-style functions, which cannot
run under Compose without emulators and cannot run ffmpeg.

## App endpoints

- **`/tick`** is the planner. It reads the weekly grid, creates a job for each
  upcoming slot with a deadline some lead time before air (about 45 minutes for
  talk, 5 for news), and starts any jobs that are due.
- **`/jobs/run`** produces one segment: gather sources, write the script, voice
  it, mix in beds and jingles, normalise loudness, then upload the result as
  HLS chunks with metadata (show, air window, sources used, transcript, cost).
- **`/live.m3u8`** is the live playlist. It works out from the schedule and the
  current time which chunks are on air. If a segment is not ready, it moves
  down the fallback ladder: a repeat, then music, then an emergency loop.
- **`/admin`** lists upcoming segments with a preview and a remove button.
- **`/feed.xml`** is the podcast RSS feed for shows marked `podcast: true`.

Jobs are rows in the database driven by the tick, so there is no queue service.

### Design principles

- **The app is a pure function of the request plus database state.** Making
  the clock an external caller of `/tick` is what removes the always-on
  process.
- **A segment is a finished set of chunks with an air window.** Live playout,
  repeats, podcasts and previews are all different ways of reading the same
  store.
- **Content is generated ahead of air, not live.** "Real-time" news is a batch
  job that runs shortly before each bulletin.
- **Nothing mixes live.** Each segment gets its fades and loudness set when it
  is made, and every segment is encoded identically so chunks can follow each
  other in one playlist.
- **Talk scripts are written as scenes, one job per scene.** The voice provider
  caps the size of a dialogue request (see the findings in PLAN.md), and a
  scene-sized job also fits inside serverless request time limits (15 minutes
  on Lambda, 60 on Cloud Run).

## Config

```
station/
  station.yaml      # name, language preset, timezone, outputs, daily budget
  schedule.yaml     # weekly grid + the clock within each hour
  hosts/            # one file per presenter: persona, speech style, voice
  shows/            # one file per show: format, hosts, sources, length
  sources.yaml      # RSS feeds, APIs, book and article folders
  music/  jingles/
```

A show is a format plus settings, so adding one needs no code:

```yaml
# shows/tech-half-hour.yaml
format: two-host-talk
hosts: [ah-ming, ka-yan]
duration: 25m
sources: [tech-feeds]
lead_time: 45m
podcast: true
```

Secrets (LLM key, voice key, YouTube stream key) and the storage and database
URLs live in an `.env` file, not in the station folder. Switching between
Compose and serverless is a change to those URLs.

## Extension points

| Plugin type | Ships with | Others can add |
|---|---|---|
| Formats | bulletin, two-host talk, reading, digest, music block | drama, quiz, listener-message show |
| Providers | one LLM, ElevenLabs and MiniMax for voice | Azure, local models |
| Sources | RSS, HKO weather, local text files | market data, YouTube chat |

**Language is a preset.** Cantonese is a bundle of script-style rules (口語,
number and date normalisation) and default voices, so the same code can run a
station in another language.

**Sources come in two tiers**, kept as separate types in `sources.yaml`:

- Structured data (weather, markets, traffic), which can be read nearly
  verbatim.
- News prose, which must be rewritten, attributed and passed through the
  grounding check.

## Guardrails

- **Grounding check** on news formats: every claim in the script must appear in
  the source text, or the segment is dropped.
- **Budget cap**: a daily voice spend limit in `station.yaml`; when hit, the
  planner schedules repeats.
- **AI disclosure**: a required station ident stating the hosts are
  AI-generated.
- **Kill switch**: removing a segment in the admin page takes it out of the
  playlist.

## Stack

Beyond the services above, the app is TypeScript on Node, with ffmpeg in the
image for mixing, loudness normalisation and HLS chunking.

## Deployment

| Mode | What runs where | Suits |
|---|---|---|
| Single machine | Everything in Compose on one VPS or home server, including the relay | Self-hosters, development, a station that needs YouTube Live |
| Serverless | App on a scale-to-zero container service, managed Postgres, R2, a cloud scheduler | Near-zero idle cost and a web audience of any size |
| Hybrid | Serverless as above, plus a small always-on machine running only the relay | Serverless with YouTube Live |

Other parts are the same in every mode:

| Part | Where | Why |
|---|---|---|
| Script writing | LLM API | Cost scales with hours of fresh content |
| Voice | ElevenLabs or MiniMax API | The largest running cost |
| Podcast to Spotify | The app's own RSS feed, submitted once | No podcast host needed |
| Monitoring | External check that `/live.m3u8` is advancing, plus error tracking for the app | Dead air is the failure that matters most |

### Trade-offs of HLS-only playout

- Listeners are 10 to 30 seconds behind the clock, which is acceptable for
  radio.
- There are no live crossfades; fades are baked into each segment.
- The first playlist request after idle is slow on serverless; a CDN in front
  hides most of it.
- Web listeners are served by the CDN, so audience size does not load the app.
  Running costs scale with hours of fresh content, not audience, which is why
  the budget cap matters more than the choice of cloud.
