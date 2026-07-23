# asktube — app playlists & captions

*Wire contract verified 2026-07-23 against `asktube@cd472fd`. This file is the single repair site when the API drifts — phases point here and never restate it.*

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

`no-captions-available`, `extension-error`, `timeout`, `not-released`, `player-not-found` are capture-error codes that appear in the **`error` field** — never in `status`.

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

## Identity — the MCP token and the browser must be the same account

**Check this before anything else.** Caption queues are per-account. If the MCP token belongs to account A and the browser is logged in as account B, `captions_prioritize` writes into A's queue while the browser's capture loop drains B's. Nothing captures, no error is raised, and the poll simply times out — the most expensive silent failure on this path.

1. **MCP side:** `me_get()` → `{ sub, email }`.
2. **Browser side:** from a tab on `https://app.asktube.xyz/**` (same origin, needs the session cookie), run via `javascript_tool`:
   ```js
   await fetch('/api/me', { credentials: 'include' })
     .then(async r => r.ok ? { ok: true, ...(await r.json()) } : { ok: false, status: r.status })
   ```
   `/api/me` is backed by the same read as `me_get`, so **`sub` is directly comparable**. It authenticates from the `asktube_sess` cookie only, so it always reports the *browser's* user.
3. **Compare `sub`** — it is the canonical id and `email` is optional in the schema. Report the **`email`** back to the user, though; that is the identity they recognise.

| Result | Meaning | Do |
|---|---|---|
| `sub` matches | good | continue |
| `{ ok: false, status: 401 }` | browser not logged in | stop; ask the user to log in at `https://app.asktube.xyz/ui/` |
| `sub` differs | **wrong account** | **stop before queueing anything** |

On a mismatch, report both identities and the two fixes — log into `app.asktube.xyz` as the MCP account's email, **or** re-issue the MCP token for the browser account — then stop. No amount of retrying fixes it, and proceeding burns the whole capture timebox.

*Fallback if `/api/me` is unreachable:* `localStorage.getItem('asktube-active-user')` holds the `sub`, but it is a cached mirror written after the page's own boot call and can lag — corroborate with it, never approve on it alone. Scraping `[data-testid="account-menu"]` yields the email only.

## Capture protocol — the order is load-bearing

Capture runs in the user's browser. MCP only queues; `https://app.asktube.xyz/ui/` must be open, foregrounded and logged in.

0. **Assert the tab is visible — before the toggle, every run.** Via `javascript_tool`: `document.visibilityState`. If it reports `"hidden"`, the tab is throttled and **drains nothing regardless of the toggle**. Foreground it — a `computer` screenshot does this — and re-assert until it reads `"visible"`. ⚠️ **In an MCP session a hidden tab is the *default* state, not an edge case**: the agent never looks at the tab, so nothing foregrounds it. Check this first rather than treating it as a stall remedy. *Observed:* 20 ids sat at `queued` with the toggle correctly on `priority`; a reload did not start the drain, and a screenshot did.
1. **Set the mode before queueing.** `tabs_context_mcp` first, then a new tab to `https://app.asktube.xyz/ui/` (the logged-in account shows bottom-left). The `/home` banner carries a 3-state toggle whose **label is the current mode**, cycling `Capturing all` → `Priority only` → `Paused` →. Read the state from `data-test-state` on `[data-testid="captions-mode-toggle"]` (`running` | `priority` | `paused`) rather than from a screenshot, and click until it is `priority`. `running` also drains the entire library backlog — slow and unwanted. `paused` drains nothing. The banner reading **"Downloading captions in the background"** is the positive confirmation that the drain has started.
2. **Wait for enrichment.** A just-added video has a playlist row but no `videos` row yet, and `captions_prioritize` **silently returns `accepted: []`** for it. Poll `app_playlists_videos({ id })` on a background `sleep` of 5–10s — never a tight loop — until every row has a non-null `title`.
3. **Queue** ≤50 ids per call and **verify `accepted` covers what you sent**; re-poll and re-send anything it dropped.
4. **Poll `captions_status`** every 5–10s. Capture is ~1–2s per video regardless of length. The "You're all caught up on captions" banner is **stale until you queue** — `captions_status` is authoritative.
5. **Timebox 4–5 minutes**, then proceed with everything `done` and prune the rest.

## Failure modes

- Items stuck `queued` → work the remedies **in this order**, cheapest and likeliest first:
  1. **Visibility.** `document.visibilityState === "visible"`? If not, foreground the tab (a `computer` screenshot does it). This is the most common cause by a wide margin — see protocol step 0.
  2. **Toggle.** Re-cycle it back to `priority` and re-interact between polls.
  3. **Reload.** Only if 1 and 2 didn't take. **Safe when every item reads `queued` and none reads `fetching`** — run that check first, every time.

  *Observed, run 2026-07-23:* 20 ids sat at `queued` with the toggle correctly on `priority`. The reload — applied first, and safe by its own test — did **not** start the drain; a screenshot did, and a probe taken right after the reload had reported `visibilityState: "hidden"`. An earlier note in this file credited a reload with clearing a 9-id stall; that was most likely the foregrounding effect of the interaction accompanying it, not the reload.
- **Never refresh mid-download** — in-flight items corrupt to `failed`. "Idle" has a precise test: `captions_status` shows **no item in `fetching`**. `queued` items are not in flight and survive a reload; that check is exactly what makes the reload above safe, so run it first, every time.
- Right after ingest, `videos_list` beats `videos_search` — the search index lags.
- `start_priority_capture` is a client-side tool and is **not reachable over external MCP**. The toggle is what drains the queue; don't go looking for a tool that does it.
