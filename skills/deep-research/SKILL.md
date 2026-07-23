---
name: deep-research
description: >-
  Deep research on any topic, grounded in YouTube captions the user actually holds via asktube.
  Use when someone wants a researched answer built from real video sources — "research X", "what's
  the current thinking on X", "go deep on X", "build me a learning roadmap for X" — or wants to
  extend, deepen or reshape an earlier run in deep-research/. Mines the user's asktube library,
  tunes YouTube discovery from what it finds, curates a corpus, reads the transcripts, and writes
  the deliverable in whatever shape was asked for.
argument-hint: "<topic> [+ directives: how to research it, what shape the output should take]"
license: MIT
compatibility: >-
  Needs the asktube MCP server and browser automation driving the user's own logged-in Chrome
  (the asktube extension does the caption capture). Written for Claude Code — reference/tooling.md
  maps every Claude Code tool name to the capability it stands for.
metadata:
  version: "0.1.0"
---

# deep-research

Deep research over YouTube, grounded in transcripts rather than recall. Discovery runs in the
browser, against YouTube itself; the corpus, its metadata and its captions live in **asktube**. No
API keys anywhere. Every claim in the deliverable traces to a video id and a real timestamp.

## The request

**Raw request:** $ARGUMENTS

*(If your agent doesn't substitute `$ARGUMENTS`, the raw request is whatever the user just asked for, verbatim.)*

Parse it into four slots, restate them back in ≤4 lines, then start. Don't wait for confirmation.

| Slot | What it captures | If unspecified |
|---|---|---|
| **TOPIC** | the subject, who it's for, the decision or understanding it must enable | required — if the request names no topic, ask; this is the only blocking gap |
| **PROCESS** | *how* to research: depth, recency, sub-topics to force in or out, source bias (official vs practitioner), corpus size, `into app playlist <name>` to reuse one, "my library only" | phase defaults |
| **OUTPUT** | *what shape* the deliverable takes: "a 4-week study plan", "a one-pager", "a checklist I can paste into Linear", "just the shortlist" | a deep-research report |
| **RESUME** | "resume at research", "re-render as a one-pager", "add 10 more videos on X", a bare slug | fresh run |

**Directives beat defaults. Directives never beat the invariants or the output contract.**

## Invariants

These hold no matter what PROCESS or OUTPUT say.

1. **Never fabricate.** Stats, video ids, quotes, channel names, timestamps — all come from tools.
2. **Timestamps come from real caption `start` values**, converted to `[mm:ss]`. Check the conversion: seconds are 00–59.
3. **Cite only what you read.** A video may be cited only if its captions reached `done` and you read them. A video added but never captured is a *gap*, not a source.
4. **Every non-obvious claim carries `(videoId, [mm:ss])`.**
5. **Corroborate before quantifying.** A specific number, benchmark result, price or date enters the deliverable only if it comes from a **primary** source (the author, the institution, or the vendor) **or** two independently-produced sources agree. Otherwise it stays in `notes/`, named as single-sourced in the coverage section. Grounding proves a claim was *said*, not that it is *true* — an AI-narrated summary with a real timestamp is still an AI-narrated summary, and on YouTube that is now the likeliest way a wrong number reaches the page.
6. **State coverage honestly** — what the corpus doesn't cover, which videos were dropped and why, which recency policy applied and what it excluded. "Found nothing credible on X" is a real, publishable result.
7. **Before declaring a gap, rule out a tooling artifact.** An empty harvest, a stalled capture queue and a genuinely thin topic look identical. Re-run with different vocabulary and check `captions_status` before writing "little content exists on X". A **small** result set is not an empty one — on a niche topic, four good hits per query is the signal, not a failure.
8. **Never write an empty file, section or directory.** Create folders on first write, not up front.
9. **Sequence anything whose result you'd have to discard.** A call is only independent of another if you would keep its output regardless of how the other resolves. Library contents are per-account, so `videos_search` is *not* independent of the identity check even though the two look unrelated. Batching pressure — from the harness or from your own instinct to save a round trip — is not a reason to collapse a real dependency.
10. **Autonomous.** At most one `AskUserQuestion`, in Explore, before the brief is written — and only for a slot `$ARGUMENTS` left open whose answer would change the run. After that there are no approval checkpoints: finish and report. The one exception is a precondition only the user can satisfy — notably an asktube **identity mismatch** (Explore step 0). That halt does **not** count against the one-question budget, and the better move is to stop *and offer the options you can see* (fix the account, or run library-only) rather than stop flatly — the user may pick a path you didn't list.

## Output contract

OUTPUT decides shape, format, length and file names. It does not relax these properties:

- **Answers the brief** — every question in the brief's Frame answered, or explicitly listed as unanswered with the reason.
- **Grounded** — every non-obvious claim carries its video id and timestamp.
- **Each cited source carries a watch verdict** — `watch in full` / `skim <range>` / `skip — the notes are enough`.
- **A coverage section** — corpus size, recency policy and what it excluded, drops with reasons, sub-topics nothing credible covered.
- **Portable Markdown** unless OUTPUT says otherwise: relative links, `- [ ]` for anything checkable, no tool-specific syntax.
- **No layer restates another.** If a document would mostly re-render one you already wrote, merge them instead.

When the requested shape is a **roadmap or plan**, it additionally must be dependency-ordered, name a single **first move**, and give each step a "done =" a reader can evaluate without asking you.

## Output location

`deep-research/<SLUG>/` in the current working directory, `SLUG` = short kebab-case of TOPIC. Nothing is scaffolded ahead of use:

```
deep-research/<SLUG>/
├── brief.md    append-only — Frame (Explore) → Query plan (Explore) → Ledger (Research) → Amendments
├── corpus.md   which videos, why each, what was captured, what was dropped, the apl- id
├── notes/      the evidence layer — your own structure, created on first write
├── report.md   the deliverable (or whatever name the OUTPUT shape implies)
└── retro.md    only when a mechanic misbehaved — this is how this skill gets fixed
```

## Phases

Read the phase file and follow it, then continue to the next. Each is written to be entered cold.

⚠️ **A phase is not entered until its file has been read in this session.** Carrying execution
momentum out of one phase and straight into the next one's tool calls is the most common way the
second phase file gets skipped entirely — the run then improvises a process that already exists in
writing. If you are about to capture captions and have not read `research.md`, stop and read it.

1. **Explore** → [phases/explore.md](phases/explore.md) — mine the asktube library, clarify what's still open, write the brief, tune the YouTube search from what mining found, vet, curate into an app playlist.
2. **Research** → [phases/research.md](phases/research.md) — capture the captions, read them, write the deliverable.

The asktube wire contract and caption-capture protocol live in [reference/asktube.md](reference/asktube.md); the phases invoke it rather than restating it.

Tool names throughout are **Claude Code's**. On any other agent, read [reference/tooling.md](reference/tooling.md) first — it maps each one to the capability it stands for and names the one substitution that is not negotiable (browser automation must drive the user's own logged-in Chrome).

Live Chrome is one shared browser: **browsing is serial — never fan sub-agents onto it.** Sub-agents are for stateless work (reading captions, ranking captured text).

On RESUME, enter at the named phase and skip the earlier one.
