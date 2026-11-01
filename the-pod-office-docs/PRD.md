# The Pod Office (Orca edition)

**Type:** personal project · **Base:** soft fork of Orca (stablyai/orca, MIT) · **Goal:** a full ADE that's fun to look at and built around how _I_ work

> Orca is the IDE. The Pod Office is where my team sits: my office, my agents, and my workflow, on top of it.

**Status:** v2.1. Phase 0 ran on Oct 31, 2026 on macOS with Claude Code 2.1.314 and Haiku. Facts below are from the code and from those runs. Results are in §10; one decision is still open (permission mode, §8).

---

## 1. Why this shape

- I want a **full ADE**: real terminals, an editor, diffs, worktrees, git. Orca already has all of that.
- I want it to **look fun** (an office with mascots) and **work my way** (my roster, my rules, PM → Planner → tickets).
- So: **fork Orca, and build my stuff as an Orca plugin inside the fork.** I only change Orca's own code where the plugin API can't do what I need, and each of those changes is small and written down.

**Non-goals for v1** (personal tool, so these are deliberate): remote/SSH hosts, mobile, folder workspaces, Windows/Linux testing, agents other than Claude Code, multiple project repos at once, a packaged or shareable build. I run it from source. The plugin avoids shell-only and hard-coded POSIX paths so these stay possible later.

---

## 2. What the code actually does

### Confirmed

| Thing                       | Evidence                                                                                                                              | What it means for me                                                                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Worker has full access      | Plain `fork` with no permission model (`plugin-host-process.ts`)                                                                      | It can read `team/` and spawn `git` and the Orca CLI                                                                                                |
| Status event                | `workspaceId`, `paneKey`, `state`, `receivedAt`, optional `mainAgent.{state,outcome,stateStartedAt}` (`plugin-agent-status-event.ts`) | Drives mascots. No terminal scraping                                                                                                                |
| CLI coverage                | `worktree create/list/ps/rm`, `agent launch`, `terminal list/switch`, `file diff/open-changed`, `repo list` all exist                 | No core change needed for orchestration                                                                                                             |
| Compile before launch       | `worktree create` without `--agent`, then `agent launch --worktree`                                                                   | The profile can be in place before the agent starts                                                                                                 |
| Claude hooks are user-level | Written to `~/.claude/settings.json` (`hook-settings.ts`)                                                                             | Per-worktree settings don't touch Orca's hook file                                                                                                  |
| Dev loading                 | `orca plugin dev load <dir>`, file watcher, `orca plugin logs`                                                                        | No install or catalog step                                                                                                                          |
| Version gate is fine        | `pluginHostVersion` raises source builds to the schema's newest `Since`                                                               | The `1.4.214` in `package.json` doesn't block plugins. It's expected: upstream `main` shows the same version while the latest release is `v1.4.245` |

### Constraints v1 of this PRD missed

