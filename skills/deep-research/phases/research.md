# deep-research · Research — capture → read → deliver

Get the curated corpus readable, read it into evidence, and write the deliverable.

*Tool names below are Claude Code's. On any other agent, read `../reference/tooling.md` first.*

## Resolve once

Resolve all of these now. Nothing below re-derives them.

| Var | Resolution |
|---|---|
| `SLUG` | the slug in `$ARGUMENTS` if it names a `deep-research/*` folder; else the newest one |
| `DIR` | `deep-research/<SLUG>/` |
| `BRIEF` | `<DIR>brief.md` — **read in full**. It is the scope, the questions, the recency policy, the PROCESS and OUTPUT directives. Do not re-derive any of them. |
| `PLAYLIST_ID` | the `apl-…` in `BRIEF` or `<DIR>corpus.md`; else `app_playlists_list({ query })` |
| `CORPUS` | `app_playlists_videos({ id: PLAYLIST_ID })` — the authoritative video set |
| `SHAPE` | the OUTPUT directive in `BRIEF`, overridden by any shape wording in `$ARGUMENTS`; default = a deep-research report |
| `AMENDMENT` | anything in `$ARGUMENTS` past the slug — append to `BRIEF` as a dated amendment, then honour it |

**Precondition: none in the browser.** Caption capture runs on asktube's backend — MCP queues it and `captions_status` reports it. Nothing has to be open, focused or logged in anywhere for a capture to complete. This phase needs no browser tools at all.

**Tools — one `ToolSearch` call:** `select:mcp__asktube__app_playlists_videos,mcp__asktube__app_playlists_remove_item,mcp__asktube__captions_prioritize,mcp__asktube__captions_status,mcp__asktube__captions_get,mcp__asktube__videos_get,mcp__asktube__videos_list`

## 1. Capture

Follow `../reference/asktube.md` (relative to this file) exactly: wait for enrichment (non-null titles) → `captions_prioritize` and **verify `accepted`** → poll `captions_status` → timebox and prune. Don't restate the protocol here; if it misbehaves, that's a `retro.md` entry.

⚠️ **The enrichment wait is the one ordering that matters.** `captions_prioritize` silently returns `accepted: []` for a video whose `title` is still null, so queueing early looks like it worked and captures nothing. Verify `accepted` against what you sent, every time.

**The capture wait is not dead time.** Seeds carried over from the library are already `done` and need no capture — start the deep read on them while the queue drains, and poll `captions_status` *between reads* rather than between sleeps. On a corpus that is half seeds this takes the entire capture window off the critical path.

Then close the corpus: proceed with everything `done`, `app_playlists_remove_item({ id, videoId })` for each video never captured (one call per video), and record every drop with its reason in `<DIR>corpus.md` — so the playlist equals the readable corpus.

## 2. Read — how you research this is up to you

PROCESS in the brief is the user's instruction on method; follow it. Where it's silent, choose. What follows is what has worked, not a procedure you owe anyone.

Read the whole transcript before pinning any timestamp. `captions_get({ videoId })` (method `cat`, the default) returns the whole transcript as one plain-text string under `text` — cheapest, no timestamps. `method: "head"` / `"tail"` with `len` return timestamped `{ start, text }` lines; convert `start` seconds to `[mm:ss]`. Understand first, then go back for the seconds.

**If a transcript turns out not to be English, stop reading it.** Drop the video, and record it in `<DIR>corpus.md` as a language drop that discovery vetting missed — asktube has no multi-language support yet, so there is nothing to salvage. This costs nothing: you already hold the text.

⚠️ **Page with `nextFrom`, never `from + len`**, and treat `complete: false` as a stated gap rather than an absent finding. Both rules and why they exist are in **Reading captions at length** in `../reference/asktube.md` — read it before paging an hour-long transcript.

Fan out with sub-agents when the corpus splits into groups that can be understood independently — one per sub-topic, **never onto the browser**. Give each agent exactly: its videoIds, the brief questions it owns, the invariants copied verbatim, and one output instruction — write its file under `<DIR>notes/`, return only the path plus a short summary and any concerns. Do not tell it what it should find. Do not paste this session's history into its prompt.

**One question, one owner.** Agents are split by *video group*; questions must be split too. Sharing a **video** between agents is fine and often necessary. Sharing a **question** means two agents write the same section from different evidence, and you discover it only when the Ledger has one question pointing at three notes with three different answer tables. *Observed:* Q3 was answered in three notes and Q4 in two, with non-matching threshold tables. Assign each brief question to exactly one agent and say so in the prompt. If a question genuinely needs two independent passes — because you *want* the spread on a contested claim — make that a deliberate instruction ("agent B re-derives Q4 independently; disagreement with agent A is the finding"), not an accident.

**Don't poll `notes/` — it is empty by construction until an agent finishes**, because each writes one file at the end. You are notified on completion; that is the signal. *Observed:* ~15 background waits, several of 120s, against agents that finished in 1-5 minutes.

**The agent wait is the largest idle block in the run — spend it reading.** Longer than the capture wait, and unlike capture there is nothing to poll. Use it to build your own grounding on the sources the deliverable will lean on hardest: the primary source for each quantitative question, and anything you already expect to quote. That work is not duplicated effort — it is the verification pass below, done early, and it is what lets you start writing before the last agent returns.

