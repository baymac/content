---
title: 'Automating Beatport playlist sorting with Claude Code'
date: '2026-04-25'
tags: claude, automation, beatport, playwright, python
ai-gen: true
---

I had ~4000 tracks sitting in a Beatport playlist called `Library Songs` and twenty genre playlists I'd been hand-sorting them into for months. Open track, read genre, click `Add to Playlist`, pick destination, dismiss the duplicate warning if it pops, next track. I kept telling myself I'd finish it next weekend.

So I wrote one chunky paragraph for Claude Code's skills creator. It had the credentials, the source playlist, the destination playlists, what the modal did, the genre-to-playlist mapping, and one line at the end: "first try this with the LLM one step at a time, recording timings and selectors, then write code to automate it." Then I said: rewrite this like a skill. That's the whole bootstrap.

## What the skill creator gave back

Claude Code's skills creator turns a prose prompt into a proper skill folder. From my one paragraph, it produced:

- `SKILL.md` — frontmatter with the trigger description, then a structured body covering credentials, per-track flow, classifier rules, and failure modes
- `references/classifier.md` — the genre-to-playlist map extracted into its own file with exact-match and word-contains rules
- `scripts/discovery_template.py` — a Playwright skeleton with empty `SELECTORS = {}` and `TIMINGS = {}` dicts for phase 1 to fill in
- `scripts/automate_template.py` — the executor skeleton with the `modal-races-modal` pattern wired up

Three things the creator did with my prompt that turned out to matter:

1. It promoted my "try with the LLM one step at a time" line into the entire discovery-first methodology in `SKILL.md`. One sentence became a multi-paragraph contract about probing the UI manually before writing code.
2. It split the classifier into `exact match` and `contains` rules, with ordering guidance ("check more specific terms first"), because my prompt mentioned some genres have parenthesised qualifiers.
3. It added a `Failure modes` section covering selector drift, hover-required actions, modal stacking, genre tag truncation, and pagination vs infinite scroll. None of those were in my prompt.

The skill body also added an `Order of operations` checklist with a hard dry-run gate: you don't run on the full playlist until you've shown the user a 3-track preview and gotten confirmation. That gate is the most important line in the skill.

The lines that became real constraints weren't the headlines — they were the throwaways. "Note some left side values might not be exactly the same so you can do pattern matching" became the entire fuzzy-match policy in `classifier.md`, including the order-matters rule for `Hard Techno` vs generic `Techno`. "If any of the genre is not mentioned in the key map, then ignore" became the `skipped: no_match` outcome and a logging rule. "Keep track of them until it is finished" became the resumable `processed_track_ids.txt` pattern.

A skill is only as good as the edge cases in your bootstrap prompt. Write it like you're describing the job to a new hire. Include the failures, the workarounds, the "oh and by the way..." footnotes.

## Probe before you automate

The skill says it explicitly: probe one step at a time, record selectors and timings, then write the script. Three things on Beatport that would've burned me if I'd skipped this:

1. The `OneTrust` cookie banner sits behind the viewport and silently intercepts pointer events. Every click in the script retries until it times out.
2. The track list is virtualised. A `page.locator(...)` snapshot at load time misses 95% of the playlist.
3. The `Add to Playlist` modal animates in with an overlay that intercepts clicks for ~400ms after open.

Discovery output is `discovery_notes.md` at the repo root — selectors, observed timings, quirks. Phase 2 reads it and codifies. Without phase 1, I'd have found out about the cookie banner on track 800.

## The pivot: drive the API, not the modal

Two minutes into discovery, I noticed the data calls. Beatport's web app authenticates with `NextAuth`, but every actual fetch hits `api.beatport.com/v4/...` with `Authorization: Bearer <jwt>`. Once you have the token, the web UI is a thin client over a clean REST API.

I told Claude to forget the modal. Find the API.

Claude ran a sniffer probe — headless Playwright that logged every request to `api.beatport.com` after login. The whole surface fell out in three runs:

```
GET    /v4/my/playlists/?page=1&per_page=50              # list user playlists
GET    /v4/my/playlists/{id}/tracks/?page=N&per_page=100 # paginate Library Songs
GET    /v4/my/playlists/{id}/tracks/ids/                 # cheap dedup query
POST   /v4/my/playlists/{id}/tracks/bulk/                # add. body: {"track_ids":[N]}
DELETE /v4/my/playlists/{id}/tracks/bulk/                # remove. body: {"item_ids":[entry_id]}
```

4054 tracks across 41 pages of 100. No infinite scroll at the API level. Just plain pagination. The job that looked like "drive a SPA's modal four thousand times" became "loop and POST."

If the page hits a JSON endpoint, drive that endpoint. Selectors drift, animations race, virtualised lists hide rows. Versioned APIs don't move much.

## The duplicate trap

The `SKILL.md` talked about a `Duplicate Tracks Detected` modal that pops when you try to add a track already in the destination. The execution flow handles it: race two waits, click `Cancel` on the warning.

When Claude probed the API directly, it added a test track to the `Trance` playlist. 200 OK. Added it again. 200 OK. The track count went up. Third time. Up again.