| Constraint                                | Evidence                                                                                                                                                                | Consequence                                                                                                                                                                                                                                                               |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Panel CSP is strict**                   | `default-src 'none'; connect-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src data:; font-src data:` (`plugin-panel-shell.ts`)                | `panel.html` must be **one self-contained file**: inline JS and CSS, sprites as `data:` URIs. No `assets/` loading, no network, no `<audio>` files. Limit is 10 MiB                                                                                                       |
| **Panels unmount**                        | Sidebar content renders only for the active tab (`right-sidebar-panel-content.tsx`)                                                                                     | The panel holds no state, and it can't be relied on for alerts. The worker owns state and sends attention alerts itself                                                                                                                                                   |
| **Panel calls are budgeted**              | 30 messages per 10 s and 64 KiB per message, per plugin (`plugin-frame-bridge.ts`)                                                                                      | Polling must be slow and snapshots small                                                                                                                                                                                                                                  |
| **Workers are reaped**                    | Started on demand and killed after 5 idle minutes; checked every 60 s (`plugin-worker-manager.ts`)                                                                      | No long-lived timers or memory-only state. Persist to plugin storage and derive "stuck" from timestamps                                                                                                                                                                   |
| **Worker env is an allowlist**            | Only `PATH`, `HOME` and similar pass through; `ORCA_USER_DATA_PATH` is stripped (`plugin-worker-env.ts`)                                                                | **A worker calling the CLI in a dev build would reach the wrong app.** The CLI defaults to the packaged `orca` profile while dev runs `orca-dev`. See seam S0                                                                                                             |
| **Panels live in the right sidebar**      | Manifest reference                                                                                                                                                      | An office scene needs a narrow-first layout. The full-size route is in §6                                                                                                                                                                                                 |
| Manifest details                          | Schema                                                                                                                                                                  | The `building` icon is invalid (using `bot`). `publisher` is deprecated. Maximum 3 event subscriptions                                                                                                                                                                    |
| Panes aren't terminals                    | Event has `paneKey`; `terminal switch` wants a handle                                                                                                                   | `agent launch --json` returns both, so I persist them at launch. For agents I didn't launch, match `recordedPaneKey` from `terminal list --json`; `tabId:leafId` did **not** match in testing. Chat sessions have no terminal handle and `terminal switch` refuses theirs |
| Restored status isn't delivered           | `restoredUnconfirmed` rows are dropped from events                                                                                                                      | After a restart, state must be rebuilt from `orca worktree ps --json`                                                                                                                                                                                                     |
| Outcome is usually absent                 | A clean finish had no `outcome`; an auth failure had `failure` (one run each). `agent-turn-outcome.ts` says absent means unknown, never success                         | Done with no outcome means "ready for review". Any present non-success outcome means "check me"                                                                                                                                                                           |
| **Attention is `waiting`, not `blocked`** | Permission prompts and questions both reported `waiting` in runs; `agent-status-projector.ts` maps both to it on purpose. `blocked` is for other kinds, such as sign-in | "Needs me" is built on `waiting`. `blocked` stays as a rarer alert                                                                                                                                                                                                        |
| `done` fires at startup                   | A new pane emitted `done` before its first `working`                                                                                                                    | A ticket moves to `review` only after its pane has been seen `working`                                                                                                                                                                                                    |
| **Launched agents ran in bypass mode**    | `agent launch` showed `bypass permissions on`; a plain `claude` in a terminal showed `auto mode on`. I haven't traced where the launch default comes from               | No approval prompts appear, so `tools.allow` does nothing. `tools.deny` **did** hold under bypass, using `Edit(path)` form                                                                                                                                                |

---

## 3. What it feels like

- Orca's normal window, with my **Office** panel showing a Product room and an Engineering room.
- I chat with my PM to write a PRD. The Planner turns it into tickets.
- I click **Start** on a ticket: the overlap check runs, an Orca worktree is created, my profile is compiled into it, the agent launches in an Orca terminal, and a mascot takes a desk.
- Mascots type while working, show a `?` when they need me, and raise a hand when done. If the panel is closed, a desktop notification tells me an agent needs me.
- **Click a mascot → Orca jumps to that agent's terminal.** Questions, and approvals if I run with them on, happen right there.
- Click a raised hand → Orca's diff view for that worktree → I commit and merge with Orca's own source control.

---

## 4. Features

| #   | Feature                           | How                                                                                                                    | Core change? | Phase |
| --- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------ | ----- |
| F1  | Office view, rooms per department | Self-contained panel, narrow-first, DOM + CSS + emoji                                                                  | S1           | 1     |
| F2  | Live mascot states                | `agent.status.changed` → worker state → panel polls snapshot                                                           | S1           | 1     |
| F3  | Roster + profiles                 | `team/agents/*.md` read by the worker                                                                                  | S1           | 2     |
| F4  | Base rules + inheritance          | Worker merges `base-rules.md` + role rules                                                                             | —            | 2     |
| F5  | Profile compiler                  | Writes `CLAUDE.local.md` and `.claude/settings.local.json`, excluded from git (§8)                                     | —            | 2     |
| F6  | PM → PRD                          | `orca agent launch --surface terminal` in the primary checkout                                                         | —            | 3     |
| F7  | Planner → tickets                 | Same, writes `team/tickets/*.md`                                                                                       | —            | 3     |
| F8  | Overlap check                     | `git ls-files -- ':(glob)<scope>'` per ticket, then intersect the results                                              | —            | 2     |
| F9  | Worktree per ticket               | §8 start sequence                                                                                                      | —            | 2     |
| F10 | "Needs me" strip                  | Derived from state: `waiting` and `blocked`. Worker sends `notifications.show` on entering either                      | S1           | 1     |
| F11 | Approve from UI                   | Click mascot → jump to terminal, approve there                                                                         | —            | 1     |
| F12 | Diff review                       | `orca file open-changed`; commit and merge in Orca's own UI                                                            | —            | 3     |
| F13 | Scope alerts                      | When a turn ends, `git diff --name-only <base>...` plus untracked files, checked against scope. No hook or seam needed | —            | 3     |
| F14 | Stuck detection                   | Computed on read: `working` and `now - lastEventAt > N min`. No timer                                                  | —            | 2     |
| F16 | Project memory                    | Plain markdown `team/` folder                                                                                          | —            | 2     |

