---
name: agentic-qa
description: Run a parallel, evidence-backed QA pass on an app or branch (devices, simulators, emulators, or browsers) and publish a segmented report with screenshots/videos and optional defect-fix PRs. Use when asked for 'full QA', 'robust QA', a release QA review, 'test all user stories', or to re-run/extend an agentic QA pass.
---

# Agentic QA

This procedure worked on the Dash Wallet Android v12 cutover (2026-09-19). An orchestrator
ran 12 device streams, one static-review stream and six device-reproduction streams on 8
emulators. That was about 40 emulator-hours and produced 118 user stories, ~120 defects
(15 S1), 2,500 screenshots and 105 videos. After that, a fix loop gave each ship blocker its
own fix agent and an independent before/after validator. `references/prompts.md` has the
exact prompts that drove it. `references/templates.md` has the briefs and report skeletons the
orchestrator wrote.

Reference run: the report is at https://pastapastapasta.github.io/dash-wallet-v12-qa/. The
original workspace (plan, briefs, helper scripts, site builder) was `~/workspace/dash-wallet-qa/`.
If you have access to it, reuse its `bin/` scripts when the target is Android.

Use the phases and the output shape below. Adapt everything else (platform, tiers, stream count,
tooling) to the project in front of you.

## What made the report better than a normal QA pass

1. **An orchestrator and many subagents, not one agent.** The main session plans, builds the
   harness, writes briefs, dispatches streams, merges results and publishes. It does not do the
   testing itself. Each stream is one subagent that owns one device and one theme. Use Opus
   subagents for testing and validation. Use a stronger model only for involved code fixes.
2. **Scope comes from the diff.** Diff the branch against the last release. Group the changes by
   area and rank the risk (core / every screen that reads the changed seam / regressions in
   untouched flows). Also test beyond the diff: regression sweep, localisation, accessibility,
   edge cases, huge data sets, upgrade paths, a mainnet or production read-only pass.
3. **A baseline build next to the candidate.** Build the last release too, sign both with the
   same throwaway key so an in-place install is a real upgrade, and re-run every suspected
   regression on the baseline. "Pre-existing" and "regression" are different verdicts.
4. **A claim without a file path is not a result.** Take a screenshot at every step, record
   video for money or upgrade flows, pull logs after every test, and save the exact grep that
   proves a point as a `.txt` next to the screenshots. BLOCKED with a reason is a valid result.
   Invented progress is not.
5. **A static-review stream, then device reproduction.** One stream reads the code with no
   device and files code-level findings (SR-xx) with file:line. A second wave takes each SR item
   to a device and marks it REPRODUCED / PARTIAL / NOT REPRODUCED / BLOCKED. This caught the
   worst bug in the run: a watchdog that cancelled itself.
6. **Question expected values.** The brief's expected balance was stale. The agents checked it
   against the chain explorer instead of reporting a false failure.
7. **Harness knowledge feeds back into the briefs.** Each stream reports environment problems
   separately from product defects. The orchestrator appends them to the brief as "harness notes
   from <stream>" so the next wave does not rediscover them.

## Phase 1 — Plan and build the harness

- Identify the base, candidate and baseline refs. Read the change set. Write `QA-PLAN.md` with
  the environment table, change areas ranked by risk, a tiered test matrix (A = the user's named
  scenarios, B = core-risk depth, C = money/data flows on the changed seam, D = secondary
  features, E = regression sweep plus static checks), the execution model (which stream owns
  which device and tiers) and the severity scale.
- Set up everything the subagents need without human help: SDKs, emulators/simulators,
  signed builds of baseline and candidate, an `env.sh`, and a helper CLI covering
  install/reinstall/launch/kill/UI dump/tap-by-text/screenshot/record/pull-logs/grep-log/memory
  sampling/network toggle/locale/font scale/lock-unlock/exit-info. Subagents should never need
  to improvise device plumbing. Ask the user for sudo only if it is actually needed.
- Test funds and fixtures: faucet scripts, throwaway accounts, shared read-only reference
  fixtures (mark these NEVER MUTATE in the brief), and a synthetic "massive" data set if size
  matters.
- Write `AGENT-BRIEF.md` (template in references). Every stream reads it first.

## Phase 2 — Run the streams (`/goal`)

- Launch the streams in parallel. Each gets one device serial, an evidence dir
  `evidence/<stream>/`, its tier/test IDs and a time budget. Group work so long waits (syncs,
  confirmations) overlap with shorter tests.
- Use `/goal <condition>` so a Stop hook keeps the orchestrator working until the report exists.
  Watch for `evidence/*/REPORT.md` appearing with Monitor, not sleep loops.
- Keep a running `DEFECT-LEDGER.md` while reports arrive. Use one ID space (D-001…), dedupe
  across streams (list every stream that saw a defect), record status (candidate / confirmed /
  pre-existing / resolved-not-a-bug), and keep ENV-n rows separate. When the orchestrator wants
  something characterised, send targeted follow-up repros (baseline check, DB diff,
  escape-hatch matrix) to a stream.
