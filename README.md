# opencode-background-agents — house fork

A deployment fork of [`AeonDave/opencode-background-agents`](https://github.com/AeonDave/opencode-background-agents)
(based on **v0.1.1**, MIT) adding a small **supervised lifecycle layer** for
background delegations in [OpenCode](https://opencode.ai).

**Why this fork exists:** in daily orchestrator use, background delegations died
in three annoying ways — their IDs didn't say *who* died, a usage-limit death
meant re-delegating from scratch, and there was no way to put a run on hold.
This fork fixes all three without changing the plugin's async-first model.

---

## What's different from upstream

### 1. Agent-prefixed delegation IDs

IDs used to be purely random (`spare-moccasin-peafowl`). Now the delegation ID
carries the child role:

```
explore-disgusted-yellow-ape
coder-exciting-ivory-horse
```

Notifications, `delegation_list()`, artifacts, and state files all use the
prefixed form, so an ID identifies *what kind of agent* it belongs to at a
glance.

### 2. `delegation_resume(id, prompt?, timeout_minutes?, model?)` — no more re-delegating after a death

A delegation that died mid-run (e.g. **provider usage limit**, transient
provider error) or completed and needs follow-up can be **continued in its
original child session**:

- the ID stays the same; the previous attempt's output is archived as
  `<id>.run<N>.md`;
- a fresh timeout window starts and a new `<task-notification>` is delivered
  at the next terminal state;
- `prompt` is optional — resume after a quota reset with
  *"continue where you stopped"* plus the state so far;
- optional `model` / `timeout_minutes` overrides for the resumed run.

Honest scoping: **`timeout` and `cancelled` delegations are NOT resumable** —
upstream hard-deletes their child sessions, so the tool returns a loud
"re-delegate instead" error rather than pretending.

### 3. `delegation_pause(id)` — freeze a run, keep the session

Pauses a RUNNING delegation: aborts its current turn **without deleting the
child session** and freezes the timeout and debounced-completion timers.
`delegation_resume(id)` picks the same session back up later. A paused
delegation genuinely rests — guards were added so the aborted turn's
`MessageAbortedError` cannot finalize it as `error`, and the completion /
idle-event paths ignore paused records.

New record fields: `resumeCount`, `pausedAt`. New status: `"paused"`
(included in state persistence + restore parsing).

### 4. Supervision lifecycle at a glance

```
delegate ──▶ running ──▶ complete ──▶ delegation_resume ──▶ running …
               │  ▲
               │  └── delegation_steer (mid-run redirect, extends window)
               ├──▶ paused  ──┐
               │              └── delegation_resume (same session, optional prompt)
               ├──▶ error (usage limit, provider death) ──▶ delegation_resume
               ├──▶ timeout / cancelled (session deleted) ──▶ re-delegate
               └──▶ delegation_stop ──▶ partial output kept, session deleted
```

---

## Why not upstream features?

Checked against upstream and the ecosystem at the time of forking (Oct 2026):

- upstream `delegate()` has **no resume-by-ID** — terminal states are final by
  design, so a usage-limit death forced a from-scratch re-delegation;
- [`opencode-drawer-agents`](https://github.com/fredcamaral/drawers) supports
  resume (`bg_task(task_id=…)`), but is a different plugin family and tool
  surface (`bg_*`);
- [`oh-my-openagent`](https://github.com/code-yeongyu/oh-my-openagent) has
  resume + provider-exhaustion fallback, but ships its own agent ecosystem.

This fork keeps the AeonDave tool surface (`delegate`, `delegation_read`,
`delegation_list`, `delegation_status`, `delegation_peek`, `delegation_steer`,
`delegation_stop`) and adds exactly two tools (`delegation_pause`,
`delegation_resume`) — nothing else changes.

## Install (OpenCode ≥ 1.18)

> **Note:** on opencode 1.18.x, npm packages in the `plugin` config array are
> silently never loaded (upstream anomalyco/opencode#33455). The reliable path
> is directory auto-discovery via `~/.config/opencode/plugins/`.

```bash
# 1. install the package as a dependency of the config dir
cd ~/.config/opencode && npm install github:talha7k/opencode-background-agents#main

# 2. shim it into the plugins directory
cat > ~/.config/opencode/plugins/background-agents.ts <<'EOF'
import BackgroundAgentsPlugin from "@talha7k/opencode-background-agents"
export default BackgroundAgentsPlugin
EOF
```

Then restart OpenCode. Verify with a headless session that executes a tool and
check `~/.local/share/opencode/delegations/` appears.

## Provenance

- Base: upstream `v0.1.1` (`692161c`), diff kept minimal (4 files, ~280 lines).
- The same changes were first developed as idempotent patch scripts against the
  npm-installed package; they are preserved for reference in this fork's
  history.
- Everything else (tool surface, storage layout, state schema version,
  environment variables `BACKGROUND_AGENTS_TIMEOUT_MINUTES` /
  `BACKGROUND_AGENTS_STRICT_READONLY`) is upstream behavior — see the full
  [upstream README](docs/UPSTREAM_README.md).

## License

MIT, inherited from upstream. All credit for the original plugin design goes to
[@AeonDave](https://github.com/AeonDave/opencode-background-agents) and the
KDCO / oh-my-openagent lineage it derives from.
