# Splitstone API

Splitstone's public API gives you the platform's competitive data: players, matches with their splits, timelines and Twitch VOD links, live matches, leaderboards, playoff points and weekly cups. It is for trackers, stats sites, Twitch chat commands, Discord bots and stream overlays. It is read-only: nothing in it changes anything on Splitstone.

- **Address:** `https://api.splitstonemc.com/v1/`
- **Answers:** JSON. Times are ISO 8601 in UTC (`2026-11-06T18:04:11Z`); durations are whole milliseconds (`timeMs`, `rtaMs`).
- **Machine-readable description:** [openapi.yaml](openapi.yaml).
- **Problems and requests:** open an issue in this repository.

## Limits and keys

| You send | Limit | For |
|---|---|---|
| Nothing | 500 requests per 10 minutes per IP address | Almost everyone |
| `API-Key: ssd_…` | The key's own limit, counted per key | A site that needs more; email **contact@splitstonemc.com** with what you are building and how many requests you expect |
| `Private-Key: ssp_…` | As without a key | Reading one player's own live match, for their overlay |

Every answer carries `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset` (when the window ends, in Unix seconds). Past the limit you get `429` with `Retry-After` in seconds.

**A player's key** is made in the game: **Settings → Linked accounts → API key → Make and copy**. It is copied to the clipboard and shown only then. Making a new one stops the old one. Treat it like a password: an overlay should keep it out of anything shown on stream.

Answers say how long they may be kept (`Cache-Control: public, max-age=N`): 5 seconds for live data, up to a day for a finished match. Please keep them that long rather than asking again. Browsers may call the API from any page (`Access-Control-Allow-Origin: *`).

## Errors

Errors are `{"error": "what went wrong"}` with the status: `400` for a bad parameter, `401` for an unknown `API-Key` or a missing `Private-Key`, `403` for a `Private-Key` that is not that player's, `404` when there is no such thing, `405` for anything but `GET`, `429` past the limit, and `503` when the service cannot answer.

## Players

A player is named by their Minecraft name (any case) or UUID, with or without dashes.

### `GET /players/{player}`

The public profile: `uuid`, `name`, `memberSince`, `country` and `bio` (null when hidden), `linkedAccounts` the player shows (`service`, `handle`, `url`), `ratings` per mode (`mode`, `elo`, `ratedMatches`, `draws`, `place` on the leaderboard or null, `top20`), `lastPlayed`, `recentMatches` (the last 100, from the player's side) and `bests` per mode (best finish, sum of best and each split's best, with when each was set).

### `GET /players/{player}/matches`

The player's matches, newest first, in the [match list](#match-lists) form. Parameters: `mode`, `season`, `before`, `after`, `count`.

### `GET /players/{player}/versus/{other}`

Head-to-head over every match the two played on different sides:

```json
{"players": [{"uuid": "…", "name": "Alice"}, {"uuid": "…", "name": "Bob"}],
 "modes": [{"mode": "ranked_1v1", "matches": 3, "wins": {"<alice uuid>": 2, "<bob uuid>": 1}, "draws": 0,
            "eloChange": {"<alice uuid>": 25, "<bob uuid>": -25}}]}
```

`eloChange` is the sum of the rated changes between them; weekly cup and Major games change no Elo, and it is null in modes whose Elo is never shown.

### `GET /players/{player}/versus/{other}/matches`