**Cut from v1:** cost meter (F15, Orca already shows account usage), in-panel Merge button, profile `skills:` (I haven't read how `orca skills install` scopes its installs), approve buttons (S3).

---

## 5. How it fits together

```
┌──────────────────── Orca (my fork) ─────────────────────┐
│  Terminals · editor · diffs · git · worktrees            │
│  Agent hooks → status per pane                           │
│                                                          │
│  ┌─── plugins/pod-office/ ───────────────────────────┐   │
│  │ panel.html  (built, single file, holds no state)  │   │
│  │     │  S1: invoke my worker's command             │   │
│  │     │      snapshot every 2 s + click actions     │   │
│  │     ▼                                             │   │
│  │ main.mjs  worker (reapable, so it persists state) │   │
│  │   • agent.status.changed → state → storage        │   │
│  │   • reconcile with `orca worktree ps --json`      │   │
│  │   • read team/, compile profiles                  │   │
│  │   • spawn Orca CLI (S0) and git                   │   │
│  │   • notifications when an agent needs me          │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Rules I'm keeping:**

1. **Plugin first.** Core changes only through the seams in §6.
2. **Files are never triggers.** The worker reads `team/` when I click something. Editing a file never wakes an agent.
3. **One owner per file area.** Tickets declare scope; the overlap check runs before Start.
4. **Don't touch Orca's hook files.** My settings go only in the worktree's `.claude/settings.local.json`, merged with anything already there.
5. **The worker is disposable.** It can die at any moment. Everything it needs comes from plugin storage, `team/`, or the CLI.
6. **Ticket status is not stored in ticket files.** It lives in plugin storage, so status changes never dirty the repo or cause merge conflicts. Ticket files are the plan; storage is what happened.
7. **`team/` is read from the primary checkout only.** Worktrees contain stale copies, so engineer profiles deny edits to `team/**`.

### State the worker persists

`tickets[id] = { state, worktreeId, worktreePath, baseRef, paneKey, startedAt, lastError, launchOperationId }`, plus `panes[paneKey] = { state, outcome, lastEventAt }`.

Ticket states: `planned → starting → running → review → closed`, with `failed` reachable from `starting`. A pane with no ticket shows as a "visitor" mascot, so agents I start by hand still appear.

Prefer `mainAgent.state` over `state` when present, so subagent activity doesn't make the mascot flicker.

**On worker start:** load storage, then reconcile against `orca worktree ps --json`. A ticket whose worktree is gone becomes `closed`.

---

## 6. Changes to Orca's core ("seams"), kept in FORK.md

| Seam   | Change                                                                                                                   | Size                                             | Needed for                           |
| ------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------ |
| **S0** | Add `ORCA_USER_DATA_PATH` to `WORKER_ENV_ALLOWLIST`                                                                      | 1 line + test                                    | Worker's CLI reaches the running app |
| **S1** | New host method, on panel and editor surfaces: invoke one of **this plugin's own** worker commands and return its result | Host API spec + binding + generated docs + tests | Everything in the panel              |
| **S3** | Later: answer a pending approval from a plugin                                                                           | Unknown                                          | Approve buttons                      |

**S1 is deliberately request/response only.** Polling every 2 s uses 5 of the 30 messages per 10 s, leaving room for clicks, and a call wakes a reaped worker. One second of mascot lag doesn't matter, and panels unmount anyway, so a push channel would still need a snapshot call on mount. Add push only if polling proves bad.

S1 rules: a plugin can reach only its own worker; `trusted-worker` plugins only; `orca.host.has()` gates it; results stay within the 64 KiB message cap. The panel uses a single multiplexing command, `pod-office-rpc` with `{ op, ...args }`, so the command palette stays uncluttered. Before writing it, read `plugin-host-call-adapter.ts`, `plugin-frame-bridge.ts`, and how `command.invoke` reaches the worker today, then reuse that path. Run `pnpm generate:plugin-schemas` afterwards.

**Dropped from v1's list:**

- **S2** (tool events): F13 now uses `git diff`. If I ever want it live, the hook server already has tool name and input, so it's an optional field on the existing status event, with no fourth subscription.
- **S4** (full-tab office): try the no-seam route first. A `customEditors` contribution for a file like `team/office.podoffice` opens in the editor area using the same kind of frame. That's why S1 includes the editor surface. Phase 3 stretch, not a commitment.

**Soft-fork habits:**

- Add `upstream` as a git remote (only `origin` exists today). Merge upstream on purpose, about every two weeks; `main` moves daily.
- After each merge, re-run the Phase 0 smoke test. Plugins are Alpha, but Host API 1 is documented as frozen.
- Seam changes get focused tests. Everything else stays in `plugins/pod-office/`.
- Keep the MIT notice. Rename the app if I ever share it.

---

## 7. Folder layout

```text
orca-fork/
├── plugins/pod-office/
│   ├── orca-plugin.json
│   ├── main.mjs                   # worker (no npm dependencies)
│   ├── panel.src/                 # office UI source + sprites
│   ├── build-panel.mjs            # inlines everything into panel.html
│   └── panel.html                 # build output, the only thing the frame loads
├── FORK.md
└── …Orca…

my-project-repo/
└── team/                          # committed, read from the primary checkout
    ├── context.md
    ├── agents/  base-rules.md · pm.md · planner.md · frontend.md · backend.md
    ├── specs/   PRD-001-auth.md
    ├── tickets/ T-101-auth-api.md · T-102-login-ui.md
    └── decisions/
```

No game engine in v1: DOM, CSS animation and emoji cover every state in §9. Add canvas sprites only after the workflow works.

**Manifest** (run `pnpm validate:plugin plugins/pod-office`, which is the real authority):

```json
{
  "manifestVersion": 2,
  "execution": "trusted-worker",
  "id": "pod-office",
  "name": "The Pod Office",
  "version": "0.1.0",
  "engines": { "orca": ">=1.4.246" },
  "pluginApi": 1,
  "main": "main.mjs",
  "contributes": {
    "panels": [{ "id": "office", "title": "Pod Office", "icon": "bot", "entry": "panel.html" }],
    "commands": [{ "id": "pod-office-rpc", "title": "Pod Office: Internal" }],
    "events": [{ "on": "agent.status.changed" }, { "on": "workspace.removed" }]
  },
  "capabilities": [
    { "kind": "workspace:read" },
    { "kind": "agent:read" },
    { "kind": "notifications:show" },
    { "kind": "storage:own" },
    { "kind": "settings:own" }
  ]
}
```

`terminal:send` is gone because nothing uses it. `settings:own` holds the chosen project repo.

**Calling the CLI:** `spawn(process.execPath, [cliEntry, ...args])`, with no shell. `cliEntry` resolves relative to the plugin, at `../../out/cli/index.js`, which needs `pnpm build:cli`. The worker's env already has `ELECTRON_RUN_AS_NODE=1`, and with S0 the right profile.

---

## 8. Agent profiles and tickets

```markdown
---
name: frontend
display_name: 'Fiona'
department: engineering
model: <my pick>
sprite: 👩‍💻
scope: ['apps/web/**', 'packages/ui/**']
tools:
  allow: ['Edit', 'Bash(npm run lint*)', 'Bash(npm test*)']
  deny: ['Edit(team/**)', 'Bash(git push*)'] # file rules use Edit(); Claude warns Write() never matches
inherits: base-rules.md
---

## Rules

- Tailwind only, no CSS-in-JS
- Components under 150 lines; split otherwise
- No new dependencies without asking me
- Reuse components from packages/ui first
```

```markdown
---
id: T-102
title: Login UI
agent: frontend
scope: ['apps/web/login/**'] # optional; narrows the agent's scope
depends_on: [T-101]
---

What to build, and how I'll know it's done.
```

Malformed profiles or tickets show as an error card in the panel naming the file and field. They never crash the worker or block the rest of the list.

**Starter roster:** PM (Product, edits `team/specs/**` only) · Planner (Engineering, edits `team/tickets/**` only) · Frontend · Backend. PM and Planner run in the primary checkout, because their output has to land where the worker reads it.

**base-rules.md:** no new deps without asking · no function over 50 lines · no one-caller abstractions · delete the dead code you replace · no comments that repeat the code.

**What the compiler writes into a worktree:**

- `CLAUDE.local.md`: base + role rules + `context.md` + ticket. Phase 0 confirmed it loads as memory, so it should survive compaction better than a prompt
- `.claude/settings.local.json`: permissions from `tools`, merged with any existing file
- Both paths added to `<git-common-dir>/info/exclude`, confirmed to keep `git status` clean. That file is shared by all worktrees, so entries are added once

The launch prompt stays short: `Work on T-102 as described in your instructions.`

**Open decision: permission mode.** Launched agents ran in bypass, so nothing asks before acting except what `deny` stops, and `allow` lists are decoration. Two options:

- **Bypass + `deny` + F13 (assumed for now):** least friction. Limits are `deny` rules, and I review every diff.
- **Prompts on:** needs tracing where the launch default comes from and retesting. Closer to "approvals one click away", but with more interruptions.

Either way `tools.allow` is not a safety feature while bypass is on, and bypass agents can run any command `deny` doesn't match.

**Start-ticket sequence (worker):**

1. Validate: profile exists, every `depends_on` is `closed`, no overlap with `running` or `review` tickets. Otherwise stop and show why.
2. Set the ticket to `starting` and persist it. A second click is ignored.
3. `orca worktree create --name T-102 --repo <id> --json`, and persist `worktreeId`, `worktreePath` and `baseRef`.
4. Compile the profile into the worktree.
5. `orca agent launch --agent claude --worktree <selector> --model <m> --surface terminal --prompt "…" --json`
6. Persist `outcome.handle`, `outcome.paneKey` and `operationId` from the launch result, and set the ticket to `running`.

**On failure:** set `failed`, store the error, keep the worktree, and offer **Retry** (resumes at the first incomplete step) and **Discard** (`orca worktree rm`). A lost launch reply must use `--retry-launch <operationId>`. The CLI docs say never to retry as a fresh launch, because that can double-start an agent.

---

## 9. Mascot states

| Source                                               | Mascot                                                                     |
| ---------------------------------------------------- | -------------------------------------------------------------------------- |
| `working`                                            | typing                                                                     |
| `waiting`                                            | yellow `?`, top of "needs me", desktop notification (question or approval) |
| `blocked`                                            | red `!`, top of "needs me", desktop notification (sign-in and similar)     |
| `done`, no outcome, pane has been `working`          | raised hand then coffee; ticket moves to `review`                          |
| `done` + `success`                                   | same, with ✅                                                              |
| `done` + any other outcome                           | ⚠️ "check me"                                                              |
| `done` before any `working`                          | ignored (startup noise)                                                    |
| `working` with no event for N min (computed on read) | head-scratch                                                               |
| Out-of-scope files at turn end (F13)                 | ⚠️ with the file list                                                      |
| Ticket `failed`                                      | ❌ with the error                                                          |
| Pane with no ticket                                  | visitor at the door                                                        |
| No agent                                             | empty desk                                                                 |

Long-running tools can look stuck, so head-scratch is only a hint. It never triggers anything automatically.

---

## 10. Plan

Each phase has an exit test. Dates are estimates; the gates are what matter.

### Phase 0: prove the risky assumptions (done)

- [x] Add `upstream`, follow `CONTRIBUTING.md`, get a dev build running
- [x] Load a probe plugin with `orca plugin dev load`, see `agent.status.changed` in `orca plugin logs`

Run dev commands with a clean env if working inside Orca: inherited `ORCA_*` variables target the real app, and `CLAUDE*` variables leak into dev agents.

| Gate                | Result                                                                                                       | What it changed                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| G1: CLI from worker | **Pass.** With S0, the worker's CLI reached the dev app; `git` works too                                     | None                                                |
| G2: Status          | **Pass, with changes.** Observed `working`, `waiting`, `done`. Never saw `blocked` from Claude               | §2, §9                                              |
| G3: Pane mapping    | **Pass, with changes.** Launch returns handle and `paneKey`; `terminal switch` reports success               | §2, §8. Not checked visually: the window was hidden |
| G4: Profile sticks  | **Pass for `deny`,** including under bypass. `allow` is untested and moot under bypass                       | §8 decision                                         |
| G5: Git exclude     | **Pass.** Status clean; `CLAUDE.local.md` loaded                                                             | Replaces `.pod/context.md`                          |
| G6: Reap            | **Pass.** Reaped after idle; a new event restarted the worker. Restoring my state isn't tested until Phase 2 | None                                                |
| G7: Chat focus      | **No route found.** `terminal switch` refuses chat handles, so the PM stays a terminal                       | F6                                                  |

Results are also in `FORK.md`.

### Phase 1: live office (about 1 weekend day)

- [x] S0 with a focused test
- [ ] S1 with focused tests
- [ ] `plugins/pod-office/` skeleton and panel build script
- [ ] Event → state → storage; reconcile on start; `rpc` ops `snapshot` and `focusPane`
- [ ] Rooms and emoji mascots, narrow-first; "needs me" strip; blocked notification

**Exit test:** start two agents by hand. Both mascots react within about 2 s, clicking jumps to the right terminal, an agent waiting on me notifies me with the panel closed, and after restarting Orca the office rebuilds correctly.

**Timebox:** if S1 isn't working after one full day, stop and rethink. The fallback is a native right-sidebar React tab: more fork surface, but no frame limits.

### Phase 2: my team (about 1 weekend day)

- [ ] Read `team/agents/*.md` and `team/tickets/*.md`, with error cards
- [ ] Choose the project repo (`orca repo list --json`), saved in plugin settings
- [ ] Compiler, overlap and dependency checks, start sequence with Retry and Discard
- [ ] Stuck hint

**Exit test:** two hand-written tickets become two agents in two worktrees, each following its own profile. An overlapping third ticket is refused with the shared files listed. Killing the worker mid-run loses nothing. `git status` in both worktrees shows no compiled files.

**This is the real MVP.** I'll use it on real work for a few days before Phase 3.

### Phase 3: my workflow (weekend 2)

- [ ] PM and Planner launch buttons; tickets appear on refresh
- [ ] Raised hand → `orca file open-changed`; close a ticket and remove its worktree
- [ ] Scope alerts (F13)
- [ ] Decide whether `orca orchestration` tasks and gates could replace my ticket state, based on using mine
- [ ] Polish: walking animation, sprites, a WebAudio chime while the panel is open, try the full-tab custom editor

### If I run behind, cut in this order

1. Polish and full-tab
2. Scope alerts (F13)
3. Planner automation: write tickets by hand
4. Stuck hint

**Non-negotiable core:** Phases 0–2.

---

## 11. Risks

| Risk                                                    | Likelihood | Mitigation                                                                                                              |
| ------------------------------------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------- |
| S1 is harder than it looks                              | Medium     | Request/response only, reuse `command.invoke`, one-day timebox, native-tab fallback                                     |
| Upstream merges break seams or plugin APIs              | Medium     | Two small seams, deliberate merges, smoke test after each                                                               |
| Worker reaped mid-operation                             | Low        | An in-flight command blocks reaping; steps are persisted and retryable. A very long worktree setup is the case to watch |
| Status semantics change in a Claude Code or Orca update | Medium     | §9 is from one small test run; recheck in the post-merge smoke test                                                     |
| Bypass agents do something a `deny` rule misses         | Medium     | Worktree isolation, F13 after the fact, every diff reviewed, no `git push` rule. See the §8 decision                    |
| Low disk during dev                                     | Medium     | Install plus a running dev app used about 6 GiB on my machine; keep 8 GiB free                                          |
| Dev profile differs from my daily Orca                  | Certain    | `orca-dev` has separate settings and repos, so I set it up once and use it for this project                             |
| Agents race on shared files beyond declared scope       | Medium     | Overlap check is only as good as the scopes; conflicts surface at merge                                                 |

---

## 12. Later

- Seam S3: approve/deny buttons in the office
- Multi-floor office, custom layouts, canvas sprites
- Per-agent cost from session transcripts
- More roles (a Reviewer that runs tests, a changelog writer)
- Other agents, once a per-CLI compiler is worth it
- A packaged build (check first whether auto-update would overwrite my fork)
- Offer S0/S1 upstream, so my fork stays tiny
