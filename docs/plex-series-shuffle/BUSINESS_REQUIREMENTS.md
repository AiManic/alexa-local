# Plex "Series-Aware Shuffle" — Business Requirements

| | |
|---|---|
| Status | Draft v0.1 |
| Owner | Christian Hoare |
| Date | 2026-10-03 |
| Companion doc | [TECHNICAL_SPECIFICATION.md](TECHNICAL_SPECIFICATION.md) |

## 1. Problem statement

Plex's built-in shuffle picks episodes at random. When shuffling a TV playlist or
a whole TV library, the viewer can land on *Season 3 Episode 7* of a show they
have not watched since the start. Episodic content is meant to be watched in
order, so random shuffle breaks continuity and spoils storylines.

## 2. Goal

Provide a shuffle mode where **the order of shows is random, but the order of
episodes within each show is always sequential**.

### Worked example

Shows: A, B, C. Each starts at S01E01.

| Play # | Shuffle picks | Plays |
|---|---|---|
| 1 | A | A S01E01 |
| 2 | B | B S01E01 |
| 3 | A | **A S01E02** (not a random A episode) |
| 4 | C | C S01E01 |
| 5 | B | **B S01E02** |
| 6 | A | **A S01E03** |

Rule: *every time a show is picked, it plays the episode that follows the last
one played from that show in this shuffle.*

## 3. Users and scenarios

| Persona | Scenario |
|---|---|
| Household viewer | "Shuffle all my sitcoms" for background viewing without losing continuity. |
| Household viewer | Shuffle one playlist (e.g. "Comfort TV") and stop/resume across several evenings, picking up where each show left off. |
| Server admin | Install once; works for every user/client without per-device setup. |

## 4. Scope

### In scope
- Series-aware shuffle for: a **Plex playlist** (episodes), a **TV library section**, or a chosen **subset of shows**.
- Persistent per-show progress so the shuffle can be paused and resumed.
- Mixed playlists: non-episode items (movies) are treated as stand-alone "shows" of one item.
- Works with the standard Plex clients (TV apps, web, mobile) without modifying them.

### Out of scope (v1)
- Modifying Plex Media Server internals or the native "Shuffle" button.
- Music shuffling.
- Multi-user shared shuffle state (each Plex user has independent state).
- Transcoding, library management, metadata editing.

## 5. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | User can start a series-aware shuffle from a playlist, a library section, or a list of shows. | Must |
| FR-2 | Within one show, episodes play strictly in order (season, then episode; specials configurable). | Must |
| FR-3 | The choice of which show plays next is random. | Must |
| FR-4 | The first episode of each show is the first **unwatched** episode by default, or S01E01 if the user selects "from the beginning". | Must |
| FR-5 | When a show runs out of episodes it is dropped from the rotation; shuffle ends when all shows are exhausted (or loops, if "repeat" is on). | Must |
| FR-6 | Shuffle position is saved so a stopped session resumes at the correct next episode per show. | Must |
| FR-7 | Option to avoid the same show twice in a row (when >1 show remains). | Should |
| FR-8 | Option to weight shows (e.g. equal per show vs proportional to remaining episodes). | Should |
| FR-9 | Option to exclude shows/items and to honour "skip watched". | Should |
| FR-10 | Playback is started on a user-selected Plex client with a single action (web UI, CLI, or API call). | Must |
| FR-11 | If the user manually skips or watches an episode outside the shuffle, the shuffle's next-episode pointer for that show stays correct. | Should |
| FR-12 | Admin can run multiple named shuffles ("Sitcoms", "Dramas"). | Could |

## 6. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-1 | Queue creation for a 500-show library completes in < 10 s. |
| NFR-2 | No changes to media files or Plex database; Plex is accessed only via its supported HTTP API. |
| NFR-3 | Plex auth tokens are stored securely (env/secret store), never logged. |
| NFR-4 | Self-hosted, runs on the same LAN as the Plex server (Docker image or Node process). |
| NFR-5 | Fails safely: if the service is down, normal Plex playback is unaffected. |

## 7. Constraints and assumptions

- Plex retired its in-server **Plugin Framework**; a classic server-side "plugin" is no longer a viable delivery model. This is delivered as a **companion service** that uses the Plex HTTP API (see TSD §2).
- Some features (webhooks) require an active **Plex Pass** subscription.
- Remote control of a specific client requires that client to expose the Plex "companion" control interface; otherwise the user starts playback from the client themselves (see TSD §6).
- Plex API behaviour in the TSD should be validated in a spike against a real server before build.

## 8. Success criteria

1. In a test library of ≥ 5 shows, a 30-episode shuffle never plays episode *N+1* of a show before episode *N*.
2. Stopping and restarting a shuffle resumes every show at the correct next episode.
3. Works on at least the Plex web player plus one TV/mobile client.
4. Viewer needs no more than one action to start a shuffle.

## 9. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Client does not support remote control | Can't auto-start playback | Use a pre-built Play Queue / playlist the user opens (Mode A) |
| Plex API changes | Breakage | Isolate API calls in one module; pin tested server version |
| Webhooks need Plex Pass | Dynamic mode limited | Fall back to polling sessions / websocket alerts |
| Very long queues slow some clients | Poor UX | Generate queue in rolling windows (e.g. 50 items) |

## 10. Existing alternatives considered

Tools such as **Tunarr / dizqueTV / PseudoTV** (channel-style scheduling), **Kometa** (collections) and **Tautulli** (monitoring) cover related ground but none provide "random across shows, sequential within show" as a first-class mode on arbitrary playlists. Worth a short evaluation before building.