- Severity: **S1** money/data wrong or lost, security bypass, crash loop, never finishes;
  **S2** needs user action to recover, memory kill, wrong history, privacy leak;
  **S3** flow bug with a workaround; **S4** cosmetic or log noise. Mark regressions and
  pre-existing issues explicitly.

## Phase 3 — Assemble the report

Output shape (keep it; the value is in the segmentation):

1. `EXEC-SUMMARY.md` has the verdict first, then what was tested (builds, devices, counts), a
   table of ship blockers grouped by area (ID, defect, reproduced on device?), release gates
   that are not product bugs, S2 clusters, S3/S4 themes, what passed cleanly (keep this as
   regression evidence), caveats and not-covered items, and a **suggested fix order**.
2. `STORY-MATRIX.md` has one row per user story: `| # | story | verdict | proof paths | defect IDs |`
   with verdicts PASS / PASS+ (works, defects noted) / FAIL / BLOCKED / N/R.
3. `DEFECT-LEDGER.md` has the consolidated register **sorted by severity**. Each row carries
   title, streams, status, a repro/root-cause note with file:line and evidence paths.
4. Per-stream `evidence/<stream>/REPORT.md` holds the detail.
5. `QA-REPORT.md` concatenates all of the above. `evidence/index.html` is a gallery.

## Phase 4 — Publish to GitHub Pages

- Build a static site (`site/`): convert screenshots to WebP, copy videos and small text/log
  files, and render every markdown file to HTML.
- **Every evidence path in the report must be a clickable link** that opens the file (turn code
  spans and paths into anchors). Give defect and story rows anchors
  (`DEFECT-LEDGER.html#D-037`).
- Ledger pages are sorted by severity.
- Explain black screenshots on the page. Secure screens (FLAG_SECURE, password fields) capture
  as black by design, so capture a UI-hierarchy dump there instead.
- Redaction: ask first. If every seed/key/log is a throwaway test artefact, publish them
  verbatim (that is what the user chose). Never publish anything tied to real funds or
  identities.
- Push to a dedicated public repo with Pages enabled, check the live URLs, then share the link.

## Phase 5 (optional) — Reproduce, fix, validate, PR

- **Reproduce statically-found items.** Brief: `SR-REPRO-BRIEF.md`. Each stream takes a few SR
  items, proves or disproves them on a device, and updates the ledger. Fault injection through
  the debug bridge is allowed. Changing the app is not.
- **Fix.** One subagent, worktree and branch per defect (or per tightly coupled group). Use
  Opus for straightforward fixes and the strongest model for involved ones. Brief:
  `FIX-BRIEF.md` requires a minimal root-cause fix, a unit test that fails before and passes
  after, a signed build, and a **QA validation recipe** (steps that fail before and must pass
  after, plus the log lines that prove it). No push, no PR.
- **Validate.** A different Opus subagent (brief: `QA-FIX-BRIEF.md`) runs BEFORE on the base
  build and AFTER on the candidate with identical steps, then regression checks and at least two
  adversarial variations. The verdict is FIXED / PARTIALLY FIXED / NOT FIXED / FIXED BUT
  REGRESSION, and the report ends with the 3–6 best artefacts for the PR.
- **PR.** One PR per defect onto the branch under test (stack only if they depend on each
  other). Put key screenshots/clips inline in the PR body and link a per-PR page on the Pages
  site (`pr/<topic>/`). Follow the project's PR template and babysit the PRs.
- If the branch moves while you work, rebuild the base and re-check the sync-type items on the
  new head before fixing them. They may already be fixed.

## Gotchas from the first run

- The orchestrator's context fills up. Keep all state on disk (plan, ledger, notes.md per
  stream) so a `/compact` loses nothing.
- Subagents are sometimes blocked from writing their REPORT.md. Tell them to put the full
  report in their final message as a fallback, then save it verbatim.
- Stream IDs collide when streams file defects in parallel. Reserve ID ranges per stream or
  renumber at merge time and record the mapping.
- Shared rate limits (faucets, APIs) need a per-wallet "request once" rule. Headless faucet
  clients may get Cloudflare-banned, so use a real browser tool.
- Android specifics: `adb root` kills running logcat pipes. Emulators clamp RAM. Shell/root
  broadcasts bypass not-exported receivers (false positives, so `adb unroot` first). To stall an
  app's network without flipping connectivity, use `iptables -m owner --uid-owner <uid> -j DROP`.
  Put greps inside `adb shell "…"` to avoid broken pipes. Tap PIN pads by resource-id with ≥1.2 s
  gaps.
- Rate limits on the strongest model can kill fix agents midway. Resume them with SendMessage
  (context intact) instead of relaunching.