Depth follows value, not fairness. A video carrying hard specifics earns a long note; one that turns out to be 90% off-topic, or a funnel for the creator's paid program, earns **one line** saying so and is dropped from the citations. Never write a full block for a video you're about to tell the reader to skip — and never promote an idea into the deliverable on the strength of a video you rated skip.

### What good notes look like

Concrete over thematic: the exact number, the exact wording, the sequence someone actually ran and the result they got — each with its `[mm:ss]`. "Distribution is the bottleneck" is worth nothing. "Ranked #7 with no award, still got both newsletters, his highest-ever signups `[36:48]`" is worth the whole read.

You choose the files under `<DIR>notes/` and their internal shape — one per theme, one per video, one big file, whatever the material wants. **Default to one section per brief question** unless the material clearly wants otherwise: it makes the Ledger mechanical to write and turns the brief into a checklist you can actually tick. Organising by video instead tends to scatter the answer to question 4 across five files, and you find out only when writing the Ledger.

Record **which claims are single-sourced and which are corroborated** as you go — Invariant 5 decides what may enter the deliverable, and reconstructing that at write-up time means re-reading everything. A one-word tag per claim is enough.

### When the transcript points at the screen

Phrases like *"here's the data"*, *"look at these numbers"*, *"you can see"*, *"I'll give you a minute to look at those"* — followed by no spoken figures — mean the evidence is **visual**, and the caption is a pointer rather than the content. This is not rare: demonstration-heavy videos (tool reviews, benchmark tests, screen-shares) put numbers in the *frame* and use the audio for narration, which is exactly the genre a corpus about software or tooling is made of. A caption-only pipeline systematically under-reads its best sources.

- **Never let a missing table read as an absent finding.** Record the timestamp and the *shape* of what's there — "8-row accuracy table, values not spoken" — as an explicit gap in the note.
- **Escalate when the claim is load-bearing** for a quantitative brief question: go find a source that *speaks* its numbers rather than shipping n=1 where n=8 was on screen. If none exists, the finding is "the comparison exists but is unreadable from captions" — say that, with the timestamp, and don't quantify from it.
- *Observed:* an 8-article accuracy test narrated exactly one row, and a 3-keyword × 4-tool comparison spoke none of its scores. The run's entire error-magnitude claim ended up resting on a single keyword.

### Verify before you lean

`captions_get` is cheap; a wrong timestamp or a mis-transcribed number in the deliverable is not. **Before writing, re-read the raw caption for every citation carrying a number into the deliverable** — prices, benchmarks, thresholds, dates, index sizes. Ten `head` calls is the whole cost.

This is not distrust of the sub-agents; it is that **YouTube auto-captions are demonstrably unreliable on exactly the tokens that matter**. Real examples from one run: one vendor's product name rendered as a rival's, a price whose leading digits were lost (`"starting at9 Euros"`), proper nouns mangled beyond recognition, and the same statistic attributed to two different populations within one video. Agents that flag these are working correctly — but only the write-up stage knows which of them the deliverable actually depends on.

Only these properties are fixed: **grounded** (every non-obvious claim carries its videoId + a real `[mm:ss]`) · **proportional** · **non-redundant** (if you're about to write it twice, write it once and link) · **specific** · **never empty**.

## 3. Ledger → append to `BRIEF`

For each brief question: **answered** / **partly answered** / **unanswerable from this corpus**, one line each, with the note file that answers it. That ledger is the deliverable's coverage section, and the only honest measure of how thin the run was.

## 4. Deliver

One document at `<DIR>report.md`, or the name `SHAPE` implies. **There is no template.** A research report, a 4-week study plan, a ranked watch order, a one-pager and a checkbox tracker are all valid. Adapt to `SHAPE`; never pad to fill it.

With no directive, the default shape is a **deep-research report**: the topic framed in a paragraph → the brief's questions answered in order, each with its grounded citations and watch verdicts → what changed your mind or contradicted the consensus → the coverage section. When the goal is learning, the roadmap shape — dependency-ordered steps, a named first move, a "done =" per step — is first-class; reach for it when the user is trying to *learn* the topic rather than *understand* it.

`captions_get` is licensed and encouraged here: **re-read the raw caption for anything you're about to lean on.** A deliverable written only from your own summaries is a summary-of-a-summary, and that is where invented specifics come from.

Then re-read the **output contract** in `SKILL.md` and check the deliverable property by property, the brief's questions included: each answered, or explicitly marked unanswered with the reason from the Ledger.

**Verify any quantitative part of `SHAPE` instead of eyeballing it.** "500 words", "10 items", "one page" are all checkable — so check. A deliverable that runs 780 words against a 500-word directive has missed an instruction, not made a stylistic choice. Count citations out of the body word count if the user asked for prose length.

## 5. Close out

Report in ≤5 lines: the path, the single most useful thing found, the biggest coverage gap, and what to do next.

If the run hit anything this skill got wrong — a stale tool contract, a mechanic that didn't behave as documented, a step that wasted time — append it to `<DIR>retro.md` with what you observed and what the fix would be. That file is how this pipeline gets fixed.