Beatport's dedup is client-side only. The `Duplicate Tracks Detected` modal is the web app refusing to make the call. The server has no opinion on duplicates. If you naively replay the bulk endpoint, you stuff the same track into the same playlist as many times as you want.

Had Claude trusted the `SKILL.md` narrative and built the executor around the modal pattern, it would have silently created thousands of duplicates.

The fix is a pre-fetch:

```python
def list_track_ids(playlist_id: int) -> set[int]:
    r = client.get(f"{API_ROOT}/my/playlists/{playlist_id}/tracks/ids/")
    return {item["track_id"] for item in r.json()["results"]}

dest_track_ids = {
    name: list_track_ids(pl_id) for name, pl_id in destinations.items()
}

if track_id in dest_track_ids[dest_name]:
    log({"track_id": track_id, "outcome": "duplicate"})
    continue

add_track(dest_id, track_id)
dest_track_ids[dest_name].add(track_id)
```

Three lines of real dedup logic. Wouldn't have existed without phase 1. Probing beats trusting the spec.

## Auth: Playwright in, httpx the rest

I didn't want to run Playwright for 4000 API calls. Just needed the bearer token. So:

1. Headless Playwright runs the login flow once: navigate, dismiss the cookie banner, fill the form on `account.beatport.com`, wait for the OAuth callback.
2. A request listener grabs the first `Authorization: Bearer <jwt>` header sent to `api.beatport.com` after login.
3. Browser closes.
4. `httpx.Client` takes over with that header and runs the entire classifier loop.

Token TTL is ~60 minutes. Full run takes ~35 minutes. If a `401` comes back mid-run, the client re-runs the Playwright login and retries. Wired that path once, never had to use it.

## Logging and resumability

For a 40-minute autonomous run, visibility matters more than speed. Everything goes to JSONL — one JSON object per line, streamed to stdout and appended to `logs/run_log.jsonl`. Every track gets a line:

```json
{"event": "track", "track_id": 16282207, "label": "abcdefu (Original Mix) by Fat Tony, MEDUN, Tiffany Aris", "genre": "Dance / Pop", "destination": "Dance", "outcome": "duplicate", "ts": 1777118772.0}
{"event": "track", "track_id": 13257248, "label": "About Us (feat. EMME) (Extended Mix) by Emme, Le Youth", "genre": "Melodic House & Techno", "destination": "Melodic House", "dest_id": 7241417, "entry_id": 371916542, "outcome": "added", "ts": 1777118772.6}
```

The `outcome` field is enumerable (`added`, `duplicate`, `skipped_no_match`, `skipped_no_genre`, `error`, `noop_empty_items`) so you can `jq 'select(.outcome=="error")'` and audit just the bad ones. It's append-only, so if the run crashes, the log is intact.

Next to the log: `state/processed_track_ids.txt`, a flat list of every track ID we've made a routing decision on. Re-running the executor reads that file and skips anything already processed. Idempotent by construction.

The 3-track dry run wrote the same JSONL format. Before saying go, I scrolled four lines, saw `abcdefu → Dance (duplicate)`, `About Us → Melodic House (added)`, `Above It All → Melodic House (added)`, `Abyss → Bass House (added)` and that was enough to trust the pattern.

## What broke mid-run

About 40 tracks into the full run, the executor crashed with an `IndexError` on `resp["items"][0]`. Beatport's bulk-add endpoint had returned `{"items": [], "playlist": {...}}` for one track — a 200 with empty items. Probably a tombstoned or region-locked track the server silently no-ops on.

The fix was three lines:

```python
items = resp.get("items") or []
if not items:
    log({"event": "track", "outcome": "noop_empty_items", ...})
    mark_processed(track_id)
    continue
```

Resumed from `processed_track_ids.txt`, no work lost. The state file paid for itself the first time something went sideways.

## Result

Source playlist had 4054 entries at run time:

```
4054 entries in Library Songs (live API)
  3397 unique track_ids
   657 in-source duplicates (same track at multiple positions)
    44 tombstoned (delisted by Beatport — iterator skips them)

3357 routing decisions logged across all runs
  2801 added            (83.4%)
   461 skipped_no_match (13.7%)  — Pop, Organic House, World Music, etc.
    77 noop_empty_items  (2.3%)  — 200 with empty items[], silently no-op'd
    18 duplicate         (0.5%)  — already in destination, caught by pre-check
     0 hard errors
```

I cross-checked against a Beatport CSV export from earlier. The 44 tombstoned entries from the API matched exactly the 44 tracks in the CSV that never appeared in my logs. Same number, not a coincidence. Verified, not guessed.

Wall time: ~38 minutes for the resumed full run, plus ~12 minutes across the dry runs and the first crashed attempt.

If Beatport ships a UI change tomorrow, none of the code breaks. I add tracks to `Library Songs`, type "run the beatport classifier", Claude does the rest.

## Future work

- Log a `repeat_in_source` outcome for the silent-skip case so the run is fully accounted for
- Add a `tombstoned` outcome instead of dropping those entries inside the iterator

## Acknowledgements

Claude Code wrote the skill, ran the discovery, built the executor, and managed the 35-minute autonomous run. I wrote one paragraph and pressed go.
