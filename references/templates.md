# Briefs and report skeletons

These are generalised from the files the orchestrator wrote for the Dash Wallet run
(`~/workspace/dash-wallet-qa/`). Copy them into the QA workspace
and fill them in. Cut sections that do not apply.

Suggested workspace layout:

```
<qa-root>/
  env.sh                 # toolchain + QA_* paths, sourced by every shell
  bin/                   # helper CLI (device driver), build, faucet, site builder, report builder
  QA-PLAN.md  AGENT-BRIEF.md  REPORT-TEMPLATE.md
  DEFECT-LEDGER.md  STORY-MATRIX.md  EXEC-SUMMARY.md  QA-REPORT.md
  builds/                # baseline + candidate, same throwaway signing key
  evidence/<stream>/<test>/NN-description.png|mp4|txt, REPORT.md, notes.md
  site/                  # rendered Pages site (git repo, pushed to GitHub)
  pr/<topic>/            # per-fix worktrees (phase 5)
```

---

## AGENT-BRIEF.md (every test stream reads this first)

```markdown
# QA agent briefing (read fully before touching a device)

You are one of several parallel QA agents testing <app>, <branch/version> (<one-line summary of
the change>). You own exactly ONE device: <serial>. Never touch another device, never kill
shared processes, never rebuild. <Safe environment> only, throwaway accounts only.

## Setup, every shell
source <qa-root>/env.sh ; APP=$QA_ROOT/bin/<helper>   # read `cat $APP` once for every subcommand
Builds: baseline <file> (<version>), candidate <file> (<version>). Same key, so reinstall = real upgrade.
App logs: <where they live and how to pull/grep them>.

## Driving the UI
<how to dump UI text, tap by text/id, type, go back, take screenshots; which screens expose no
text so you must screenshot and read the image; PIN/credentials to use>.
Waiting: poll with the helper plus logs, never assume. Screenshot every time something changes.

## Evidence discipline (non-negotiable)
- Evidence dir: $QA_EVIDENCE/<stream>/<test-id>/. Screenshot EVERY named step as NN-description.png.
- Record video for flows that move money/data or cross an upgrade boundary.
- Capture logs for the whole session plus resource samples (memory/CPU) at an interval. Pull
  app logs after every test and after any crash, and save crash/exit info.
- Save the exact grep/command output that proves each claim as a .txt next to the screenshots.
- A claim without a file path is not a result. Never fabricate. If you could not observe
  something, say BLOCKED and why.
- If an expected value from the brief disagrees with what you see, verify it against an
  independent source (explorer, API, DB) before calling it a failure.

## Shared fixtures and limits
- Reference fixture <X> is READ-ONLY: <what must never be done with it>.
- Funding/rate-limited services: <how to request, the limit, "request once per account, never loop">.

## Reporting
Write $QA_EVIDENCE/<stream>/REPORT.md following REPORT-TEMPLATE.md. Keep notes.md as a running
timestamped journal so nothing is lost if you are cut off. If you cannot write REPORT.md, put
the full report in your final message. Your final message to the orchestrator is the summary
table, the defects table and paths.

Severity: S1 money/data wrong or lost, security bypass, crash loop, never finishes · S2 needs
user action to recover, memory kill, wrong history, privacy · S3 flow bug with workaround ·
S4 cosmetic. Say whether each defect is a regression (reproduce on the baseline) or pre-existing.

## Useful log patterns
<grep patterns for the risky subsystems>

## Harness notes from earlier streams
<appended by the orchestrator as streams report environment problems>
```

---

## REPORT-TEMPLATE.md (per stream)

```markdown
# QA report: <stream / test IDs>   (agent, device, date/time window)

## Summary table
| Test | Verdict (PASS/PASS+/FAIL/BLOCKED) | Key evidence file(s) | One-line finding |

## Per-test detail
### <ID> <title>
- Build(s) + version
- Steps executed (numbered, wall-clock timestamps)
- Observations (log lines quoted verbatim with timestamps and file)
- Assertions: actual vs expected
- Screenshots / videos: paths
- Resources: peak memory etc. from the sample file; any kill/crash info
- Verdict + reasoning

## Defects found
| ID | Sev | Title | Repro steps | Evidence | Suspected area / file:line / commit | Regression? |

## Environment problems (not product defects)
## Not run / blocked, with reason
```

---

## Consolidated artefacts (orchestrator)

**STORY-MATRIX.md**, grouped by tier:

```markdown
| # | User story | Verdict | Proof (paths under evidence/) | Defects |
```

**DEFECT-LEDGER.md**, sorted by severity, one ID space, ENV rows at the end:

```markdown
| ID | Sev | Stream(s) | Title | Status (candidate/confirmed/pre-existing/resolved) + root cause w/ file:line | Evidence |
```

**EXEC-SUMMARY.md** sections: Verdict · What was tested (builds, devices, counts of stories /
screenshots / videos / hours) · Ship blockers table grouped by area (ID, defect, reproduced on
device?) · Release gates (non-product) · S2 clusters · S3/S4 themes · What passed cleanly ·
Not covered / caveats · Suggested fix order.

---

## SR-REPRO-BRIEF.md (static finding → device)

```markdown
Goal: the static-review stream produced code-level findings (SR-xx) that have not been shown on
a device. REPRODUCE your assigned items and capture logs + screenshots + video that PROVE or
DISPROVE each. NOT REPRODUCED, with what you tried, is a valid and valuable result.

Read your SR rows and open the cited file:line in the candidate worktree (and the baseline to
decide regression vs pre-existing). Fault injection through the device bridge (permissions,
killing processes, network drops, low-memory signals) is fine. Do not modify or rebuild the app.
Time budget ~<N> h. Start any item with long waits first.

REPORT.md: header (device, build versions, fixtures, times) · one row per item
| SR-id | REPRODUCED / PARTIAL / NOT REPRODUCED / BLOCKED | what you did | decisive evidence | notes |
· a detail section per item (steps, timestamps, verbatim log lines, what the user sees, whether
the impact statement needs correcting) · new defects · environment problems.
```

---

## FIX-BRIEF.md (one fix agent per defect)

```markdown
You are fixing ONE defect (<ID>) from the QA report (<Pages URL>, ledger row, evidence dir).
Work only in your worktree <qa-root>/pr/<topic>/ on branch fix/<topic> based on <base ref>.

Rules: minimal root-cause fix, no drive-by refactors. Match the surrounding style. Add unit
tests that fail before and pass after, run them and the existing tests of every touched file,
and run the formatter. Use a conventional commit that references the defect ID. Stage only
the files you changed. Do not push or open a PR.

Deliverable (final message):
1. Root cause with file:line.  2. Changes + why they are minimal.  3. Tests + results.
4. Commit SHA.  5. Built artefact path (or build error).
6. QA validation recipe: exact steps that FAILED before (expected wrong behaviour) and must
   PASS after, the log lines/greps that prove it, and regression checks around the change.
7. Risks, follow-ups, uncertainties.
```

---

## QA-FIX-BRIEF.md (independent validator per fix)

```markdown
You are validating ONE candidate fix (<ID>) interactively before it becomes a PR. Given: the
fix branch/worktree, candidate build, base build (same head without the fix), the fix agent's
recipe, and your device + evidence dir $QA_EVIDENCE/PR-<topic>/.

Read the diff first. You are the reviewer's eyes: judge whether it fixes the ROOT CAUSE in the
ledger row, not just the recipe's symptom.

1. BEFORE on the base build: run the recipe's failing steps and capture the wrong behaviour
   precisely (numbers, log lines, screenshots, video for money/sync flows).
2. AFTER on the candidate: the same steps and the same artefacts, with the proving log lines quoted.
3. REGRESSION: the surrounding user story still works, no new crashes, no new warning/error storms.
4. ADVERSARIAL: at least two variations the recipe did not mention (timing, kill point, locale,
   rotation, low memory, offline).
5. Run the fix's unit tests.

REPORT.md: header · Verdict FIXED / PARTIALLY FIXED / NOT FIXED / FIXED BUT REGRESSION (one
sentence why) · BEFORE, AFTER, regression and adversarial tables · "PR evidence": the 3–6 best
artefacts (before shot, after shot, decisive log excerpt, one video) · new findings and code
review concerns. Stop early if the candidate crashes on launch or is obviously wrong.
```

---

## PR body shape (per fix)

```markdown
<What was wrong, user-visible impact, QA ID + link to ledger row>
## Root cause   (file:line)
## Fix
## QA
Verdict: FIXED — <one line>. Full report: <Pages>/pr/<topic>/
| | Before (base <sha>) | After (<sha>) |
|---|---|---|
| <step> | <inline screenshot> | <inline screenshot> |
<decisive log excerpt> · <video link> · regression + adversarial summary
## Tests
```
