# deep-research · Explore — mine → clarify → brief → discover → curate

Mine what the user already has, let it sharpen the search, then build a vetted corpus and get it into an asktube app playlist.

*Tool names below are Claude Code's. On any other agent, read `../reference/tooling.md` first.*

## Resolve once

Resolve all of these now. Nothing below re-derives them.

| Var | Resolution |
|---|---|
| `SLUG` | the slug in `$ARGUMENTS` if it names a `deep-research/*` folder; else kebab-case of TOPIC |
| `DIR` | `deep-research/<SLUG>/` |
| `BRIEF` | `<DIR>brief.md` — read it if it exists (this is a resumed or extended run); otherwise you write its Frame in step 3 |
| `AMENDMENT` | anything in `$ARGUMENTS` past the slug ("go wider on pricing", "official sources only") — append to `BRIEF` as a dated amendment, then honour it |
| `PLAYLIST_ID` | the `apl-…` recorded in `BRIEF` → else `app_playlists_list({ query })` if PROCESS names one to reuse → else created in step 6, and written back into `BRIEF` |
| `SIZE` | corpus size from PROCESS. Where PROCESS is silent, **derive it from the question set**: enough that every brief question has at least one source addressing it directly, and every *quantitative* question has two independently-produced ones. 15–25 is the fallback before the brief exists. Do **not** scale the corpus to the length of the deliverable — reading fifteen sources to write 500 confident words is the correct trade, not waste. |

**Tools — two `ToolSearch` calls, up front.**

- asktube: `select:mcp__asktube__me_get,mcp__asktube__videos_search,mcp__asktube__videos_list,mcp__asktube__videos_get,mcp__asktube__captions_get,mcp__asktube__channels_list,mcp__asktube__channels_search,mcp__asktube__app_playlists_list,mcp__asktube__app_playlists_create,mcp__asktube__app_playlists_add_item,mcp__asktube__app_playlists_videos`
- Chrome: `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__browser_batch`

Call `tabs_context_mcp` **first** and work in a **new** tab — never the user's. Inside a `browser_batch` every item needs an explicit `tabId`, and the batch stops at its first error. If the tab group disappears, a standalone `navigate` recreates it and returns fresh ids.

## Output of this phase

`<DIR>corpus.md` — the decision doc a cold Research phase starts from. A reader must be able to answer: which videos are we studying and **why each one** (one line: authority / specificity / recency) · which were **already captioned** in the library vs newly curated · the app-playlist id and its videoIds · which brief question the corpus does **not** yet cover, and what was tried · what recency and vetting policy applied and what it excluded. Rank best-first; a table is usually right. Nothing else is prescribed.

## 0. Verify identity — first, and it can stop the run

Caption queues are per-account, so the MCP token and the browser session must be the **same** asktube account. A mismatch makes the entire capture phase fail silently, and it is cheapest to find now — before discovery, not after.

Follow **Identity** in `../reference/asktube.md`: `me_get()` for the MCP side, `fetch('/api/me')` from an `app.asktube.xyz` tab for the browser side, compare `sub`.

⚠️ **Run this alone — nothing else in the same block.** Do not batch `videos_search`, `channels_*` or any other library call alongside it to save a round trip. Library contents are per-account: mining against an unverified account yields a corpus you must discard wholesale, and a wrong library looks entirely plausible while you're reading it. The calls only *look* independent (SKILL.md, Invariant 9).

If they differ, or the browser isn't logged in: **report both identities, name the two fixes, and stop.** This halt is exempt from the one-`AskUserQuestion` budget, so prefer stopping *with options* over stopping flat — "fix the account" and "run library-only, no capture needed" are both real paths, and the user may choose a third. Nothing you can do resolves a wrong account.

## 1. Mine the library

The user's captioned library is free, instant, already indexed, and it is the best query-tuning signal available. It only needs TOPIC, so it runs before the questions get asked. Search at three widths, 5–8 searches, timeboxed:

