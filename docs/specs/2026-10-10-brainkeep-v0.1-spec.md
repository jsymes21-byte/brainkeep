# Spec: BrainKeep v0.1

*Status: DRAFT, awaiting James's review · 2026-10-10 · target: VC expo, Monday 2026-10-12*
*North star: [THESIS.md](../../THESIS.md). Every section below is checked against it.*

> **This is a prototype.** James's own Brain is the working system; BrainKeep v0.1 exists
> so someone can install it and look at how it works. Two rules follow from that:
>
> - **Break loudly.** Anything not working stops with a non-zero exit and a plain message
>   saying what failed and where. No fallbacks, no retries, no "degrade gracefully", no
>   swallowed exceptions. A broken prototype that says so beats one that looks fine.
> - **Prototype scope.** No upgrade paths, no migration, no uninstall, no compatibility
>   shims. Build the shortest honest version of each piece.

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
   or a line a human can open. *(This is claim 3 applied to failures.)* In the prototype
   this means **break loudly**: a failure is an error the user sees, not a log line.

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
| **A. Capture: never lose the work** | P1 hooks never block · P3 safe to run twice · P5 lifecycle · P8 write-back lands | **Code** — seven mechanisms, one winner each (§ Capture below) | — |
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
   SessionStart ── fetch (10 s cap) · inject hot.md · status block (ahead/behind, FAILED-, pending)
   UserPrompt   ── router: routes.json exact → BM25 (vault-search)
   PreToolUse   ── class gate on Library writes; records the path (= own paths)
   PreCompact   ── copy transcript → raw store (sync, milliseconds)
   Stop         ── receipt gate: the session cannot end until it has written back
   SessionEnd   ── timeout 60: copy → index → commit own paths → search → push → confirm → status
                   │
   ┌───────────────▼──────────────────────────────┐   ┌──────────────────────────┐
   │ the Library (git, markdown)                  │   │ raw store (NOT git)      │
   │  hot.md · index.md · index/NNN.md (generated)│   │ ~/.local/share/brainkeep/│
   │  shelves/<class>/  each with CARD.md         │   │   raw/<harness>/<sid>.jsonl
   │  decisions · gotchas · governance · journal  │   │   raw/INDEX.md (1 line/capture)
   │  .brainkeep/status/  (push, FAILED-*, lock)  │   │ 0700 dir, 0600 files     │
   └───────────────▲──────────────────────────────┘   └──────────────────────────┘
                   │  the Keepers (agents + skills)
   brain-manager · vault-maintenance (the Night Watch, nightly timer)
   wiki-capture (owns hot.md) · brain-loop · agent-brain-adapter
   agent-builder + agent-builder-worker
                   │  the builder kit
   Superpowers (vendored, modified): brainstorming → writing-plans → execute → finish
