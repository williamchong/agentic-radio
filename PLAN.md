# AI Radio: Plan

An open, configurable AI talk-radio system. A station is a folder of config
files run by a small set of services. The first deployment is a Cantonese
station in the style of RTHK Radio 1 and 雷霆881.

For how the system is put together, see [ARCHITECTURE.md](ARCHITECTURE.md).

Status: planning. Nothing is built yet. Last updated 2026-10-02.

## Goals

- A talk-led station: news, weather, themed talk shows, book and article
  reading, with music as filler.
- Streams 24/7 to YouTube Live and a web stream, and publishes shows as
  podcasts (which is how they reach Spotify; Spotify takes no live streams).
- Other people can clone the repo, edit config, add API keys and run their own
  station in any language, on one machine with Docker Compose or on a
  serverless platform.

## Non-goals

- Phone-ins and guest interviews (no invented callers or simulated real people).
- Political opinion and personality comedy.
- Live sport or racing commentary.
- Commercial pop music.

## Programming

Modelled on 雷霆881's short themed slots, which suit generation better than
long host-led blocks: one topic and one source set per slot.

| Show | Format | Sources |
|---|---|---|
| News and weather on the hour | bulletin | RTHK RSS, government press releases, HKO API |
| Morning and evening news magazine | two-host talk | Same as bulletin, plus world feeds |
| World digest | two-host talk | VOA Chinese, BBC中文, DW中文 |
| Tech half-hour | two-host talk | Hacker News API, arXiv, tech feeds |
| Finance wrap | digest | HKMA API, HKEX filings (no stock tips) |
| Knowledge chat (講東講西 style) | two-host talk | Wikipedia, 粵文維基百科 |
| Book and article reading | reading | 維基文庫, Project Gutenberg, ctext.org |
| Overnight | music block + repeats | CC0 classical recordings, earlier segments |

Later, as a stretch: original serial drama or storytelling, and a show where
hosts respond to real listener messages from YouTube live chat.

Target is 6 to 8 hours of fresh speech a day, with repeats and music filling
the rest, as real stations do overnight.

## Findings so far (checked 2026-10-02)

### Voice: ElevenLabs

- `eleven_v4` and `eleven_v4_turbo` list Cantonese (yue). v3 and older models
  do not.
- Text to Dialogue works on v4: a list of `{text, voice_id}` lines returns one
  mixed audio file. Keep each request at or under about 2,000 characters,
  roughly 7 minutes of Cantonese.
- Single-voice requests on v4 take up to 10,000 characters.
- List price is $0.08 per 1,000 characters for v4 and $0.04 for v4 Turbo.
  Seven hours of fresh speech a day is about 3.5 million characters a month.
- MiniMax is the cheaper comparison and has not been checked yet.

### Sources

- HKO open data API returns the forecast, outlook and warnings in Chinese as
  JSON.
- RTHK local news RSS works, with 120 to 280 character summaries per item.
- Yahoo HK News RSS works but is an aggregator of other publishers, gives
  summaries only and mixes in lifestyle content. Use it only as a signal of
  which stories are big.
- Yahoo Finance HK has no official API; do not depend on scraping it.

### Music

A public-domain composition does not make the recording public domain. Use
recordings that are themselves CC0 or public domain (Musopen, Open Goldberg
Variations, filtered Internet Archive items), avoid non-commercial licences,
and keep a manifest of each track's source URL and licence for disputing false
Content ID claims.

### Other Hong Kong stations

Only three licensed broadcasters remain: RTHK, Commercial Radio and Metro.
None of the current weekday schedules checked has a dedicated book-reading
slot, so that strand would distinguish this station.

## Build order

1. **Test render**: a 2-minute two-host bulletin from today's HKO forecast and
   RTHK headlines, on ElevenLabs v4.
2. **Bulletin format end to end** in Docker Compose: fetch, script, voice, mix,
   upload as HLS chunks.
3. **Playout**: the live playlist endpoint with music fallback, playable in a
   browser.
4. **Planner and two-host talk format**, driven by `schedule.yaml` and the
   tick.
5. **Reading format, admin page, podcast feed.**
6. **YouTube relay** as an optional Compose service.
7. **Packaging**: example station config, setup docs, and a serverless
   deployment of the same images.

## Open questions

- Does v4 keep a voice stable across stitched chunks, and read Cantonese
  numbers, dates and mixed English correctly?
- Is the 口語 script quality good enough? This needs judging by ear.
- What are the reuse terms of each news source, including Yahoo's RSS terms?
- Which finance data source replaces Yahoo Finance?
- What does the LLM cost per hour of content?
- Are the untested sources (data.gov.hk feeds, HKMA, HKEX, world news RSS,
  維基文庫) usable in practice?