- **Exact** — `videos_search({ query, sort: "relevance", limit })` on the topic's own words.
- **Adjacent** — the field the topic sits inside. A "Product Hunt launch" goal is also served by "go to market", "SaaS launch", "first 100 users". Run adjacency **even when the exact search hits** — this is where mining pays.
- **Authority** — `channels_list` / `channels_search` for credible sources the library already follows here. ⚠️ **`channels_search` ANDs its terms** across title, handle and description — search **one word at a time** (`SEO`, then `marketing`, then `growth`), never a topic phrase. *Observed:* `"SEO marketing indie hacker startup growth"` returned **0** against a 956-channel library while `"SEO"` alone returned **8**, including the run's single most important primary source. A multi-word query returning nothing is a fact about the query, not the library. Pass `trackStatus: ["tracked","untracked"]` — the authority channel you want is often one the user follows but never indexed.

⚠️ **Caption full-text search is high-precision, low-recall.** Unlike web search it rewards the exact phrase and punishes conceptual paraphrase. On one run `recursive language models RLM` returned 9 hits while the paraphrase `long context window REPL agent decomposition` returned 2 — and the paraphrase missed the primary source entirely. Run the topic's own words verbatim *before* you get clever, and read a paraphrase's silence as a fact about the query, not about the library.

On the two or three strongest hits, `videos_get` for `caption.status` and `captions_get({ videoId })` (method `cat`) to hear how the field actually talks. This is reconnaissance, not the deep read.

If the library returns nothing relevant, **say so in one line and move on**. That's a normal outcome for a topic the user has never watched, not a failure — and it's a brief fact worth recording, because the coverage section will report the corpus as 100% newly discovered. Don't invent adjacency to make the step feel productive.

## 2. Clarify — one round, gaps only

Now you know what the library holds, so ask a better question. One `AskUserQuestion`, covering **only** slots `$ARGUMENTS` left open whose answer would change the run. If the args settled everything, **ask nothing and continue**.

Worth asking when open: the recency policy, which sub-topics matter most, and the output shape.

⚠️ **Recency guardrail.** Hybrid is the default — fresh for fast-moving tools and tactics, evergreen allowed for durable fundamentals. On an **evergreen** topic, strict-recent (≤1 yr) cuts the canonical sources, which are typically 2–4 years old. Say so before the user picks it.

**Don't ask about recency when the topic postdates every plausible cutoff.** If the canonical work is months old, no policy excludes anything and the question burns the one slot you get. Settle it yourself and record the reasoning in the brief — including the consequence worth stating in coverage: a corpus with no old sources cannot show how the claims aged.

## 3. Write the Frame → `BRIEF`

TOPIC and the decision it enables · the user's starting level · **the 5–10 questions the finished deliverable must answer** · recency policy · what is explicitly out of scope · PROCESS and OUTPUT copied verbatim where the user was specific.

The questions are the acceptance test for the whole run. Writing them *before* any search is the point — they can't be retrofitted to whatever you happened to find. Make them answerable and specific ("what does a realistic day-one traffic number look like?", not "understand launch traffic").

`brief.md` is append-only: later learning becomes a dated amendment at the bottom, never an edit to the top.

## 4. Turn the mining result into a query plan → append to `BRIEF`

This is why mining ran first. Write these five things under `## Query plan`:

- **Vocabulary** — the terms practitioners actually use in these titles, and the homonyms to steer away from ("product hunt" collides with e-commerce "product hunting"). Queries use the former and disambiguate against the latter.
- **Coverage** — sub-topics the library already answers (search shallowly or not at all) and the ones it doesn't (search hard). Effort is allocated against this, not spread evenly.
- **Authority channels** — search *within* them (`<channel> <sub-topic>`) rather than hoping generic relevance surfaces them.
- **Seeds** — library videos already `done` on captions that belong in the corpus. They cost nothing and are readable immediately, with no capture wait.
- **Queries** — 5–8 concrete search strings, each tagged with the brief question it exists to fill.

