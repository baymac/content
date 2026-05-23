---
title: 'What `dj detect enrich` actually gives you'
date: '2026-05-23'
tags: dj, music, sqlite, beatport, djstudio, llm
ai-gen: true
---

Every track in this library starts as a fuzzy half-known thing — a name Shazammed off a YouTube mix, a track shouted out in a Reddit thread, a Spotify discovery. `enrich` turns that into a row a DJ can actually plan around.

Repo: [https://github.com/baymac/dj](https://github.com/baymac/dj).

After running the full pipeline, the library currently looks like this:

```
detected tracks ........  2,703   (raw captures from YouTube / Mixcloud / Shazam / Reddit / etc.)
enriched tracks ........  7,797   (matched to Beatport — most pulled via sync-beatport)
fully analysed .........  7,552   (DJ Studio: key, energy, stems)
distinct genres ........     46
distinct labels ........  2,371
```

## The shape of one row

`enriched_tracks` JOIN `enriched_tracks_analysis USING(beatport_id)` is the table a DJ actually queries. Here's what one row holds:

| field | source | example |
|---|---|---|
| `artist`, `title`, `mix_name` | Beatport | `Eric Prydz`, `Call on Me`, `Extended Mix` |
| `bpm` | Beatport (rounded) | `126.0` |
| `tempo_precise` | DJ Studio (WASM beat detection) | `126.013` |
| `key` | Beatport (tonal) | `Bb Major` |
| `mik_key` | DJ Studio MIK (Camelot) | `6A` |
| `mik_key_confidence` | DJ Studio | `0.992` |
| `genre`, `sub_genre`, `label`, `catalog_number`, `isrc` | Beatport detail | `House / Progressive House / Pryda / PRYDA065 / GB...` |
| `release_date`, `length_ms` | Beatport detail | `2024-02-23`, `441000` |
| `mik_nrg` | DJ Studio MIK | `9` (1–10) |
| `vocals_avg`, `drums_avg`, `bass_avg`, `melody_avg` | DJ Studio Demucs | per-stem RMS |
| `vocals_peak`, `drums_peak`, `bass_peak`, `melody_peak` | DJ Studio Demucs | per-stem peak RMS |
| `cue_points_count` | DJ Studio | `7` |
| `analysis_json` | DJ Studio | energy segments + 1Hz stem curves + per-segment stem RMS |
| `rk_analysis_json` | rekordbox PSSI | semantic phrase labels (Intro/Verse/Chorus/Outro/Up/Down/Bridge) |

The interesting one is `analysis_json`. Its `energy.segments[]` array carves the track into structural sections — a 5-min club track typically has 6–8 segments — and each segment has a 1–9 energy rating, a beat range, and per-stem RMS. The `stems[*].curve_1hz` array is one mean-RMS-per-second per stem, so for a 5-minute track you get ~300 floats per stem describing exactly where the vocals come in, where the bass drops, where the drums are stripped back.

A real fragment from one track:

```json
{
  "tempo": {"bpm": 130.42, "downbeat_time_sec": 0.299},
  "key": {"main": "8A", "main_confidence": 0.992},
  "energy": {
    "overall": 7,
    "segments": [
      {"start_sec": 0.0,    "end_sec": 44.2,  "energy": 5},
      {"start_sec": 44.2,   "end_sec": 103.2, "energy": 6},
      {"start_sec": 103.2,  "end_sec": 117.9, "energy": 4},
      {"start_sec": 117.9,  "end_sec": 176.9, "energy": 7},
      {"start_sec": 176.9,  "end_sec": 191.7, "energy": 6},
      {"start_sec": 191.7,  "end_sec": 265.4, "energy": 7},
      {"start_sec": 265.4,  "end_sec": 324.3, "energy": 6}
    ]
  },
  "stems": {
    "vocals": {"avg_rms": 0.017, "curve_1hz": [0.003, 0.003, ...]},
    "drums":  {"avg_rms": 0.142, "curve_1hz": [...]},
    "bass":   {"avg_rms": 0.108, "curve_1hz": [...]},
    "melody": {"avg_rms": 0.071, "curve_1hz": [...]}
  }
}
```

This is what nothing else in the DJ tooling stack will give you in one place: structural sections + stem energy curves + key + tempo + release context, all keyed on the same `beatport_id`.

## Why each field matters for actually playing a set

**BPM.** Mixing >6 BPM apart needs a creative transition or you lose the floor. Sort and bucket aggressively. `tempo_precise` matters for sync — Beatport rounds `130.4` to `130` and that 0.4 BPM compounds into drift after 2 minutes.

**Key (Camelot).** `mik_key` is in Camelot wheel notation. The compatibility rule: same number (8A↔8B is a relative-key switch, mood shift), adjacent number same letter (8A↔7A or 8A↔9A is energy up/down with same mood). Two tracks that are key-compatible AND within 4 BPM are essentially guaranteed to blend; you can build a 60-min set as a pure SQL query.

**Release date.** A 2026 set built only from 2026 releases sounds fresh but flat. Most working DJs aim for ~70% from the last 18 months and ~30% older anchors the crowd recognises. The `release_date` column makes this trivial to enforce.

**Energy segments + stems.** This is the actually-magic part. `mik_nrg` (1–10) is the track's overall energy, but the segment array tells you the *shape* — where the intro ends, where the breakdown sits, where the drop hits. Combined with per-stem RMS per segment, you know whether a "drop" is a kick-heavy peak or a vocal moment. The 1Hz stem curves let you answer "where in this track do the vocals come in?" to the second, which is what you need when you're mixing a vocal track over a beats-only outro.

## Sample of the joined dataset

Real rows from the DB:

| artist | title | bpm | bp_key | mik_key | nrg | voc | drm | bas | mel | cues |
|---|---|---|---|---|---|---|---|---|---|---|
| Dimitri Vegas, Martin Garrix, ... | Tremor | 128.0 | Gb Minor | 11A | 9 | 0.049 | 0.146 | 0.117 | 0.065 | 8 |
| W&W, Blasterjaxx | Rocket | 128.0 | Eb Major | 2A | 9 | 0.077 | 0.210 | 0.055 | 0.105 | 8 |
| Eric Prydz | Call on Me | 126.0 | Bb Major | 6A | 9 | 0.007 | 0.074 | 0.052 | 0.110 | 7 |
| Avancada, Darius & Finlay | Xplode | 131.0 | Eb Minor | 2A | 9 | 0.036 | 0.149 | 0.066 | 0.103 | 7 |
| Tigger, Giovani | New Delhi | 138.0 | Db Major | 12A | 9 | 0.011 | 0.099 | 0.191 | 0.051 | 7 |

Notice the stem breakdown: `Rocket` has vocals_avg=0.077 (high), `Tremor` has 0.049 (medium), `Call on Me` has 0.007 (basically instrumental — classic Prydz). The numbers cleanly separate vocal-driven tracks from instrumental ones, which is exactly what you need when you're picking the next track.

## What you can do once an LLM agent has this DB

Point Claude Code (or any agent that speaks SQLite) at `~/Music/dj/dj.db` and the queries write themselves. A few that are useful in practice:

**"Build me a 90-minute warmup → peak → cooldown set in compatible keys."**

```sql
WITH ramp AS (
  SELECT et.beatport_id, et.artist, et.title, et.bpm, eta.mik_key,
         eta.mik_nrg, eta.duration_sec,
         CAST(SUBSTR(eta.mik_key, 1, LENGTH(eta.mik_key)-1) AS INT) AS k_num,
         SUBSTR(eta.mik_key, -1, 1) AS k_letter
  FROM enriched_tracks et
  JOIN enriched_tracks_analysis eta USING(beatport_id)
  WHERE et.genre LIKE '%House%'
    AND eta.duration_sec BETWEEN 300 AND 500
    AND et.release_date >= date('now', '-18 months')
)
SELECT artist, title, bpm, mik_key, mik_nrg
FROM ramp
WHERE mik_nrg <= 6 AND bpm BETWEEN 122 AND 124
ORDER BY mik_nrg, bpm
LIMIT 8;
```

The agent can iterate — pick the next bucket, constrain `k_num` to ±1 from the previous track's key, walk the energy up.

**"Find me instrumental tracks with long beat-only intros — good for transitions."**

```sql
SELECT et.artist, et.title, eta.cue_points_count,
       eta.vocals_avg, eta.drums_avg
FROM enriched_tracks et
JOIN enriched_tracks_analysis eta USING(beatport_id)
WHERE eta.vocals_avg < 0.02
  AND eta.drums_avg > 0.10
  AND eta.cue_points_count >= 6
ORDER BY eta.drums_avg DESC;
```

**"Show me 2026 releases on labels I've played from before."**

```sql
SELECT et.artist, et.title, et.label, et.release_date, et.bpm, eta.mik_key
FROM enriched_tracks et
JOIN enriched_tracks_analysis eta USING(beatport_id)
WHERE et.release_date >= '2026-01-01'
  AND et.label IN (
    SELECT DISTINCT label FROM enriched_tracks
    WHERE detected_track_id IS NOT NULL
      AND label IS NOT NULL
  )
ORDER BY et.release_date DESC;
```

**"Where in 'Call on Me' do the vocals first come in?"** — answered by parsing `analysis_json.stems.vocals.curve_1hz` and finding the first index where RMS crosses a threshold. The agent can write the JSON query inline.

**"Suggest a closing track that energy-ramps down from the last drop of the previous track."** — the agent reads the previous track's last `energy.segments` entry, then searches for tracks whose first segment is one energy notch lower and in a key adjacent to the previous track's `mik_key`.

The point isn't any one of these queries. It's that the whole library has been flattened into one SQLite file where every track has the same set of comparable, numeric, machine-queryable fields. Tempo precision, key compatibility, structural sections, stem energy, release recency, label provenance — all in one row, all joinable, all in the place an LLM can reason about without you ever opening a DAW.

Beatport gives you the rights to the track. `enrich` gives you the data to actually play it.
