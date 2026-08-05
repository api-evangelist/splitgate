---
name: Walk a Splitgate player's match history into full match detail
description: Page through a player's Splitgate match history with cursor pagination, then expand any match into its full team and per-player breakdown.
api: openapi/splitgate-third-party-openapi-original.yml
operations: [searchPlayers, getPlayerMatches, getMatchDetails]
generated: '2026-08-05'
method: generated
source: openapi/splitgate-third-party-openapi-original.yml
---

# Walk a Splitgate player's match history into full match detail

Use this for "how did their last games go", scoreboard reconstruction, or building a recent-form summary.

## Before you start

- Base URL: `https://api.1047games.com`; `Authorization: Bearer <jwt>` on every call (root-level `bearerToken`, `bearerFormat: jwt`). Token acquisition is not documented by 1047 Games.
- On 2026-08-05 the declared host did not resolve. Verify DNS before assuming an outage is yours.
- You need a `playerId` (UUID). If you only have a gamertag, resolve it first with `searchPlayers` — see the player-lookup skill.

## Steps

1. **Page the history** — `getPlayerMatches`
   `GET /v1/game/splitgate2/players/{playerId}/matches?limit=25`
   Response: `items[]` of `PlayerMatch`, plus `nextPageAnchor` and `totalItemCount`.

   Each `PlayerMatch` is the player's-eye view: `matchId`, `joinedAt`, `leftAt` (ISO 8601 UTC), `playlistSlug`, `mapSlug`, `platform`, and an embedded `stats` block for that match.

   **Pagination rules.** `limit` defaults to 25 and is capped at 100. To take the next page, pass the previous response's `nextPageAnchor` back as `anchor`. Anchors are opaque and **only valid within a single query** — if you change `limit` or the player, start over from no anchor. Stop when `nextPageAnchor` is absent, not when a page comes back short.

   Do not fan out one request per page in parallel; cursor pagination is inherently sequential, and quotas are per-operation.

2. **Expand a match** — `getMatchDetails`
   `GET /v1/game/splitgate2/matches/{matchId}`
   Returns `MatchDetails`: `matchId`, `status`, `startedAt`, `endedAt`, `updatedAt`, `matchTypeSlug`, `gameModeSlug`, `playlistSlug`, `mapSlug`, and `teams[]`.

   `status` is one of `InProgress`, `Finished`, `Crashed`, `Abandoned`. Two fields are only meaningful once a match is finished: team `placement` and `endedAt`. Do not report a winner for a match that is `InProgress`, `Crashed` or `Abandoned`.

   `teams[]` is `MatchTeamDetails` — `teamId`, `placement` (1-indexed; **ties share a placement value**, so never assume placements are unique), and `members[]` of `MatchPlayerDetails` (`playerId`, `displayName`, `platform`, `stats`).

3. **Reconstruct a scoreboard**
   Sort `teams` by `placement` ascending, and within a team sort `members` by `stats.score`. Each member's `stats.summedOver.type` will be `Match` with the matching `matchId` — assert that before you attribute the numbers to this game.

   `getMatchDetails` is the operation the spec's own quota example names as capped per hour. Batch-expanding a long history is exactly the pattern that trips it: expand on demand, and cache by `matchId` (a `Finished` match is immutable, so `updatedAt` is a safe cache key).

## Errors

Same envelope everywhere — `reason`, `message`, `domain` (`Players` or `Matches`), `metadata`, `error.status`:

- `404` / `NotFound` on `getMatchDetails` — the match id is unknown; `metadata` echoes the `matchId`.
- `400` / `InvalidArgument` — inspect `error.violations[]`; a stale or cross-query `anchor` shows up here.
- `error.status: ResourceExhausted` — `error.violations[]` names the throttled `subject` path. No 429 code and no `Retry-After` are published; back off and slow the walk.
- `500` / `Internal`, plus modeled-but-unwired `Unavailable` and `DataLoss` statuses — treat as transient and retry with backoff. All operations are GET, so retries are safe.