## 5. Discover + vet

Run the brief's queries, **relevance-sorted** (`&sp=EgIQAQ%253D%253D` = Type→Video). One `browser_batch` per query: `navigate` then `javascript_tool`, both carrying the explicit `tabId`. The extractor below is the only tool that returns video **ids**; use the text tools only for reading a single page's prose.

```js
const N=20, SCROLLS=1, SEL='ytd-video-renderer, yt-lockup-view-model';
const w=ms=>new Promise(r=>setTimeout(r,ms));
for(let i=0;i<25&&!document.querySelector(SEL);i++){await w(300);}
for(let i=0;i<SCROLLS;i++){window.scrollTo(0,document.documentElement.scrollHeight);await w(900);}
const rows=[...document.querySelectorAll(SEL)].map(v=>{
  const a=v.querySelector('a#video-title, a#video-title-link, a.yt-lockup-metadata-view-model__title');
  const id=((a?.href||'').match(/[?&]v=([^&]+)/)||[])[1]||'';
  const chA=v.querySelector('ytd-channel-name a, .yt-content-metadata-view-model__metadata-row a');
  const m=[...v.querySelectorAll('#metadata-line .inline-metadata-item, .yt-content-metadata-view-model__metadata-text')].map(s=>s.textContent.trim());
  return {title:(a?.title||a?.textContent||'').trim(), id,
    ch:(chA?.textContent||'').trim(), chUrl:chA?.href||'',
    views:m.find(t=>/view|watching/i.test(t))||'', age:m.find(t=>/ago/i.test(t))||'',
    dur:(v.querySelector('ytd-thumbnail badge-shape .badge-shape-wiz__text, ytd-thumbnail-overlay-time-status-renderer #text')?.textContent||'').trim(),
    desc:(v.querySelector('.metadata-snippet-text, #description-text')?.textContent||'').trim()};
}).filter(x=>x.id);
// Sentinel — a surviving continuation renderer alongside a short harvest means
// YouTube's infinite-scroll fetch never resolved and you hold page one only.
({truncated: rows.length<N && !!document.querySelector('ytd-continuation-item-renderer'),
  count: rows.length, rows: rows.slice(0,N)});
```

`N` = rows kept per query, `SCROLLS` = load depth. `1/20` is fast; `3/40` favours recall — raise both when the brief asks for breadth.

- An **empty array is a failure signal, not an empty topic** (selector drift or a load race). Re-run once with different vocabulary, then report — never proceed on a silent zero.
- ⚠️ **`truncated: true` is the more dangerous failure, because it returns plausible data.** The continuation fetch silently failed and you are holding YouTube's first page. *Observed:* every query returned exactly **4 rows** at `1/20`, at `4/40`, and with an accumulator that collected across incremental scrolls; forcing it harder killed the renderer (`Runtime.evaluate timed out after 45000ms`). A run that trusts this curates a 4-per-query corpus and then reports a thin topic that is in fact enormous.

  **The remedy is more queries, not more scrolling.** Page one is a *sample* of one phrasing, so widen the query set — the vocabulary variants and authority-channel searches already in your Query plan — until the same sources start recurring across different phrasings. That recurrence, not row count, is what tells you the topic is covered. Then say so in coverage: discovery was page-one per query, N phrasings.
