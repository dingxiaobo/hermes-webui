# Agent instructions for Hermes WebUI

This file is the shared entry point for AI assistants working in this repository.
Keep it project-specific and safe to publish. Do not put personal machine setup,
private network details, credentials, tokens, or local-only workflow notes here.

## What this repo is

Hermes WebUI is a browser frontend for [Hermes Agent](https://hermes-agent.nousresearch.com/).
It is deliberately a **Python + vanilla JavaScript** app with no build step, no
bundler, no frontend framework. The Python server lives in `server.py` + `api/`
and the frontend is plain `<script>` tags in `static/` plus a single `style.css`.
Do not add a build pipeline, framework, or bundler without explicit justification
and a rollback story — this is a hard design constraint, not a preference.

## First two minutes

1. Read `README.md` (quick start, features, env vars, Docker).
2. Read `CONTRIBUTING.md` (PR shape, what goes in the description, AI disclosure).
3. Read `docs/CONTRACTS.md` (subsystem routing) and follow links into the area
   you will touch. Start with `docs/rfcs/README.md` for runtime/streaming work
   and `docs/UIUX-GUIDE.md` + `DESIGN.md` for UI work.
4. For install / bootstrap / provider / first-run / Docker / WSL support, read
   `docs/onboarding-agent-checklist.md` **before** running commands or reading
   logs. It has hard safety rules and the role split between human and assistant.
5. `ARCHITECTURE.md` is the deep reference (modules, endpoints, data flow).
   `TESTING.md` is the manual browser test plan + automated test commands.

## Commands an agent actually runs

| Goal | Command |
|---|---|
| Run the full test suite locally | `./scripts/test.sh` |
| Run a focused test set | `./scripts/test.sh tests/test_regressions.py -v` |
| Pin a specific Python for tests | `HERMES_WEBUI_TEST_PYTHON=/path/to/python3.12 ./scripts/test.sh` |
| Launch the server (foreground) | `python3 bootstrap.py` |
| Launch as a daemon (PID at `~/.hermes/webui.pid`) | `./ctl.sh start` / `status` / `logs --lines 100` / `restart` / `stop` |
| Ship-style launcher | `./start.sh` |
| Static-JS runtime guard (`no-const-assign` / `no-import-assign`) | `npm run lint:runtime` (after `npm install --no-save eslint`) |
| Python forward-only lint gate (curated ruff rules on **changed lines only**) | `python3 scripts/ruff_lint.py --diff origin/master` |
| Whole-tree ruff backlog (informational, never blocks) | `python3 scripts/ruff_lint.py --all` |
| Reference undefined-globals gate (#3696 class) | `python3 scripts/scope_undef_gate.py` |
| Headless browser brick-class smoke | `python tests/browser_smoke.py` (needs `playwright install chromium`) |

> **Always use `./scripts/test.sh`, never bare `python3 -m pytest` or `pytest`.**
> The script creates/uses the repo `.venv`, pins to Python 3.11–3.13, and
> installs the dev test deps (`requirements-dev.txt`). A direct pytest on an
> unsupported system interpreter is the #1 way to get a misleading failure
> during validation. If you see "unsupported interpreter", rerun through
> `scripts/test.sh` before debugging product code.

> The forward-only ruff gate (CI `lint` job + `scripts/ruff_lint.py`) only
> reports findings on **lines the PR adds or modifies** vs `origin/master`.
> The existing tree has a cosmetic F401 backlog tracked in #3273; do not
> reformat it as drive-by cleanup. The cleanest way to silence a real
> false positive is `# noqa: <CODE>` with a one-line reason.

## Architecture in one screen

- **Backend**: `server.py` is a thin routing shell (`BaseHTTPRequestHandler`,
  `ThreadingHTTPServer`); all GET/POST handlers live in `api/routes.py` (large,
  if/elif dispatch — no decorator framework). Helpers in `api/helpers.py`
  (`j`, `bad`, `require`, `safe_resolve`); security headers set in one place.
- **Auth**: optional password + passkeys in `api/auth.py`. Off by default; set
  `HERMES_WEBUI_PASSWORD` to enable. Cookie is signed HMAC, HTTP-only, 24h TTL.
- **SSE streaming**: `api/streaming.py` + `api/streaming.py:_run_agent_streaming`.
  Watch the `/api/upload` ordering rule in `do_POST` — multipart reads the
  request body, so the upload check must come **before** any `read_body()`.
- **Sessions**: `api/models.py` (plain class, JSON-per-file, in-memory LRU with
  `_index.json` for `all_sessions()`). `INFLIGHT` and the active-sid guard in
  `static/messages.js` prevent session-switch-mid-flight from clobbering state.
- **Frontend modules** in `static/`: `ui.js`, `workspace.js`, `sessions.js`,
  `messages.js`, `panels.js`, `commands.js`, `boot.js`, `onboarding.js`,
  `i18n.js`, `login.js`, `icons.js`, `sw.js`, `terminal.js`,
  `assistant_turn_anchors.js`, `outline.js`, `pwa-startup.js`. Loaded as
  classic `<script>` tags sharing one implicit global scope — see
  `scripts/scope_undef_gate.py` for the brick class that catches.
- **State directory** (runtime, **outside the repo**): `~/.hermes/webui/`
  contains `sessions/`, `workspaces.json`, `last_workspace.txt`, `settings.json`,
  `projects.json`. Override with `HERMES_WEBUI_STATE_DIR`.
- **Logs**: `~/.hermes/webui/bootstrap-8787.log` for `start.sh`/`bootstrap.py`,
  `~/.hermes/webui.log` for `ctl.sh`. Structured JSON per request.

## Repo-specific gotchas agents miss

- **No build step.** Editing `static/*.js` or `static/style.css` takes effect
  on the next browser reload. There is no `npm run build`, no `dist/`, no
  transpile step.
- **Process-global env vars for the agent run.** `TERMINAL_CWD`,
  `HERMES_EXEC_ASK`, `HERMES_SESSION_KEY`, `HERMES_HOME` are set via
  `os.environ` in `_run_agent_streaming`. Two concurrent chat requests clobber
  each other; this is single-user/single-concurrent-request safe. Tracked as
  Architecture Phase B in `ARCHITECTURE.md`. Don't add per-request isolation
  in your PR — it's a known architecture migration, not a local fix.
- **State coupling to hermes-agent.** WebUI imports Agent modules directly
  (`api/config.py`, `api/providers.py`, `api/streaming.py`) and reads Agent
  state layout directly. The release boundary in `README.md` Compatibility
  section says: **upgrade WebUI and hermes-agent together**, record both
  versions in issue reports. This is not a stable API boundary yet (#1925,
  #2491).
- **Tooling is dev-only.** `package.json` is *not* a build manifest; its only
  dependency is `eslint`, used solely as the runtime-error guard. The app
  itself remains pure Python + vanilla JS.
- **Auto-discovery, not config files, picks the agent.** `bootstrap.py` and
  `start.sh` walk `HERMES_WEBUI_AGENT_DIR` env, then `$HERMES_HOME/hermes-agent`,
  then a sibling `../hermes-agent`, then auto-install via the official Hermes
  installer. Trust auto-discovery; do not hardcode a path in a script.
- **Classic-script shared global scope.** A function declared *inside* another
  function is **not** global. Calling it bare from a sibling top-level scope
  throws `ReferenceError` at runtime (only). `scripts/scope_undef_gate.py`
  exists to catch this class — run it after editing any `static/*.js`.
- **`api/routes.py` is large** (≈770KB, thousands of lines). Prefer the
  helper functions in `api/helpers.py` (`j`, `bad`, `require`, `safe_resolve`)
  and the existing per-module separation over weaving new logic into the
  monolithic dispatch.
- Keep one logical change per PR; split unrelated refactors or cleanup.
- Read `docs/CONTRACTS.md` and the linked contract/RFC for the touched
  subsystem before editing.
- For local pytest runs, use `./scripts/test.sh` instead of bare `python3`,
  `python -m pytest`, or `pytest`. The script creates/uses the repo `.venv`,
  pins execution to Python 3.11-3.13, and installs missing dev test dependencies.
  `HERMES_WEBUI_TEST_PYTHON` selects the supported base interpreter used to
  create or rebuild `.venv`; it must not install test dependencies into a
  system/Homebrew interpreter directly.
  If a direct pytest invocation reports an unsupported interpreter, rerun through
  `./scripts/test.sh` before debugging product code.
- Prefer the existing Python + vanilla JavaScript structure. Do not add
  dependencies, build tools, frameworks, or long-lived processes without clear
  justification and a rollback story.
- Update docs when changing setup, onboarding, runtime behavior, architecture,
  testing guidance, or user-facing workflows.
- Do not edit `CHANGELOG.md` in ordinary contributor PRs. The release workflow
  owns changelog updates through release commits. If a change is release-note
  worthy, include concise release-note wording in the PR body instead.
- For UI or UX changes, include before/after evidence and test relevant
  desktop, narrow, and mobile states.
- For behavior changes, add or update automated tests where practical and list
  the manual verification performed.
- For runtime, streaming, recovery, replay, compression, or sidebar metadata
  changes, name the state layer being mutated and prove the relevant invariant.

## Onboarding / install / first-run safety

The repo reads and writes real agent state, sessions, workspaces, credentials,
and cron data. **Treat local validation as potentially destructive** unless
you have confirmed the active state directories.

Hard rules (from `docs/onboarding-agent-checklist.md`):

- Use isolated `HERMES_HOME` and `HERMES_WEBUI_STATE_DIR` for trials unless
  the human explicitly asks to use real state.
- Never delete or overwrite a real `~/.hermes` directory without explicit
  approval.
- Never print API keys, OAuth tokens, cookies, full `.env` files, full
  `auth.json` files, or password hashes.
- Collect non-secret status and log evidence before recommending a fix:
  `/health`, `/api/onboarding/status`, structured log tail, and `find` of
  the state dir.
- Do not modify real cron jobs, real sessions, real profiles, or real memory
  files during an onboarding trial.
- Do not expose WebUI on a public interface without password protection
  (`HERMES_WEBUI_PASSWORD`) and explicit human approval.
- Do not proxy or tunnel `localhost` / `127.0.0.1` / private LAN / Docker
  container loopback paths through external services.

Isolated trial pattern:

```bash
mkdir -p ~/hermes-onboarding-test
HERMES_HOME=~/hermes-onboarding-test/.hermes \
HERMES_WEBUI_STATE_DIR=~/hermes-onboarding-test/webui \
HERMES_WEBUI_PORT=8789 \
python3 bootstrap.py
# open http://127.0.0.1:8789
# log at ~/hermes-onboarding-test/webui/bootstrap-8789.log
```

## PR / change shape

- One logical change per PR; split unrelated refactors and cleanup.
- Read `docs/CONTRACTS.md` and the linked contract/RFC for the touched
  subsystem before editing. Contract-affecting PRs need a `Contract Routing`
  section in the PR body naming the contract family and evidence used.
- For runtime/streaming/recovery/replay/compression/sidebar-metadata changes,
  name the state layer being mutated and prove the invariant (regression test
  or explicit manual check).
- For UI/UX changes, include before/after evidence and test relevant desktop,
  narrow, and mobile states. Mobile breakpoints matter; touch targets ≥44px.
- Update `CHANGELOG.md` for user-visible behavior, setup, workflow, or doc
  changes that should be release-note ready. Update the matching
  `README.md` / `ARCHITECTURE.md` / `TESTING.md` / `docs/CONTRACTS.md` in
  the same PR.
- PR body must contain: Thinking Path, What Changed, Why It Matters,
  Verification, Risks / Follow-ups, Model Used. If AI helped, disclose
  provider + exact model + notable tool use. If no AI: `None -- human-authored`.
- Do not silently redefine product behavior by changing tests alone — update
  the corresponding docs in the same PR.

## Files agents should not reach for first

- `mcp_server.py` — the MCP integration is a separate surface; check whether
  the task is about WebUI proper or the MCP server before reading.
- `docs/why-hermes.md` — competitor comparison; only relevant for product
  positioning / messaging edits, never for code work.
- `CONTRIBUTORS.md` — auto-generated credit roll; do not hand-edit.
- `CHANGELOG.md` (full file) — append-only; consult the latest entries for
  release-note style but do not rewrite history.

## What this file is not

It is not a re-statement of `README.md` / `CONTRIBUTING.md` / `ARCHITECTURE.md`
/ `TESTING.md` / `docs/CONTRACTS.md`. Read those for the full story. This file
captures the agent-specific shorthand — commands, gotchas, and safety rules —
that an agent would otherwise miss or guess wrong.

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **hermes-webui** (26315 symbols, 52973 relationships, 300 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> Index stale? Run `node .gitnexus/run.cjs analyze` from the project root — it auto-selects an available runner. No `.gitnexus/run.cjs` yet? `npx gitnexus analyze` (npm 11 crash → `npm i -g gitnexus`; #1939).

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows. For regression review, compare against the default branch: `detect_changes({scope: "compare", base_ref: "master"})`.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `context({name: "symbolName"})`.

## Never Do

- NEVER edit a function, class, or method without first running `impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `rename` which understands the call graph.
- NEVER commit changes without running `detect_changes()` to check affected scope.

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/hermes-webui/context` | Codebase overview, check index freshness |
| `gitnexus://repo/hermes-webui/clusters` | All functional areas |
| `gitnexus://repo/hermes-webui/processes` | All execution flows |
| `gitnexus://repo/hermes-webui/process/{name}` | Step-by-step execution trace |

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |
| Work in the Tests area (3774 symbols) | `.claude/skills/generated/tests/SKILL.md` |
| Work in the Api area (1214 symbols) | `.claude/skills/generated/api/SKILL.md` |
| Work in the Static area (934 symbols) | `.claude/skills/generated/static/SKILL.md` |
| Work in the Scripts area (21 symbols) | `.claude/skills/generated/scripts/SKILL.md` |
| Work in the Manual area (11 symbols) | `.claude/skills/generated/manual/SKILL.md` |
| Work in the Cluster_360 area (9 symbols) | `.claude/skills/generated/cluster-360/SKILL.md` |
| Work in the Cluster_369 area (4 symbols) | `.claude/skills/generated/cluster-369/SKILL.md` |

<!-- gitnexus:end -->
