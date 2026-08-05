---
name: Look up a Splitgate player's stats and ranks
description: Resolve a Splitgate player from a display name to a player id, then read their career statistics and current competitive ranks from the 1047 Games Third-Party API.
api: openapi/splitgate-third-party-openapi-original.yml
operations: [searchPlayers, getPlayerStats, getPlayerRanks]
generated: '2026-08-05'
method: generated
source: openapi/splitgate-third-party-openapi-original.yml
---

# Look up a Splitgate player's stats and ranks

Use this when a user gives you a Splitgate gamertag and wants stats, a K/D, or a rank.

## Before you start

- Base URL: `https://api.1047games.com`
- Every call needs `Authorization: Bearer <jwt>`. The contract declares `bearerToken` (`http`/`bearer`, `bearerFormat: jwt`) at the root, so it applies to all five operations. **1047 Games does not publish how to obtain this token** — if you do not already have one, stop and tell the user that access is undocumented rather than guessing at a signup flow.
- **Availability check first.** On 2026-08-05 `api.1047games.com` did not resolve (dangling CNAME to `api-third-party-prod.maverick-global.prod.1047games.com`, NXDOMAIN). If your request fails at DNS, report that the provider's declared endpoint is down — do not retry in a loop.

## Steps

1. **Resolve the player** — `searchPlayers`
   `GET /v1/search/players?term={gamertag}&limit=10`
   `term` is required and matched case-insensitively against display name, or by exact player id. Results come back ordered by relevance in `items[]` as `{playerId, displayName}`, with `totalItemCount` and `nextPageAnchor`.
   Display names are not unique. If `totalItemCount > 1`, show the candidates and let the user pick — do not silently take `items[0]`.

2. **Read career stats** — `getPlayerStats`
   `GET /v1/game/splitgate2/players/{playerId}/stats`
   Returns a `SummaryPlayerStats`: `matchesPlayed`, `wins`, `losses`, `kills`, `deaths`, `suicides`, `assists`, `damage`, `score`, `xp`, plus `updatedAt`.
   Check `summedOver.type` before you label the numbers. It is a discriminated union — `Career`, `Season` (with `seasonSlug`), `GameMode` (with `matchTypeSlug`) or `Match` (with `matchId`). Never present a scoped stat block as a career total.
   Compute K/D yourself; the API does not return derived ratios. Guard against `deaths: 0`.

3. **Read current ranks** — `getPlayerRanks`
   `GET /v1/game/splitgate2/players/{playerId}/ranks`
   Returns `items[]` of `PlayerRank`: `matchTypeSlug` (e.g. `arenaranked`), `rankSlug` (e.g. `bronze`), `updatedAt`, and a `placement` that is one of two shapes:
   - `AbsolutePlayerRankPlacement` — `absolutePoints`, `absolutePlacement`, `maximumPlacement` (a ladder position).
   - `DivisionalPlayerRankPlacement` — `divisionPoints`, `divisionPointCeiling`, `divisionPlacement`, `maximumDivision` (a tier/division).
   Branch on which fields are present; there is no discriminator property on this union.

## Errors

Failures return the same envelope on every operation: `reason` (machine code), `message` (English, developer-facing), `domain` (`Players` here), `metadata` (string map), and `error.status`.

- `401` / `error.status: Unauthenticated` — missing or invalid JWT.
- `403` / `PermissionDenied` — token valid, not authorized for this resource.
- `404` / `NotFound` — player id does not exist. Re-run `searchPlayers`.
- `400` / `InvalidArgument` — read `error.violations[]` (`field`, `description`, `reason`) and fix the named field.
- `error.status: ResourceExhausted` — you hit a quota. `error.violations[]` gives `subject` (the throttled path) and `description`. **No 429 is declared in the contract and no rate-limit headers or `Retry-After` are published**, so back off on your own schedule; the spec's own example describes an hourly per-operation cap on match detail.

Localize on `reason`, never on `message` — `message` is explicitly not localized.
