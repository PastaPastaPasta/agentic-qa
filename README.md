# agentic-qa

A Claude Code skill for large, evidence-backed QA passes run by parallel subagents on real
devices, emulators, simulators or browsers. The output is a segmented report (executive summary,
user-story matrix with proof, severity-sorted defect ledger, per-stream reports, evidence
gallery) published to GitHub Pages. An optional fix loop gives each defect its own PR with
before/after QA.

It is distilled from the Dash Wallet Android v12 QA run. Report:
https://pastapastapasta.github.io/dash-wallet-v12-qa/

- `SKILL.md`: the procedure
- `references/prompts.md`: the prompts that drove the run, with reusable templates
- `references/templates.md`: agent briefs and report skeletons

## Install

```sh
git clone https://github.com/PastaPastaPasta/agentic-qa ~/.claude/skills/agentic-qa
```

Then ask Claude Code for a "full QA pass" on a branch, or invoke `/agentic-qa`.
