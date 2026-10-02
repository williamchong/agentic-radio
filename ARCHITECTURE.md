# AI Radio: Architecture

How the system is put together. For goals, programming, research findings and
build order, see [PLAN.md](PLAN.md).

Status: design only. Nothing is built yet. Last updated 2026-10-02.

## Overview

A station is a folder of config files run by three containers. Someone clones
the repo, edits the YAML, adds API keys and runs `docker compose up`.

```
                 station/ (YAML config, music, jingles)
                              |
+-----------------------------v------------------------------+
| station app (one codebase, three roles)                    |
|                                                            |
|  Planner --> Producers --> Segment store --> "next?" API   |
|  (schedule    (fetch -> script -> voice -> mix)  + admin UI|
|   -> jobs)                                   + podcast RSS |
+-----------------------------+------------------------------+
                              | HTTP: "what plays next?"
                     +--------v--------+
                     |   Liquidsoap    |  falls back to repeats, then music
                     +---+---------+---+
                         |         |
                    Icecast     RTMP -> YouTube Live
```

| Container | Job |
|---|---|
| Station app | Plans the schedule, generates segments, serves the next-item endpoint, admin page and podcast feed |
| Liquidsoap | Plays audio continuously, asks the app what is next, handles crossfades and fallback |
| Icecast | Serves the web listener stream |

Storage is one SQLite file plus a folder of audio on a volume. Postgres, Redis
and a separate job queue are deliberately left out until a single station
outgrows this.

## Station app

One codebase with three roles (planner, producers, server), which can be split
into separate containers later without changing the config format.

- **Planner** reads the weekly grid and creates a job for each upcoming slot,
  with a deadline some lead time before air (about 45 minutes for talk, 5 for
  news).
- **Producers** run each job through four steps: gather sources, write the
  script, voice it, then mix in beds and jingles and normalise loudness. Each
  finished segment goes into the segment store as an audio file plus metadata:
  show, air window, sources used, transcript, cost.
- **Server** reads the segment store and exposes three things:
  - the next-item endpoint, which returns the segment due now. If it is not
    ready, Liquidsoap moves down the fallback ladder: a repeat, then music,
    then an emergency loop.
  - the admin page.
  - the podcast RSS feed for shows marked `podcast: true`.

### Design principles

- **The next-item endpoint is the whole contract between generation and
  playout.** Liquidsoap knows nothing about LLMs, and the app never touches a
  live audio stream, so either side can fail without taking the other off air.
- **A segment is a finished file with an air window.** Repeats, podcasts and
  previews are all different ways of reading the same store.
- **Content is generated ahead of air, not live.** "Real-time" news is a batch
  job that runs shortly before each bulletin.
- **Talk scripts are written as scenes.** The voice provider caps the size of
  a dialogue request (see the findings in PLAN.md), so each scene is one
  request, and scene joins are natural points for a jingle or music bed.

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

Secrets (LLM key, voice key, YouTube stream key) live in an `.env` file, not in
the station folder.

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
- **Kill switch**: the admin page lists upcoming segments with a preview and a
  remove button.

## Stack

Beyond the containers and storage described in the overview:

- Station app in TypeScript on Node.
- ffmpeg for mixing and loudness normalisation.

## Hosting

The core cannot run on serverless platforms, because playout is an always-on
process holding open connections.

| Part | Where | Why |
|---|---|---|
| Station app, Liquidsoap, Icecast | One VPS, about 2 vCPU / 4 GB, in Singapore, Tokyo or Hong Kong | Always-on process, a disk, steady outbound streaming; the YouTube video encode is the main CPU load |
| Script writing | LLM API | Cost scales with hours of fresh content |
| Voice | ElevenLabs or MiniMax API | The largest running cost |
| Audio archive and podcast files | Cloudflare R2 | No egress fees |
| Podcast to Spotify | The app's own RSS feed, submitted once | No podcast host needed |
| Video listeners | YouTube Live | Free distribution at any audience size |
| Monitoring | External uptime check on the stream URL, plus error tracking for the app | Dead air is the failure that matters most |

### Bandwidth

Direct Icecast listeners are the cost that scales with audience: 100 listeners
at 128 kbps around the clock is roughly 4 TB a month. Two mitigations:

- Send most listeners to YouTube and treat Icecast as secondary.
- If the web audience grows, switch the web stream to HLS: Liquidsoap writes
  audio chunks to R2 and a CDN serves them, so listener count stops affecting
  the server.

Server and LLM costs are fixed or scale with content hours, not audience, which
is why the budget cap matters more than the choice of cloud.