Their matches against each other, in the [match list](#match-lists) form, with the same parameters.

### `GET /players/{player}/seasons`

Each season the player played, newest first: `season`, `current`, and per mode `elo` and `place` (an ended season's final ones; the current season's are the player's now), `matches`, `wins`, `draws`, `losses` and `bestMs`.

### `GET /players/{player}/live`

Needs that player's `Private-Key`. The match or private room they are in now, or `{"match": null}`. Never cached.

```json
{"match": {"id": 1000000412, "kind": "room", "mode": "room", "status": "running", "runningMs": 412000,
  "players": [{"uuid": "…", "name": "Alice", "elo": null, "split": "minecraft:nether/find_bastion", "splitMs": 301200,
               "splits": [{"split": "minecraft:story/enter_the_nether", "ms": 150400}, {"split": "minecraft:nether/find_bastion", "ms": 301200}],
               "timeMs": null, "forfeited": false, "left": false}],
  "spectators": [{"uuid": "…", "name": "Carol"}],
  "timeline": [{"uuid": "…", "event": "minecraft:story/enter_the_nether", "ms": 150400, "rtaMs": 150400},
               {"uuid": "…", "event": "splitstone:death", "ms": 170020, "rtaMs": 170020}],
  "completions": []}}
```

- `kind` is `ranked` (a ranked queue, weekly cup or Major match) or `room` (their private room, Major race seed or public room).
- `status`: a ranked match is always `running`. A room is `idle` between matches (then `id` is null and the players are the room's members), then `loading`, `countdown`, `running` and `done`.
- `timeline` is every event in order, deaths (`splitstone:death`) and resets (`splitstone:reset`) included; `ms` is the run's time, `rtaMs` real time.
- `completions` are a room's finishes, fastest first: `uuid`, `name`, `timeMs`. Each room player also has `timeMs` once finished, `forfeited` and `left`.
- `spectators` are who watches from outside the match. A room's code is never given.

## Matches

### Match lists

`GET /matches` lists every public match, newest first. Public matches are ranked queue matches, weekly cup games and Major games; private rooms and unranked tests are never listed. Seeds are never given.

| Parameter | Meaning |
|---|---|
| `mode` | Only this mode, such as `ranked_1v1`, `ranked_2v2`, `coop`, `weekly_cup` |
| `season` | Only this season |
| `cup` | Only this weekly cup's games |
| `before` | Matches older than this match id: the next page |
| `after` | Matches newer than this match id, the ones just after it |
| `count` | How many, 1 to 100; 20 by default |

Each match: `id`, `mode`, `seedType`, `season`, `startedAt`, `endedAt`, `ending` (`finish`, `forfeit`, `time`, …), `event` (`{"name": "Gold cup", "round": "Week 3"}` or `{"name": "Major 2", "round": "Quarterfinal, game 3"}`, null for a queue match), `winnerTeam` (1 or 2 in team matches) and `players`: `uuid`, `name`, `team`, `eloBefore`, `eloAfter`, `eloChange`, `timeMs` (null without a finish) and `won`. Elo is null where it is never shown (co-op and game modes), and a cup or Major game shows only the Elo brought.

### `GET /matches/{id}`

One match, finished or running. Beyond the list's fields:

- `status`: `live` or `finished`. While it runs, the seed type, the end and Elo after are null.
- Each player's `place` and `top20` on the leaderboard now, and `splits` of their side's last world (a reset starts them again; teammates share the host's world): `split`, `ms`, `rtaMs` (real time since the start) and `vodAt`.
- `timeline`: every event in the order received, deaths and resets included: `uuid`, `event`, `ms`, `rtaMs`.
- `vods`: `{uuid, url, startsAt}` for each player whose linked Twitch is shown on their profile and whose channel saved the broadcast. That player's `vod` is the same, and each of their splits' `vodAt` links straight to that moment: `https://www.twitch.tv/videos/123?t=1h04m12s`. The moment is the match start plus the split's real time, less when the broadcast began. Twitch deletes past broadcasts after 7 to 60 days, so old links can stop working.

Splits are named as the game names them: `minecraft:story/enter_the_nether`, `minecraft:nether/find_bastion`, `minecraft:nether/find_fortress`, `minecraft:story/follow_ender_eye`, `minecraft:story/enter_the_end`.

### `GET /live`

Ranked, weekly cup and Major matches running now with a top-100 player in them, where every player lets everyone watch, highest rated first, at most 100: each with `id`, `mode`, `runningMs` and its players (`uuid`, `name`, `elo`, `split`, `splitMs`, `splits`, and `liveUrl`: their Twitch channel while they stream with it shown on their profile, otherwise null). `playersOnline` gives how many games are connected.

## Boards

### `GET /leaderboards/{mode}`

The Elo ladder (`ranked_1v1` or `ranked_2v2`): placed players who played in the last 7 days, the top 100, equal Elo sharing a place. Without parameters it is this season's board with each row's `place`, `uuid`, `name`, `elo`, `rank`, `division`, `country`, `wins`, `draws`, `losses`, `bestMs`, `form` (last five results), `lastEloChange`, `placeLastWeek` and `top20`.

With `season` (an ended season gives its final places) or `country` (a two-letter code; only players who show that country), rows have `place`, `uuid`, `name`, `elo`, `rank`, `division`, `country` and `top20`. `count` takes fewer than 100.

### `GET /records/{mode}`

The fastest finishes in a ranked mode, or private rooms with `records/rooms` (rooms with every setting at its default): `place`, `uuid`, `name`, `country`, `timeMs`, `seedType` and, in 2v2, the `mate`. In 1v1 each player's best counts once; in 2v2 each pair's. Parameters: `period` (`all`, `month`, `week` or `day`; `all` by default) and `country`.

### `GET /playoffs`

This season's playoff race (or `season`'s): each Major whose field is drawn (`number`, `week`, `status`, and once settled its places with the points each earned) and the points table, the top 100: `place`, `name`, `uuid`, `ladder` (best 10 weeks), `cups` (best 6 Netherite cups), `majors` and `total`.

## Weekly cups

### `GET /cups`

Weekly cups, newest first: `id`, `season`, `week`, `rank` (the cup's rank, such as `gold` or `netherite`), `mode`, `opensAt`, `closesAt`, `matchesPerPlayer`, `settled`, `players` and `winners` (first place, shared on a tie). Parameters: `season`, `week`, `count` (1 to 100, 20 by default).

### `GET /cups/{id}`

One cup with its `results` in place order: `place`, `uuid`, `name`, `wins`, `draws`, `losses`. Its games are at `GET /matches?cup={id}`.

## Status

### `GET /status`

`{"matchesRunning": 12, "playersOnline": 340, "queueSize": 9}`: as many matches as `/live` lists, connected games, and players queueing.

## Compared with MCSR Ranked's API

Everything MCSR Ranked's API gives has a counterpart here: users → `/players`, user matches → `/players/{player}/matches`, versus → `/players/{player}/versus/{other}`, user seasons → `/players/{player}/seasons`, user live → `/players/{player}/live`, matches → `/matches`, live → `/live`, leaderboard → `/leaderboards`, record leaderboard → `/records`, phase leaderboard → `/playoffs`, status → `/status`. Splitstone has weekly cups (`/cups`) where MCSR Ranked has weekly races.
