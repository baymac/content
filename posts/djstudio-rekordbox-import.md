---
title: 'Importing DJ.Studio mixes into rekordbox'
date: '2026-04-25'
tags: dj, music, python, rekordbox, djstudio, beatport
ai-gen: true
---

I plan my sets in DJ.Studio because the timeline view is the fastest way to lay out a 60-minute mix. I play out on CDJs from rekordbox because that's what's in the booth. There's no export between them. So I wrote two scripts that talk to each side directly and meet in the middle.

Repo: [https://github.com/baymac/djstudio-export](https://github.com/baymac/djstudio-export). Two scripts. One JSON file in between.

## Why this setup

I want to listen. DJ.Studio lets me do exactly that. I can hear my mix, my transitions, my effects with clicks not analog movements. You should still practice on real gear. There are too many tracks dropping every week to practice everything. Sometimes you need speed not practice.

So I hone the mix in DJ.Studio. Then I export the hot cues for the transitions into rekordbox. The gig version matches what I rehearsed.

The bit that makes this actually useful: it works with Beatport streaming. DJ.Studio's native export does not.

## My workflow

Simple one.

1. Build a playlist in Beatport by genre. Collect the latest releases.
2. Import the playlist into DJ.Studio.
3. Do harmonic mixing. Build energy through the set. Experiment with transitions.
4. Export to rekordbox and practice.
5. Loop until the set is locked.
6. Purchase the tracks from Beatport.

Streaming lets you audition 50 tracks without committing to 50 tracks. This importer is what makes step 4 actually work.

## What it does

Takes a mix you've built in DJ.Studio and produces a rekordbox playlist with:

- Every track in the right order, created if it doesn't already exist
- Transition effect names dropped into each track's `Commnt` field for reference
- Hot cues `A` through `H` set up so you can perform the planned mix on a CDJ

The "create if missing" part matters because almost every track I use is from Beatport streaming. Those don't import via the rekordbox XML route. I tried that first. Silent drop.

## Requirements

- Python 3
- [pyrekordbox](https://github.com/dylanljones/pyrekordbox)
- DJ.Studio installed locally (the script reads its on-disk database)
- Rekordbox closed before any write. The DB is encrypted SQLite and gets locked while the app is open.

## How it works

`get_mix_info.py` reads two places on disk:

- `~/Music/DJ.Studio/Database/projects-table/{uuid}` for the mix project (track order, transitions, effects)
- `~/Music/DJ.Studio/Database/audio-library-table/{hash_prefix}/{library_key}` for track metadata (BPM, key, cue points)

It spits out a JSON file. You may take a look at the [output schema in the README](https://github.com/baymac/djstudio-export).

`import_to_rekordbox.py` reads that JSON and writes into `master.db` via pyrekordbox.

## Interesting things implemented

**Beatport streaming tracks** — rekordbox stores these as `FileType=20` with `FolderPath = /v4/catalog/tracks/{BEATPORT_ID}/`. Match by Beatport ID. Create the missing ones with the same shape so rekordbox treats them as first-class streaming entries.

**BPM is an integer * 100** — 129 BPM lives in the DB as `12900`. Spent more time than I'd like to admit before I noticed.

**`Commnt` not `Comment`** — rekordbox's field for the comment column is misspelled. I keep it that way because pyrekordbox does. e.g. `track.Commnt = "Trans out: AE_CrossFade"`.

**Two-pass cue writing** — the first version computed cue positions from DJ.Studio's beat data and wrote them straight into rekordbox. They drifted. DJ.Studio and rekordbox analyze tracks independently so their beat grids don't line up. Now Pass 1 creates the tracks. You open rekordbox so it can analyze them. Pass 2 reads the resulting `ANLZ` files (the `PQTZ` tag has the beat grid in ms) then snaps each cue to the nearest analyzed beat using `bisect_left`. `--no-snap` is there if you want raw positions back.

**Hot cue layout A-H** — first track gets play-start plus prep / transition / bass-swap / end for the outgoing transition. Middle tracks get incoming on `B`-`E`. Outgoing lives on `A` plus `F`-`H`. Bass swap cue only fires when `AE_Bass_Swap` or `AE_Bass_SwapFade` or `AE_Bass_CrossFade` is in the effects list.

## Caveats

- Rekordbox must be closed. If you forget, pyrekordbox will tell you.
- Pass 2 only does its job if you've actually let rekordbox analyze the playlist. If it can't find an `ANLZ` file for a track it falls back to raw positions then tags the row `[unsnapped]`.
- Cue IDs are `uuid4()` strings because `DjmdCue` rows want strings. If you go looking for `generate_unused_id` it won't help here.

## Future work

- A single-command "do everything" wrapper that handles the open-rekordbox-to-analyze pause by watching the `ANLZ` directory.
- Smarter prep-distance picker. Right now it's `8` / `16` / `32` beats based on transition length. Could be tuned per genre.
- An undo path. Currently the only undo is restoring the rekordbox DB backup pyrekordbox writes for you.

## Resources

- [Repo](https://github.com/baymac/djstudio-export)
- [pyrekordbox](https://github.com/dylanljones/pyrekordbox)
- [DJ.Studio](https://dj.studio/)
