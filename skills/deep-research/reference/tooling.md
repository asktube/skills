# tooling — what this skill needs, and what the Claude Code names mean

`SKILL.md` and the phase files are written in **Claude Code's** tool vocabulary, on purpose: naming
the exact tool is what makes a run fast and reproducible, and diluting it into "use your browser
tool" costs more than it buys. This file is the translation layer. Read it if you are running this
skill on anything other than Claude Code.

## The two hard requirements

**1. The asktube MCP server**, authenticated as the user. Everything in `reference/asktube.md`
(`me_get`, `videos_search`, `videos_get`, `captions_get`, `channels_*`, `app_playlists_*`,
`captions_prioritize`, `captions_status`) is an MCP tool on that server. Tool *names* are the same
everywhere; only the way your client loads them differs.

**2. Browser automation driving the user's own logged-in Chrome.** This one is not negotiable and it
is the most common way a port of this skill fails silently:

- asktube's caption capture runs **in the browser**, in the asktube extension. MCP only *queues*;
  the drain happens in the tab. A headless or clean-profile Playwright has neither the
  `asktube_sess` cookie nor the extension, so `captions_prioritize` accepts the ids, nothing ever
  captures, and the poll simply times out with no error anywhere.
- The identity check in `reference/asktube.md` compares the MCP account against the *browser's*
  account. It is only meaningful if the browser you drive is the user's real one.
- The automation must also be able to run JavaScript in the page (the search-results extractor and
  `document.visibilityState` probe both need it) and to foreground a tab.

If your agent cannot drive the user's real Chrome, this skill can still run **library-only** — mine,
brief, and read whatever already has `captionStatus: done`. It cannot discover on YouTube or capture
new captions. Say so up front rather than discovering it at the capture timebox.

## Capability map

| Capability | Claude Code | Anywhere else |
|---|---|---|
| Load MCP tools | one `ToolSearch` call per group, e.g. `select:mcp__asktube__me_get,mcp__asktube__videos_search,…` | however your client enables MCP tools; the bare names (`me_get`, `videos_search`, …) are what matter |
| Browser automation | the `claude-in-chrome` MCP — `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `javascript_tool`, `computer` | any automation attached to the user's **own Chrome profile** with the asktube extension present — see above |
| Batched browser ops | `browser_batch` (every item carries an explicit `tabId`; the batch stops at its first error) | issue the calls sequentially |
| Foreground a tab | a `computer` screenshot does it | whatever interaction your tool has that makes the tab visible; verify with `document.visibilityState` |
| Ask the user | `AskUserQuestion` — once, per Invariant 10 | ask in prose, once |
| Parallel readers | sub-agents, one per sub-topic, **never onto the browser** | read the groups sequentially; the one-question-one-owner rule still applies, and so does "don't paste session history into the prompt" |
| Read a file | the `Read` tool | any file read |
| Invocation arguments | `$ARGUMENTS` | the user's request, verbatim |
| Waiting | a background `sleep` between polls | any non-tight-loop wait; poll every 5–10s |

## Oversized tool output

Claude Code persists a tool response above ~40KB to a file and returns the path plus a ~2KB preview.
The call *succeeded* — the payload is in the file, not the response. On a long `captions_get({ method: "cat" })`
that means: read the persisted JSON back, or grep it for the passages you need, rather than
concluding the transcript is short.

This is **harness behaviour, not asktube's**. asktube returns the whole transcript (recovering
oversized ones from its R2 archive). On another client the response may simply arrive whole; if
yours truncates differently, page with `head` and `nextFrom` — see
[asktube.md](asktube.md) → *Reading captions at length*.

## What this skill does not depend on

No YouTube Data API key. No other skill, plugin or marketplace. No network access beyond asktube and
youtube.com in the browser. If you find yourself reaching for one of those, the phase file is wrong
and that belongs in `retro.md`.