- Empty `desc` is normal; YouTube shows snippets on only some results.
- **Recency is decided after ingest, not here.** The DOM reports coarse buckets (`"1 year ago"`) that cannot decide a month-precision rule, and it fails *silently*: a source displayed as "1 year ago" is anywhere from 12 to 23 months old. *Observed:* two pricing sources read "1 year ago" and were actually **17 and 23 months** old, both outside the brief's window — presented as current prices with nothing flagging it. So treat the bucket as a **pre-filter only** (for strict-recent, add `&& !/year/i.test(x.age)` to drop anything a year or older — YouTube's own "This year" UI filter is calendar-year and silently drops the trailing months). The exact date arrives free at step 6: ingest enriches every added video with a real `publishedAt`, and that is what the recency policy is enforced against.
- **Vet on** title + `desc` + channel track-record + recency, judged against the brief's vocabulary and coverage map. Drop homonyms, promo clips, thin content. Favour recall — the caption read is the real filter — but every added video costs a capture slot and a read, so hold to `SIZE`.
- **Cap sources per thesis at two.** Invariant 5 needs *two independently-produced* sources to quantify a claim — a third and fourth restating the same argument buy nothing and cost a capture slot, an agent's reading time, and a row in the watch order you will end up marking "skip". *Observed:* four videos arguing one "drop the paid tool, wire up the raw API yourself" thesis were all curated, and all four were ultimately marked skip. Keep the two strongest, log the rest in `corpus.md` as *same-thesis, dropped for redundancy*. **The exception is a contested claim** — where sources actively disagree, extra voices are evidence about the spread, so keep them and say why.
- **Tag every keeper with a source tier**, now, while the channel is in front of you: `primary` (the author, their institution, the vendor shipping the thing) · `practitioner` (built with it, reporting their own results) · `secondary` (human-made explainer) · `ai-narrated` (synthetic voice, stock footage, no primary reporting — increasingly the *majority* of results on any new technical term). This is what Invariant 5 keys off later: an `ai-narrated` source may **corroborate** a number but may never be the only source for one. Carry the tier into `corpus.md`. Keeping a couple of good `ai-narrated` explainers is fine and often useful — they are frequently the only ones that bother to tabulate benchmark results — but they are corroboration, never authority.
- *Optional, for suspiciously high-view candidates only:* read the like count from the like button's `aria-label` (`[...document.querySelectorAll('button')].map(b=>b.getAttribute('aria-label')||'').find(l=>/like this video/i.test(l))`) → like/view >3% is strong; high views with <0.5% likes is botted, exclude.
- The logged-in related/recommended sidebar is **personalized** — use it to fill a sub-topic no query reached, never as primary discovery.
- Counts are abbreviated (`1.2M`, `12K`); approximate is fine, invented is not.

## 6. Curate into the app playlist

`PLAYLIST_ID` unset → `app_playlists_create({ title: "<TOPIC>", description: "<goal, one line>" })`; keep `item.id`.

Extract the **11-character videoId** from each vetted URL yourself — the API takes ids only. Then add in **batches**: `app_playlists_add_item({ id: PLAYLIST_ID, videoIds: [...] })`, 1–50 ids per call. Filter out anything not matching `^[A-Za-z0-9_-]{11}$` first — a malformed id fails schema validation and kills the whole call.

Dedup against `app_playlists_videos({ id })` first, seeds included — they may already be members. **Adding is the ingest trigger**; it does not jump the caption queue. See `../reference/asktube.md` (relative to this file) for the full contract.

### Enforce the recency policy here, on real dates

Ingest enriches each added video with an exact `publishedAt`. That is the only date in this pipeline you can decide a month-precision rule on — the search-page buckets you vetted against cannot (see step 5).

Poll `app_playlists_videos({ id })` until titles are non-null (the same enrichment wait capture needs anyway), read `publishedAt` off each item, and `app_playlists_remove_item` anything outside the brief's window, **before** capture spends a slot on it. Record each drop in `corpus.md` with its real date and the bucket that mis-sold it. If the policy is qualitative ("roughly recent"), skip the prune and say in coverage that recency is bucket-precision.

Write `<DIR>corpus.md` (source tiers included) and record `PLAYLIST_ID` in `BRIEF`.

Then: **your next action is `Read` on `../phases/research.md`.** Not a `captions_prioritize`, not a capture, not another search. You have not entered Research until you have read it — and the pull to skip it is strongest exactly here, with a playlist freshly built and capture the obvious next move. Do not stop, but do not improvise the phase either; it is already written.
