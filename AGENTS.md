# AGENTS.md — the contract for this repo

This repo is **one layout serving two installers**. Claude Code reads
`.claude-plugin/marketplace.json` and installs a plugin; the `skills` CLI (`npx skills`) reads the
*same* file to find the skills and to group them under the plugin name. Nothing is duplicated, and
nothing is generated. Keep it that way.

```
.claude-plugin/marketplace.json   the catalogue — Claude Code AND npx skills both read this
skills/<name>/SKILL.md            the skill; the folder is the unit of distribution
skills/<name>/phases/             multi-step skills split their steps out here
skills/<name>/reference/          wire contracts, tooling maps, anything loaded on demand
```

## Five rules for adding a skill

**1. `skills/<name>/SKILL.md`, and the frontmatter `name` must equal the directory name.**
Required frontmatter: `name` (≤64 chars, lowercase/digits/hyphens, no leading, trailing or doubled
hyphen) and `description` (≤1024 chars — say *what it does* and *when to use it*, keyword-rich,
because that is all an agent sees before deciding to load you). Optional and worth using:
`license`, `compatibility` (≤500 chars — environment requirements), `metadata.version`.
Claude-Code-only extras (`argument-hint`, `disable-model-invocation`, …) are additive; other agents
ignore unknown keys.

**2. Add `"./skills/<name>"` to the plugin's `skills` array in `marketplace.json`.**
Not optional. The entry uses `source: "./"`, and with a marketplace-root source the listed paths are
the *complete* set — **a skill directory nobody lists silently never loads in Claude Code**, while
`npx skills` still finds it by scanning. That asymmetry is exactly how a skill ends up "working on
my machine" and missing for everyone who installed the plugin.

**3. Name exact tools, and ship a `reference/tooling.md` with it.**
Naming the exact tool (`ToolSearch`, `evaluate_script`, `AskUserQuestion`) is what makes a run fast
and reproducible; generalising the prose to "use your browser tool" costs capability and buys
little. Claude Code's vocabulary governs the *harness* — `ToolSearch`, `AskUserQuestion`, `Read`.
The **browser** does not: Chrome DevTools MCP is the default in every client, with Claude in Chrome
as a Claude-Code-only fallback, so a phase carries both vocabularies and `tooling.md` maps them.
Where the two tools want genuinely different *code* — `evaluate_script` takes a function that
returns, `javascript_tool` is a REPL where the last expression is the result — **ship both forms
verbatim** rather than a form plus a conversion rule. An agent rewriting code at runtime is how you
get a silent `undefined` that reads as a thin topic instead of a bug. See
[`skills/deep-research/reference/tooling.md`](skills/deep-research/reference/tooling.md).

**3a. A hard prerequisite gets a preflight, not a mid-run surprise.**
If a skill can't work without something outside it, it states the probe that proves the thing is
live, and the halt copy for when it isn't — up front, before the first phase. Prefer a probe that
distinguishes failure *modes* (`me_get` separates not-configured from stale-config from
no-access) over one that only asks whether a tool exists. Where the prerequisite is a service we
don't control, point at its own setup page instead of copying its endpoint and credentials in here:
a copy is one more thing to go stale, and connection details fail silently.

**4. Every skill folder is self-contained.**
`npx skills add <repo> --skill learn` installs *that one folder*. So a skill may never reference a
path outside its own directory — if `learn` needs the asktube wire contract, **copy**
`reference/asktube.md` into `skills/learn/reference/`. Yes, that duplicates it; the alternative is a
skill that is broken for anyone who installed it alone. At skill #3, add a sync script plus a CI
check that the copies match, rather than trusting anyone to remember.

**5. No external dependencies.**
No other skill, plugin or marketplace. No API keys — asktube already holds the metadata a skill
would otherwise reach for (exact `publishedAt`, caption text, channel data), and adding a key turns
a one-command install into a setup guide. If a skill seems to need one, that is a gap in asktube
worth filing, not a dependency worth adding.

## Before you commit

```bash
claude plugin validate . --strict            # manifest + every skill's frontmatter
npx skills add . --list                      # must show the skill grouped under "Asktube"
```

The second command is the one that matters: the grouping only appears if the CLI parsed
`marketplace.json` and matched your skill's path to the plugin entry. If your skill lists ungrouped,
you skipped rule 2.

**We follow trunk-based development: commit straight to `main`.** No feature branch, no PR. Run the
two commands above first — on trunk, a broken manifest is broken for everyone immediately, and the
CI check you were hoping would catch it is a branch-and-PR habit this repo doesn't have.

## Keeping the asktube contract honest

`reference/asktube.md` carries a "verified against `asktube@<sha>`" line. asktube ships often, and a
stale wire contract fails *silently* — the run reads a field that no longer exists and reports a
thin topic. When a skill run hits a mechanic that misbehaved, that goes in the run's `retro.md`, and
the fix lands here. Re-verify against the asktube repo before trusting an old sha.
