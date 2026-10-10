# Spec: BrainKeep v0.1

*Status: DRAFT, awaiting James's review · 2026-10-10 · target: VC expo, Monday 2026-10-12*
*North star: [THESIS.md](../../THESIS.md). Every section below is checked against it.*

## Objective

**BrainKeep is an agentic architecture that enables efficient recall and builds itself with
a human in the loop.** It is a packaged system a builder clones onto their own machine or
server and runs from day one: a markdown **Library**, the **Keepers** that maintain it,
**Gates** that wire it into Claude Code, Codex and Hermes, and a bundled **builder kit**
(agent-builder plus a modified Superpowers) for building products with it.

**Who it is for:** multi-agent power users, people who run several coding agents across
harnesses and machines, and who want to watch what is being built and keep it
human-readable.

**What ships:** machinery only. None of James's content: no pages, no projects, no
transcripts, no hostnames.

**Three claims (THESIS.md), plus one principle that came out of the 39-cell comparison:**

1. **Efficient recall.** The right page reaches the agent without re-reading the corpus.
2. **It builds itself.** The Keepers maintain the Library; agent-builder grows new agents;
   the builder kit turns an idea into a product.
3. **A human can read everything.** Nobody approves every change, but anyone can open any
   page, agent or change and understand it.
4. **Nothing fails silently.** Every pending, dropped, stale or superseded thing is a file
   or a line a human can open. *(This is claim 3 applied to failures.)*

**Positioning:** "Works under Orca, Claude Code, Codex and Hermes." Never "like Orca".
Trackers are a separate public repo, linked from the README as an optional Docker add-on.

### User stories

- **U1 Install.** On a fresh Linux box with git and Python 3.10+, I run one command, point
  it at an empty folder, pick my harnesses, and get a working Library. The command shows
  me exactly what it will change before changing it.
- **U2 Recall.** I open Claude Code, Codex or Hermes. The agent already knows where the last
  session left off (`hot.md`), and when I mention a topic the right page is injected.
- **U3 Capture.** I close a session or it compacts. The transcript lands in the raw store,
  and my Library changes are committed. If the push failed, the next session tells me so.
- **U4 Maintenance.** Every night the Night Watch audits the Library and writes a report I
  can read in two minutes.
- **U5 Build.** I describe a product. Superpowers walks me through brainstorm, then spec,
  then plan, then build. I describe a recurring job and agent-builder makes an agent for
  it. Both write their records into the Library.
- **U6 Read.** I open any shelf and its card tells me what belongs there and why each page
  sits there.

## The categories (from the 13-pattern comparison)

