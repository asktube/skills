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

**2. Browser automation, for discovery on youtube.com.** Discovery is the *only* thing the browser
does here. Caption capture runs on asktube's backend — `captions_prioritize` queues it and
`captions_status` reports it — so nothing has to be open, focused or logged in for a capture to
complete.

- The automation must be able to run JavaScript in the page: the search-results extractor in
  `phases/explore.md` is the only thing that returns video **ids**, and it reads the rendered DOM.
- Driving the user's own Chrome is **preferred, not required**. YouTube meets clean profiles with
  consent and bot interstitials, and the related/recommended sidebar is personalized — a fresh
  profile gets a different, usually worse, page. A headless or clean-profile Playwright will work if
  it gets past those.

If your agent cannot drive a browser at all, this skill still runs **library-only** — mine, brief,
and read the library. It cannot discover on YouTube. It *can* still capture: anything already in the
library, or added to a playlist by id, captures backend-side with no browser involved. Say so up
front rather than discovering it mid-run.

## Capability map

| Capability | Claude Code | Anywhere else |
|---|---|---|
| Load MCP tools | one `ToolSearch` call per group, e.g. `select:mcp__asktube__me_get,mcp__asktube__videos_search,…` | however your client enables MCP tools; the bare names (`me_get`, `videos_search`, …) are what matter |
| Browser automation | the `claude-in-chrome` MCP — `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `javascript_tool` | any automation that can load a URL and evaluate JavaScript in the page; the user's own Chrome profile is preferred — see above |
| Batched browser ops | `browser_batch` (every item carries an explicit `tabId`; the batch stops at its first error) | issue the calls sequentially |
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
