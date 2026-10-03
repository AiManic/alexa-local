# Plex "Series-Aware Shuffle" — Technical Specification

| | |
|---|---|
| Status | Draft v0.1 |
| Date | 2026-10-03 |
| Implements | [BUSINESS_REQUIREMENTS.md](BUSINESS_REQUIREMENTS.md) |

> Endpoint names and parameters below are from Plex's documented/community-documented
> HTTP API and must be verified in a spike (Phase 0) against the target server version.

## 1. Feasibility summary

**Yes, this is buildable.** The core insight is that "random across shows,
sequential within a show" does not need runtime magic: it is equivalent to a
**random interleaving of N already-ordered lists**. We can compute the whole
ordering up front and hand it to Plex as a Play Queue. A dynamic mode (top-up
the queue as playback proceeds) is layered on for endless/resumable shuffles.

## 2. Delivery model: companion service, not a server plugin

Plex deprecated and removed its in-process Plugin Framework (Python/Plex plugins),
and the native Shuffle button is not extensible. Therefore:

- A standalone **Series Shuffle service** (Node.js, matching this repo's stack) talks to Plex over HTTP.
- Optional thin front ends: small web UI, CLI, and (given this repo) an Alexa skill intent such as *"Alexa, shuffle my sitcoms"*.

```
 ┌────────────┐   REST    ┌──────────────────────┐   HTTP/JSON    ┌───────────────┐
 │ Web UI/CLI │──────────▶│  Series Shuffle svc  │───────────────▶│ Plex Media    │
 │ / Alexa    │           │  - Planner           │◀───────────────│ Server        │
 └────────────┘           │  - Plex client       │  webhooks/WS   │               │
                          │  - State store       │                └──────┬────────┘
                          └──────────┬───────────┘                       │ playback
                                     │ SQLite                     ┌──────▼────────┐
                                     ▼                            │ Plex client   │
                              shuffle state                       └───────────────┘
```

## 3. Core algorithm (Planner)

Inputs: source (playlist | section | show list), options (start mode, no-repeat, weighting, loop).

1. **Resolve items** → list of episodes with `ratingKey`, `grandparentRatingKey` (show), `parentIndex` (season), `index` (episode), `viewCount`. Non-episode items get their own singleton "show" key.
2. **Group by show**, sort each group by `(parentIndex, index)`; for playlists, keep playlist order within a show instead if "respect playlist order" is set. Specials (season 0) placed last or excluded per option.
3. **Starting pointer per show** — first unwatched episode (`viewCount == 0`) or index 0; or the saved pointer when resuming.
4. **Interleave**: repeatedly pick a show at random from shows with remaining episodes and emit its next episode.
   - *Weighting*: equal (`w=1`) or proportional (`w=remaining`) — proportional spreads shows evenly so long shows don't dominate the tail.
   - *No-repeat*: exclude the previously picked show unless it is the only one left.
5. **Loop**: when all exhausted and `repeat` is on, reset pointers and continue.

```js
function* planShuffle(groups, { weighted, noRepeat, rng = Math.random }) {
  const cursors = new Map([...groups].map(([k, eps]) => [k, 0]));
  let last = null;
  while (true) {
    let live = [...groups].filter(([k, eps]) => cursors.get(k) < eps.length);
    if (!live.length) return;
    if (noRepeat && live.length > 1) live = live.filter(([k]) => k !== last);
    const pick = weightedPick(live, weighted, rng, cursors);
    const [k, eps] = pick;
    yield eps[cursors.get(k)];
    cursors.set(k, cursors.get(k) + 1);
    last = k;
  }
}
```

Property (testable): for every show, the emitted subsequence equals its sorted episode list prefix — i.e. never out of order. Use a seeded RNG for reproducible tests.

## 4. Operating modes

### Mode A — Static Play Queue (MVP, most compatible)
1. Run Planner for the whole source (or first N=50 items).
2. `POST /playQueues?type=video&uri=server://{machineId}/com.plexapp.plugins.library/library/metadata/{firstKey}&shuffle=0&continuous=0` → returns `playQueueID`.
3. Append the remaining items in order: `PUT /playQueues/{id}?uri=…/library/metadata/{key}` (or create a Plex **playlist** and fill via `PUT /playlists/{id}/items?uri=…`, which any client can open).
4. Because ordering is baked in, no runtime tracking is needed for correctness.

Pro: works on every client; no Plex Pass. Con: queue is fixed once built; "resume" needs the state store (§5) to rebuild from saved pointers.

### Mode B — Dynamic (rolling queue)
Keeps a short window queued and extends it as playback proceeds.

- Trigger: Plex **webhook** `media.scrobble` / `media.stop` (Plex Pass) **or** the server alert websocket `/:/websockets/notifications` (`playing` / `timeline` events) filtered to the active session.
- On each event: update the show's pointer, and if remaining queued items < window threshold (e.g. 10), run Planner from current pointers and append via `PUT /playQueues/{id}`.
- Handles FR-11: if the user watches episodes outside the shuffle, a `scrobble` for an episode of a tracked show advances that show's pointer to `max(current, watched+1)`.

### Starting playback
- Preferred: command a client via the Plex Media Player/companion API: `GET http://{client}:{port}/player/playback/playMedia?key=…&containerKey=/playQueues/{id}&…` (clients discovered through `/clients` or plex.tv resources).
- Fallback: return a link/instructions; user opens the playlist/queue from any client.

## 5. Data model (SQLite)

```
shuffle        (id PK, name, plex_user, source_type, source_ref, options_json,
                play_queue_id, created_at, updated_at, active)
shuffle_show   (shuffle_id FK, show_key, title, cursor INT, total INT,
                PRIMARY KEY (shuffle_id, show_key))
shuffle_event  (id PK, shuffle_id FK, rating_key, show_key, ts, kind)  -- audit/debug
```

- `cursor` = index of the *next* episode to play for that show.
- Episode lists are re-fetched on resume (library may have gained episodes); cursor is mapped by `(season, episode)` rather than raw index to survive additions.

## 6. Plex API surface used

| Purpose | Call |
|---|---|
| Identify server | `GET /` → `machineIdentifier` |
| List sections | `GET /library/sections` |
| Episodes in section | `GET /library/sections/{id}/all?type=4` (fields: `grandparentRatingKey`, `parentIndex`, `index`, `viewCount`) |
| Show children | `GET /library/metadata/{showKey}/allLeaves` |
| Playlists | `GET /playlists`, `GET /playlists/{id}/items` |
| Create/append queue | `POST /playQueues`, `PUT /playQueues/{id}` |
| Create/fill playlist | `POST /playlists`, `PUT /playlists/{id}/items` |
| Live events | Webhooks (Plex Pass) / `ws://…/:/websockets/notifications` |
| Sessions | `GET /status/sessions` |
| Auth | `X-Plex-Token` header on all calls |

Required headers: `X-Plex-Token`, `X-Plex-Client-Identifier`, `Accept: application/json`.

## 7. Service interface

```
POST /shuffles            { name, source:{type,ref}, options, client? }  → 201 {id, playQueueId}
GET  /shuffles/:id        → state incl. per-show cursors
POST /shuffles/:id/resume → rebuild/extend queue from saved cursors, optionally start on client
POST /shuffles/:id/reset  → reset cursors
DELETE /shuffles/:id
POST /webhook/plex        (multipart `payload` JSON from Plex)
```

CLI: `plex-shuffle start --playlist "Comfort TV" --client "Living Room" --no-repeat`.

## 8. Tech stack and layout

- Node.js 20+, Express, `better-sqlite3`, `ws`, `undici`/`axios`; Jest for tests. Matches the existing Node repo.
- Suggested structure:

```
src/
  plex/client.js       // all Plex HTTP calls (single choke point)
  planner/planner.js   // pure functions, no I/O
  state/store.js       // SQLite
  modes/static.js  modes/dynamic.js
  api/server.js    api/webhook.js
  alexa/handler.js     // optional intent → POST /shuffles
test/
```

## 9. Configuration

`PLEX_URL`, `PLEX_TOKEN`, `PORT`, `QUEUE_WINDOW` (default 50), `TOPUP_THRESHOLD` (10), `DEFAULT_OPTIONS`. Secrets via env only.

## 10. Testing

- **Unit**: Planner invariants (in-order per show, no-repeat, exhaustion, weighting distribution) using seeded RNG.
- **Integration**: mock Plex server (recorded JSON fixtures); resume after partial play; library gains episodes mid-shuffle.
- **E2E (manual)**: real Plex test library with 5 shows; verify on web + one TV client.

## 11. Delivery plan

| Phase | Deliverable |
|---|---|
| 0 – Spike (1–2 d) | Verify play-queue create/append, `allLeaves`, client control, webhook delivery on target server version |
| 1 – MVP (3–5 d) | Planner + Mode A (playlist/section → Play Queue/playlist) + CLI |
| 2 – State (2–3 d) | SQLite cursors, resume, "from unwatched" |
| 3 – Dynamic (3–4 d) | Mode B with webhook/websocket top-up, outside-watch sync |
| 4 – Polish | Web UI, Alexa intent, Docker image, docs |

## 12. Open questions

1. Which clients will be used (decides whether remote start is possible)?
2. Is Plex Pass available (webhooks) or should we rely on websocket/polling?
3. Playlists containing the same episode twice — dedupe or honour?
4. Specials/season 0 and multi-episode files — default handling?
5. Should state be per Plex user (managed users / home)?
6. Evaluate Tunarr/dizqueTV first to confirm nothing existing meets FR-1…FR-6.