| Category | Patterns | v0.1 (Monday) | After Monday |
|---|---|---|---|
| **A. Capture: never lose the work** | P1 hooks never block · P3 safe to run twice · P5 lifecycle · P8 write-back lands | **Code** | — |
| **B. Coordinate: many agents, one Library** | P2 handoffs · P4 agent state · P7 several writers | P7 lock + own-path staging + writer trailer (**code**, it is A's prerequisite) · P2 `OUTCOME:` line (prompt edit) | P2 ack/ruling · P4 heartbeat + liveness rule |
| **C. Recall: the right page, fast** | P6 router · P11 wake-up · P12 superseded | Router + hot.md + stale warning ship as-is (**already lead**); hot.md word-count lint | hazard routes · `superseded_by:` banner + index filter |
| **D. Readable: any human understands it** | P13 shelving · "nothing fails silently" | **Core shelf map + shelf cards + `filed_because:`** (files, not code) | gate checks class is on the map; propose → apply reclass |
| **E. Trust: safe to install and share** | P9 secrets · P10 install | **Code**: leak scan, raw store outside git, installer, `doctor` | uninstall · upstream sync for vendored skills |
| **F. Build: products from day one** *(new)* | — | agent-builder + worker, Superpowers bundled and modified | addy agent-skills as an optional pack |

## Architecture

```
            ┌──────────── harnesses ────────────┐
            │  Claude Code    Codex     Hermes  │
            └──────┬────────────┬─────────┬─────┘
                   │  Gates (hooks, one per harness, same contract)
   SessionStart ── inject hot.md + "N commits not pushed / stale" notices
   UserPrompt   ── router: routes.json exact → BM25 (vault-search)
   PreCompact   ── observer: transcript → raw store (detached, single-flight)
   SessionEnd   ── observer: raw store + commit own paths + push, outcome to a file
                   │
   ┌───────────────▼──────────────────────────────┐   ┌──────────────────────┐
   │ the Library (git, markdown)                  │   │ raw store (NOT git)  │
   │  hot.md · index.md · index/NNN.md (generated)│   │ ~/.local/share/      │
   │  shelves/<class>/  each with CARD.md         │   │   brainkeep/raw/     │
   │  decisions · gotchas · governance · journal  │   │ 0700 dir, 0600 files │
   │  .brainkeep/status/  (pending, push, locks)  │   │ sha256 file names    │
   └───────────────▲──────────────────────────────┘   └──────────────────────┘
                   │  the Keepers (agents + skills)
   brain-manager · vault-maintenance (the Night Watch, nightly timer)
   wiki-capture (owns hot.md) · brain-loop · agent-brain-adapter
   agent-builder + agent-builder-worker
                   │  the builder kit
   Superpowers (vendored, modified): brainstorming → writing-plans → execute → finish
```

**Gates block; observers detach.** A Gate that decides something (class gate on write,
router injection) runs synchronously under a hard timeout. An observer (capture, commit,
push) forks, returns `{}` immediately, runs single-flight under a lock, and writes its
outcome to `.brainkeep/status/`. The next SessionStart reads that folder and says what
didn't land.

**One writer at a time.** All Library writes by machinery take `flock` on
`.brainkeep/status/write.lock`; the lock file names its holder (harness, pid, start time).
Commits stage the writer's own paths, never `git add -A`, and carry a
`BrainKeep-Writer: <harness>/<keeper>` trailer.

**The raw store is never in git.** Verbatim transcripts are private. Redaction happens only
when something is *promoted* from raw into the Library, and the leak scan is the backstop.

### Shelves — making the Dewey system teachable (D2)

James's Dewey mechanism ships; his 52-row map does not.

- **Core map: 10 hundreds** (000 Library itself · 100 Sessions & journal · 200 Machines &
  infra · 300 Projects · 400 Agents & skills · 500 Concepts · 600 How-tos · 700 Decisions ·
  800 Gotchas & patterns · 900 Archive). Exact labels are settled in the plan; the count
  and the rule are what this spec fixes.
- **Project-shelf template:** a user adds `3NN <project>` by copying one folder; the card
  is filled in by the Keeper that creates it.
- **`CARD.md` on every shelf:** scope (one sentence) · belongs here · does NOT belong here
  (with where it goes instead) · three example pages.
- **`filed_because:`** frontmatter line on every page, one clause, written by whoever
  filed it. A human reading the page sees why it sits there.
- The existing `class:` + `summary:` frontmatter, the generated index, and the nightly
  misfile audit carry over unchanged.

### The builder kit — Superpowers, bundled and modified

Superpowers (Jesse Vincent, MIT) is vendored at `kit/superpowers/` with its LICENSE intact
and a `NOTICE` naming the upstream commit. **"Modified slightly" means exactly these four
edits, each listed in `kit/superpowers/PATCHES.md` with its diff, so upstream can be
re-synced:**

1. **Brain first.** `brainstorming` and `writing-plans` read `hot.md` and the routed page
   before exploring code.
2. **Specs and plans are Library pages.** They are written to the project repo as upstream
   does, and a one-line pointer page with `class:`, `summary:` and `filed_because:` is
   filed on the project's shelf.
3. **Write-back at the end.** `finishing-a-development-branch` ends by running
   `wiki-capture --quick`.
4. **Agents go to agent-builder.** `writing-skills` hands off to agent-builder when the
   thing being built is an agent, not a skill.

The human gates stay where Superpowers already puts them (approve the spec, approve the
plan). No new approval gates are added (THESIS claim 3).

agent-builder and its worker ship as they run today, de-personalised: "James" becomes
"you", personal project examples are replaced by a generic one, and the groundskeeper/fleet
branches are cut.

## Tech Stack

- **Runtime:** bash + Python 3.10+ **standard library only** (`sqlite3` FTS5 for search,
  `fcntl` for locks). No runtime dependencies to install.
- **Requires:** git, one of Claude Code / Codex / Hermes. systemd (user) for the nightly
  timer is optional; without it the Night Watch runs on demand.
- **Dev only:** pytest (runs the existing unittest and pytest suites alike), shellcheck.

## Commands

```
Install (dry run, shows the plan):   ./install.sh --library ~/library --harness claude,codex,hermes
Install (apply):                     ./install.sh --library ~/library --harness claude --apply
Health check:                        bin/brainkeep doctor          # wiring + drift + status, exit 1 on any failure
Rebuild the index:                   bin/brainkeep index
Search:                              bin/brainkeep search "<query>"
Night Watch on demand:               bin/brainkeep night-watch
Leak scan (also the pre-commit):     bin/brainkeep leak-scan [--staged]
Tests:                               python3 -m pytest tests/ -q
Lint:                                shellcheck gates/**/*.sh tools/*.sh
```

One config value: `BRAINKEEP_LIBRARY`, written by the installer to
`~/.config/brainkeep/env` (0600). Every tool reads it; nothing has a hard-coded path.

## Project Structure

```
brainkeep/
├── THESIS.md                 the north star
├── README.md                 pitch, ASCII keep, diagram, 3-command quickstart
├── install.sh                dry-run by default; --apply to write; marker-owned entries
├── bin/brainkeep             the one CLI (doctor, index, search, night-watch, leak-scan)
├── keepers/
│   ├── agents/               brain-manager, agent-builder, agent-builder-worker
│   └── skills/               vault-maintenance, wiki-capture, brain-loop,
│                             agent-brain-adapter, agent-builder (+ templates, fence.py, scan_runs.py)
├── gates/
│   ├── ADAPTER-CONTRACT.md   how to write a Gate for harness X (the PR path for new harnesses)
│   ├── lib/                  shared: detach + single-flight + lock + status writer
│   ├── claude-code/          hooks + settings.json SNIPPET (never a real file)
│   ├── codex/                brain_hooks.py + hooks.json template + AGENTS.md
│   └── hermes/               brain_hooks.py + validate_note.py + hooks snippet
├── library-template/         what the installer copies into an empty Library
│   ├── CLAUDE.md  hot.md  index.md  routes.json
│   └── shelves/NNN-*/CARD.md
├── tools/                    vault-index-build.py, vault-search.py, session-end.sh,
│                             vault-maintenance.sh + checks, wake-agent.sh, leak-scan.py
├── scheduler/                systemd user timer templates (Persistent=true, ExecCondition)
├── kit/superpowers/          vendored, MIT, NOTICE + PATCHES.md
├── tests/                    mirrors the tree above
└── docs/specs/               this file
```

## Code Style

Match the Brain's existing tools: small stdlib scripts, one job each, a docstring that says
what and why, explicit exit codes, outcomes written somewhere a human can read.

```python
def push_outcome(library: Path, ok: bool, detail: str) -> None:
    """Record the push result where the next SessionStart will read it.

    Silence must never look like success, so a failure is a file, not a log line.
    """
    status = library / ".brainkeep" / "status" / "push.md"
    status.write_text(f"{'ok' if ok else 'FAILED'} {now_iso()} {detail}\n")
```

- Paths come from `BRAINKEEP_LIBRARY`, never literals.
- Every Keeper report opens with `OUTCOME: <done|partial|failed> — <one line>`.
- Markdown pages carry `class:`, `summary:`, `filed_because:` frontmatter.
- Plain names everywhere except the entrance (Library, Keepers, Shelves, Gates). Decisions,
  gotchas, governance, hot.md and journal keep their names.

## Testing Strategy

- **Carry over the clean tests:** index build (incl. idempotence), search, `fence`,
  `scan_runs`, codex/hermes `brain_hooks`. Fixtures rebuilt on a temp Library — no
  personal paths, no personal project names.
- **New tests, one per Category A/E claim:**
  - observer returns within 200 ms while its work continues (detach)
  - two observers at once → one runs, the other records "skipped, held by <holder>"
  - run capture twice on the same transcript → one raw file (content hash)
  - commit stages only the writer's paths; a foreign staged file is left alone
  - push to an unreachable remote → `status/push.md` says FAILED and SessionStart shows it
  - leak scan blocks a staged file containing an IP, an email, a token-shaped string or `/home/<user>`
  - `install.sh` without `--apply` changes nothing (tree hash before == after)
  - `doctor` exits 1 when a skill copy has drifted from its source
- **End-to-end smoke:** install into a temp HOME with a fake harness config, open/close a
  scripted session, assert the Library commit and the raw file exist.

## Boundaries

- **Always:** run the leak scan before every BrainKeep commit and push · commit by path ·
  keep THESIS claims in view · write outcomes to files · keep Superpowers' LICENSE and
  list every patch in `PATCHES.md`.
- **Ask James first:** making the repo public · any change to the live Brain's hooks or
  settings (bugs A/B, the SessionEnd timeout) · adding a runtime dependency · changing the
  shelf count or names · deploying the expo landing page · vendoring any other skill pack.
- **Never:** copy content, config files or secrets from the live Brain (`~/.hermes/config.yaml`,
  `~/.codex/auth.json`, notifier env files, monitoring drop-ins, Orca config) ·
  `git add -A` · put raw transcripts in git · execute code from the MemPalace clone · add
  an approve-every-change gate · say "like Orca".

## Success Criteria (Monday demo)

1. `./install.sh` on a clean temp HOME produces a working Library and `brainkeep doctor`
   exits 0, all in under a minute.
2. A Claude Code session started in it receives `hot.md`; mentioning a routed topic injects
   the page.
3. Closing the session leaves a raw transcript in the raw store and a commit with a
   `BrainKeep-Writer:` trailer; a forced push failure is reported at the next start.
4. All tests pass; the leak scan over the full tree reports 0 findings.
5. `kit/superpowers/` loads in Claude Code, and its four patches are listed in `PATCHES.md`.
6. The README renders on a phone: keep logo, one-line thesis, diagram, 3-command quickstart,
   link to the trackers repo.
7. Every shelf in `library-template/` has a `CARD.md` a stranger can read.

Out of scope for Monday: Category B beyond P7 and the `OUTCOME:` line, hazard routes,
`superseded_by:`, uninstall, Codex/Hermes end-to-end smoke (unit tests only), the SVG logo,
the symes.com page (separate task, same weekend if time allows).

## Decisions folded into this spec

These were recommended in-session and are taken as accepted by James's "merge it all into
a good spec". Overrule any of them in review.

| # | Decision | Taken |
|---|---|---|
| M1 | Left out: completion gate, git worktree tools, coding standards, design history, operator READMEs | yes — short new docs instead |
| M2 | One config value `BRAINKEEP_LIBRARY` | yes |
| M3 | Where two copies drifted, take the deployed one | yes |
| M4 | Telegram is an optional notifier (token *path*, never a value); default is a log file | yes |
| M5 | Only clean tests go in; the rest after the expo | yes |
| D1 | Monday = Categories A + E as code, C ships as-is, D as files and README story | yes |
| D2 | Core map + template + shelf cards + `filed_because:` | yes |
| D3 | "Nothing fails silently" becomes principle 4 in THESIS.md | yes |
| F1 | Superpowers bundled by default with four listed patches | yes (James, 2026-10-10) |

## Open Questions

1. **Exact names of the 10 core shelves** — the plan proposes them; James signs off.
2. **addy agent-skills** (spec-driven-development, planning, TDD…) — bundle as an optional
   second pack, or Superpowers only? Spec assumes Superpowers only for v0.1.
3. **"Deploy on their own servers"** — the spec assumes a Linux box the user already has
   (laptop or VPS). A Docker image or one-click droplet is not in v0.1. Confirm.
4. **Licence for BrainKeep itself** — MIT assumed (matches Superpowers).
