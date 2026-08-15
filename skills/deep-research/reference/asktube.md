# asktube — app playlists & captions

*Wire contract verified 2026-07-23 against `asktube@cd472fd`. Capture protocol rewritten 2026-08-15 for backend-side capture; the wire table has not been re-verified since. This file is the single repair site when the API drifts — phases point here and never restate it.*

## Model

An **app playlist** is an app-maintained set of videos addressed by 11-character YouTube video ids. Adding a video is the **ingest trigger**: it queues metadata enrichment and puts the video in caption scope. Adding does *not* jump the caption queue — `captions_prioritize` does that.

## Wire contract

| Tool | Params | Returns |
|---|---|---|
| `me_get` | `{ include?: ("access"\|"usage")[] }` | `{ sub, email, createdAt }` — the account **this MCP token is bound to**. `access` adds `{ youtube, mcp }`; `usage` adds the action budget and library counters. |
| `app_playlists_create` | `{ title (1–200), description? (≤2000) }` | `{ item }` → `item.id` = `apl-…` |
| `app_playlists_list` | `{ query?, sort?, dir?, limit?, offset? }` | `{ items, total }` |
| `app_playlists_add_item` | `{ id, videoIds: string[] }` — 1–50, each `^[A-Za-z0-9_-]{11}$` | `{ ok: true, videoIds }` \| `{ error: "playlist_not_found" }` |
| `app_playlists_remove_item` | `{ id, videoId }` — one per call | `{ ok: true }` \| `{ error }` |
| `app_playlists_videos` | `{ id, page?, perPage? (≤500, def 50), sort? added\|updated\|published\|title, dir? }` | `{ items, page, perPage }` — each item is a VideoSummary (`id`, `title`, `duration`, channel fields, **`publishedAt`**, `captionStatus`) plus `addedAt`. A pending item lists with `title: null`; `publishedAt` lands with enrichment and is the **only exact publish date in this pipeline** |
| `captions_prioritize` | `{ videoIds }` 1–50 | `{ requested, accepted }` |
| `captions_status` | `{ videoIds }` 1–50 | `{ items: [{ videoId, inLibrary, status, chars, lines, error }] }` |
| `captions_get` | `{ videoId, method? cat\|head\|tail, len?, from? }` | `cat` (default) → whole transcript under `text`, no timestamps · `head`/`tail` + `len` → timestamped `{ start, text }` lines plus `totalLines`, `from`, `returned`, `hasMore`, `nextFrom` · both carry `complete` · no captions → `{ available: false }`. **Page with `nextFrom` — see [Reading captions at length](#reading-captions-at-length).** |

- A malformed id is a **schema validation error that fails the whole call**, not a per-item business error. Filter before sending.
- Caption status field is flat **`captionStatus`** on `app_playlists_videos` / `videos_list` / `videos_search`; nested **`caption.status`** (plus `caption.chars/lines/lang/error`) on `videos_get`.

## Caption status — the complete set

`none` · `queued` · `fetching` · `done` · `unavailable` · `failed`

- `done` → readable. `none` / `queued` / `fetching` → keep polling.
- `failed` → **transient**. Re-`captions_prioritize` once; it escalates to `unavailable` after 3 attempts.
- `unavailable` → no transcript exists. A prioritize does re-queue it and reset its attempt count, but it will re-land here. Retry at most once, then drop the video.
- `inLibrary: false` → the id is not in the user's scope at all.

`no-captions-available`, `timeout`, `not-released` are capture-error codes that appear in the **`error` field** — never in `status`.

⚠️ The deployed `captions_status` tool *description* is loose here (it lumps `failed` in with `unavailable`). The semantics above govern.

## Reading captions at length

`cat` returns the whole transcript in `text`. `head` / `tail` return timestamped lines, at most
**2,000 per window** (`MAX_CAPTION_LINES`), and every window reports what it actually gave you:

| Field | Meaning |
|---|---|
| `totalLines` | the true length of the transcript |
| `from` | index of `lines[0]` in the full transcript |
| `returned` | how many lines this window actually holds — a clamp shows up here, never silently |
| `hasMore` | there is transcript after this window |
| `nextFrom` | **the only correct next `from`**, or `null` |

**Page with `nextFrom`. Never compute `from + len`.** A clamp makes `len` a request, not a promise:
`from + len` then points past the lines you were given and skips everything in between, yielding a
transcript that reads as continuous prose with a hole in the middle of it. `nextFrom` accounts for
the clamp by construction, so paging on it cannot skip a line.

**`complete: false` means the transcript you hold is short.** A handful of very long videos exceed
what a D1 cell holds; asktube recovers those from the verbatim R2 archive, and when that archive is
missing or unreadable it says so rather than passing off a prefix as the whole thing. Carried on
`captions_get` (both arms) and on `videos_get`'s `caption.timestamped`. Treat it as a stated gap in
coverage — never let a missing tail read as an absent finding. The two stores cut at different
points, so a video can be whole on `cat` and clipped on `head`/`tail`, or the reverse.

This is the one failure Invariant 7 cannot catch: re-running with different vocabulary never surfaces
a hole *inside* a transcript you believe you already read.

## Capture protocol

Capture runs on asktube's backend. MCP queues it; the backend fetches it. **Nothing needs to be open in a browser, and no tab needs focus** — queue, then poll.

1. **Wait for enrichment.** A just-added video has a playlist row but no `videos` row yet, and `captions_prioritize` **silently returns `accepted: []`** for it. Poll `app_playlists_videos({ id })` on a background `sleep` of 5–10s — never a tight loop — until every row has a non-null `title`.
2. **Queue** ≤50 ids per call and **verify `accepted` covers what you sent**; re-poll and re-send anything it dropped. `accepted` is the only confirmation that the capture was requested.
3. **Poll `captions_status`** every 5–10s. It is authoritative — a UI banner is not, and the "You're all caught up on captions" one is stale until you queue.
4. **Timebox 4–5 minutes**, then proceed with everything `done` and prune the rest.

## Failure modes

- **Items still `queued` at the timebox** → backend queue depth, not something the client can fix. There is no toggle to flip and no page to reload. Proceed with everything `done`, prune the rest, and record them as gaps.
- **`accepted` came back short or empty** → you queued before enrichment landed. Re-poll `app_playlists_videos` until every `title` is non-null, then re-send the ids it dropped. Protocol step 1 exists for exactly this.
- Right after ingest, `videos_list` beats `videos_search` — the search index lags.