```

**Everything runs in the foreground, under a budget it declares.** There are no detached
observers in v0.1. Every hook is a Gate: it runs synchronously, declares its own timeout
to the harness, and exits non-zero with a named error when a step fails. The one hook
that does real work, SessionEnd, declares `timeout: 60` (the Claude Code ceiling) and
needs about one second of it (§ Capture, A1).

**How "break loudly" works when the session is already closing.** A SessionEnd error
still reaches stderr, but the user may not be looking. So every failure is also a file:

| Where it fails | What the user sees |
|---|---|
| A Gate during the session | non-zero exit + one-line error in the harness, naming the script and the cause |
| A SessionEnd step | the same error on stderr **and** `status/FAILED-<what>.md` with the error and the command; the **next SessionStart puts it at the top of the context in a `BRAINKEEP FAILURE` block** and it stays there until the file is deleted |
| The CLI | non-zero exit, error on stderr, no partial output passed off as success |
| The Night Watch | report opens with `OUTCOME: failed` and the step that broke; the timer unit exits non-zero so `systemctl --user` shows it red |

No `|| true`, no bare `except:`, no retry loops, no fallback from one search method to
another. If the router's search DB is missing, the router errors; it does not quietly
inject nothing.

**One writer at a time.** The commit step takes `flock -n` on `.brainkeep/status/write.lock`;
the lock file names its holder (harness, session, pid, start time). A second writer does
not wait or skip quietly: it fails with `lock held by <holder>` and writes a `FAILED-` file.
Commits stage the writer's **own paths** — the Library files the PreToolUse class gate
approved in this session, recorded in the session's state file — never `git add -A`, and
carry a `BrainKeep-Writer: <harness>/<session_id>` trailer.

### Capture — one winner per mechanism

Capture is everything between "the agent did work" and "the work is in the Library, on
the remote, and the next session knows". It is seven mechanisms, not one. Each row names
one winner and what was dropped; nothing is merged. **Orca wins none of them**: its hooks
are status telemetry for a GUI, not capture. Its contributions land in Category B
(liveness verdicts, the outcome field) and in the scheduler templates.

| # | Mechanism | Winner | Why — the evidence | Not taken |
|---|---|---|---|---|
| A1 | **The hook shape:** how capture runs without stalling the session | **Brain's Codex path, applied to Claude** — foreground, declared budget, loud | Claude Code lets a SessionEnd hook declare `timeout` up to 60 s (docs). The Brain's pipeline minus the push takes ~1 s on 362 pages: build, commit and search-DB lines carry the same second (`obsidian-vault-sync.log` 839–841, 851–853; n=2 runs that reached the search step). Codex already runs the identical wrapper in the foreground under 55 s and is the only path with a logged push. Claude's loss is the 1.5 s default, nothing deeper | MemPalace's `& disown` + PID slot: the right tool only when you can't set the budget. Orca's `{}`-then-curl (refuted as background) and its spool: **git's local commits are the spool** — `git log origin/master..HEAD` is readable and `git push` is the replay |
| A2 | **When raw capture fires** | **MemPalace's event set minus the Stop interval:** PreCompact + SessionEnd + a nightly sweep | PreCompact carries `transcript_path` on Claude and Codex (docs). A copy is milliseconds, so it runs synchronously. A session that dies before SessionEnd is caught by a sweep, not another hook: the Night Watch copies any transcript newer than its raw copy and reports "N captured by sweep (never reached SessionEnd)" — MemPalace's own `backup_claude_jsonls.sh` idea, made a Keeper duty and made loud. Also beats Claude Code's 30-day transcript deletion (`cleanupPeriodDays`) | MemPalace's Stop-every-15 (the file is on disk anyway; the sweep catches what it would). Orca's setup/archive scripts (worktrees, not capture). **Hermes has no compaction hook** — a stated gap: SessionEnd export + sweep only |
| A3 | **Raw store identity** (safe to run twice) | **MemPalace's principle with the natural key:** deterministic key, probe first, already-there is a visible no-op | MemPalace hashes content because its chunks have no natural key. A transcript has one: `raw/<harness>/<session_id>.jsonl`. It is append-only, so longer = superset, and copy-if-longer is idempotent by construction. `raw/INDEX.md` gets one line per capture (time, harness, session, bytes, sha256, which hook) — a human-readable index of the raw store, which none of the three has | Content-hash file names (wrong key for a growing file); chunking, vectors, purge-then-insert; first-write-wins that drops provenance |
| A4 | **Curated write-back** (the Keeper's capture: hot.md, journal) | **Brain's Codex receipt gate, ported to Claude** | Verified semantics (`codex-brain/brain_hooks.py:318–354`): the first Stop without a receipt blocks once with the capture instruction; the receipt is a sha256 marker the model must include in its journal entry; `stop_hook_active` prevents a loop; once observed, the hook commits and the model never does; the next SessionStart says "unfinished write-back" if the receipt never came (`:135–141`). Claude's Stop offers the same `decision: block` and `stop_hook_active`. It is the only mechanism in the three that makes the curated layer *happen*, and the receipt is the ack P2 found missing | MemPalace's silent auto-diary (its own cell: "a large pile of unreviewed drawers, no human gate"). The Brain's Claude Stop reminder (plain stdout, SUSPECTED inert) |
| A5 | **Landing:** commit, push, and knowing it landed | **Brain's "push whatever is ahead, every run" + MemPalace's "advance the marker only on a confirmed write", applied to the push** | The Brain's rule is tested (`test_a_stranded_commit_is_retried`) and ended the 09-21 stranding of 22 commits. MemPalace's rule (`hooks_cli.py:1518–1527`) is why its Stop checkpoint never lies. Combined: after `git push` the hook confirms `origin/<branch> == HEAD` and only then writes `status/push.md: pushed <sha>`; otherwise non-zero exit, `FAILED-push.md`, and the commit stays local. New, from nobody: the SessionStart block prints `origin: in sync / ahead N / unreachable (fetch timed out 10 s)` | Orca's spool + drain-at-start (silent drops, no counter). MemPalace's fire-and-forget SessionEnd with the marker deleted in `finally` — the exact silent-failure shape |
| A6 | **Which files a session commits, and as whom** | **Brain's Hermes own-path staging, generalised + MemPalace's named-holder lock** | Own paths = the Library paths the PreToolUse class gate approved this session, recorded in the session state file. Codex and Hermes already parse every write there; Claude gets the same gate. No transcript parsing. A file written around the gate (a heredoc) is not committed and is **listed in the next SessionStart block as uncommitted** | `git add -A` (sweeps other sessions' half-done edits — Bug B). Orca's "trust the coordinator" |
| A7 | **Idempotent generated artefacts** | **Brain:** deterministic index with an idempotence test | `test_clean_build_writes_only_the_index` builds twice and asserts byte-equality — the only tested idempotence in the three. Two one-line fixes: per-pid temp name + atomic rename for the index and the search DB; the receipt token doubles as the journal dedupe key | Orca's rename-to-trash (no deletes in v0.1) |

**"Push whatever is ahead" is not a retry loop.** Each run makes exactly one push attempt
and reports it. A failed push stays visible (`FAILED-push.md`, `ahead N` at every start)
until a later run's single attempt lands, which then deletes the FAILED file.

**What each harness runs**

| Event | Claude Code | Codex | Hermes |
|---|---|---|---|
| SessionStart (Gate) | `git fetch` under 10 s · inject hot.md · status block | same, inside the 30 s hook budget (inner git timeout < 30) | `on_session_start`, same |
| PreToolUse on a Library write (Gate) | class gate **(new on Claude)** + record the path | exists + record | `pre_tool_call` exists + record |
| PreCompact (Gate) | copy transcript → raw; failure = exit 2, compaction blocked, error shown | same; failure = `continue: false` | — no hook exists |
| Stop (Gate) | receipt gate **(ported)**; `{}` once the receipt is in | receipt gate + per-turn commit (exists) | `pre_verify` reminder (exists) |
| SessionEnd (`timeout: 60`) | copy → index → commit own paths → search DB → push → confirm → `status/push.md` | audit only (Codex caps SessionEnd at 3 s) | `on_session_end`: export the session from `state.db` → raw; commit + push |
| Nightly (Night Watch) | sweep stray transcripts · confirm `origin == HEAD` · list uncommitted Library files | same | same |

**The SessionStart block** is the one place a human sees Capture's state. It is the first
thing in the context, every session:

```
BRAINKEEP · Library ~/library · origin: ahead 2 (last push failed 2026-10-10 14:05)
hot.md: 3 days old
FAILED-push.md: ssh: connect to host github.com port 22: Network is unreachable
uncommitted (written outside the class gate): journal/2026-10-10-notes.md
previous session a1b2c3: closed without write-back — recover from its journal stub
```

When nothing is wrong it is one line: `BRAINKEEP · Library ~/library · origin: in sync · hot.md: today`.

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
Health check:                        bin/brainkeep doctor          # wiring + dead symlinks + FAILED- files, exit 1 on any
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
├── install.sh                dry-run by default; --apply to write; symlinks, never copies
│                             (so nothing can drift); marker-owned config entries
├── bin/brainkeep             the one CLI (doctor, index, search, night-watch, leak-scan)
├── keepers/
│   ├── agents/               brain-manager, agent-builder, agent-builder-worker
│   └── skills/               vault-maintenance, wiki-capture, brain-loop,
│                             agent-brain-adapter, agent-builder (+ templates, fence.py, scan_runs.py)
├── gates/
│   ├── ADAPTER-CONTRACT.md   how to write a Gate for harness X (the PR path for new harnesses)
│   ├── lib/                  shared: lock (named holder) · status writer · raw-store copy · own-paths record
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
  - the SessionEnd pipeline on the template Library finishes in under 5 s, and the installed hook entry declares `timeout: 60` (A1)
  - two SessionEnds at once → one commits, the other exits non-zero with "lock held by <holder>" and writes a `FAILED-` file (A6)
  - a `FAILED-` file in `status/` → SessionStart output begins with the `BRAINKEEP FAILURE` block
  - router with a missing search DB → non-zero exit and a named error, not an empty injection
  - PreCompact copy run twice on the same transcript → one raw file, one `INDEX.md` line; a longer transcript replaces the shorter one (A3)
  - Stop without a receipt → `decision: block`; Stop with `stop_hook_active` → `systemMessage`, no block; Stop with the receipt present → `{}` (A4)
  - commit stages only the paths the class gate recorded; a Library file written around the gate is not committed and is listed in the next SessionStart block (A6)
  - push to an unreachable remote → non-zero exit, `FAILED-push.md`, and the next SessionStart block shows `ahead 1` and the failure; a later successful push deletes the file (A5)
  - leak scan blocks a staged file containing an IP, an email, a token-shaped string or `/home/<user>`
  - `install.sh` without `--apply` changes nothing (tree hash before == after)
  - `doctor` exits 1 when a hook is not wired or a symlink is dead, and names which
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
  an approve-every-change gate · say "like Orca" · swallow an error (`|| true`, bare
  `except:`, `2>/dev/null` on a step that matters, a silent fallback or retry).

## Success Criteria (Monday demo)

1. `./install.sh` on a clean temp HOME produces a working Library and `brainkeep doctor`
   exits 0, all in under a minute.
2. A Claude Code session started in it receives `hot.md`; mentioning a routed topic injects
   the page.
3. Closing the session leaves a raw transcript in the raw store (one `INDEX.md` line) and
   a commit staged by own paths with a `BrainKeep-Writer:` trailer, confirmed against
   origin; a forced push failure is reported at the next start as `ahead 1` plus the error.
4. All tests pass; the leak scan over the full tree reports 0 findings.
5. `kit/superpowers/` loads in Claude Code, and its four patches are listed in `PATCHES.md`.
6. The README renders on a phone: keep logo, one-line thesis, diagram, 3-command quickstart,
   link to the trackers repo.
7. Every shelf in `library-template/` has a `CARD.md` a stranger can read.
8. **Breaking it is loud:** unplug the remote, delete the search DB, or hold the lock from
   a second shell. Each one produces a visible, named error at the point listed in the
   "break loudly" table. None of them produces a quiet success.

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
| C1 | Hook shape: foreground under a declared 60 s budget, no detached observers (A1) | **pending** — recommended over MemPalace-style detach: the pipeline is ~1 s, and detach only buys an instant exit at the cost of the direct error |
| C2 | Receipt gate on Claude: Codex semantics verbatim (block once per session, every session) (A4) | **pending** — recommended; the alternative is to block only when the session wrote to the Library, with chat-only sessions getting the loud start-of-session notice instead. Costs one model round on the first turn; wiki-capture SKIPs honestly when there is nothing |
| C3 | PreCompact copy failure blocks compaction (exit 2) (A2) | **pending** — recommended as the rule applied literally; the copy is milliseconds and fails only for disk or permission reasons a human must fix anyway |

## Open Questions

1. **Exact names of the 10 core shelves** — the plan proposes them; James signs off.
2. **addy agent-skills** (spec-driven-development, planning, TDD…) — bundle as an optional
   second pack, or Superpowers only? Spec assumes Superpowers only for v0.1.
3. **"Deploy on their own servers"** — the spec assumes a Linux box the user already has
   (laptop or VPS). A Docker image or one-click droplet is not in v0.1. Confirm.
4. **Licence for BrainKeep itself** — MIT assumed (matches Superpowers).
