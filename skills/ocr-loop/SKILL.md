---
name: ocr-loop
description: Use when the user says "ocr loop".
---

# ocr-loop

This is a workflow for making your code changes more correct before (or
sometimes after) filing a pull request. tl;dr - have one reviewer model run an
[open-code-review](https://github.com/alibaba/open-code-review) (OCR)
delegate-mode review of your code in a loop.

OCR picks the files and the review rules. The reviewer model does the review.
Make the most minimal possible changes to appease the reviewer. Be careful of
scope creep and don't be scared to push back.

## Budget

- Default: 3 rounds total. Round 1 is full. Later rounds check only
  changed code and open findings.
- Extend to 5 only for security, permissions, auth, data loss, or
  state/concurrency, while a blocker or should stays open with evidence. Or when
  the user asks.
- Stop early if a round returns only nits. Declined nits and comment,
  doc, or format-only fixes need no new round.
- One reviewer only. The reviewer must not start subagents or reviewers.
- Keep a ledger: id, severity, outcome, usage if returned. Never guess it.

## 1. Scope

Run `git status --short`. Do not stash, discard, or include unrelated
changes. Limit review to the change paths if they separate cleanly.
Otherwise stop and ask.

Find the base after `git fetch origin`:

- PR: `gh pr view <n> --json baseRefName,headRefOid`. Check out the branch
  only if clean. Confirm `HEAD` is the PR head, or ahead only by this
  change. Base: `git merge-base HEAD origin/<baseRefName>`.
- Local: `git merge-base HEAD origin/<default-branch>`. No remote: a base
  the user names, or `HEAD` for uncommitted changes only.

OCR range mode compares refs, not the working tree. Before each round, commit
the scoped paths locally, untracked files included, so `<base>..HEAD` is the
whole change. Do not push. If the base is `HEAD` (uncommitted only), skip the
commit and use workspace mode instead.

Run `ocr --version`. If `ocr` is missing, report the blocker and stop. Do not
fall back to a plain review. State intent and out-of-scope in 1-2 sentences.
Run covering checks. Do not send code this change breaks. Note missing or
pre-failing checks in the brief.

## 2. Start the reviewer

Pick the first of these the harness can run, unless the user named one:

1. Mistral 4
2. GLM 5.3
3. Opus 5.5
4. Sol 6.1

In T3, call `orchestrator_capabilities` and match against what it lists. Do not
hardcode an ID. Record provider, model, and effort only if exposed. Use
`delegate_task` every round (`role: "review"`, `mode: "async"`). Save `taskId`.
New `clientRequestId` per round. `childThreadId` is storage, not the next
target. Do not use `t3_thread_send`, `create_threads`, or `t3_thread_launch`
for review.

If T3 tools are not listed, try one direct capability call. Outside T3, use a
native subagent on one of those models and keep its ID. In Claude Code, that
means the `Agent` tool with the model set, or a custom subagent set up to run
it. If no route runs any of them, report the blocker. Do not substitute a model.

Do not edit scoped files while review runs. Wait for the notice. Do not
poll or start a second reviewer. Timeout, error, or no reply is not
approval. On failure, cancel, verify stop, restart with the ledger.

Round 1 brief (T3 children get no history, and may not have the OCR skill):

```
Review only. Do not edit, commit, push, comment on PRs, or start subagents.
Repo: <abs path> Base: <sha> Intent: <intent> Out of scope: <scope> Checks: <results>
Use open-code-review delegate mode. OCR picks files and rules; you review.
Use `git --no-pager` and `--no-ext-diff --no-textconv --no-color` on diffs.
1. ocr delegate preview --format json --from <base> --to HEAD [--exclude <out-of-scope globs>]
   (workspace mode, base is HEAD: drop --from/--to.)
2. ocr delegate rule --format json <every reviewable path>
3. Per file: git diff <merge_base>..HEAD -- "<path>" (workspace: git diff HEAD,
   or read untracked files whole). Apply its rule group. Read surrounding code.
4. Mark every (path, status) reviewed or skipped with a reason. No silent gaps.
If --format is an unknown flag, rerun without it and read the text output.
Find bugs, broken callers, mismatch with intent.
Skip style unless it hides a bug. Hold to the smallest change. Flag code,
tests, or guards the intent does not need. Other paths, rare cases, small
perf, extra compat: nits.
blocker = intended case wrong. should = real bug callers will hit, in intent. nit = rest.
Reply:
VERDICT: APPROVE or CHANGES
COVERAGE: <reviewed>/<total>, skipped: <path: reason>
S1 [blocker|should|nit] <file>:<line> <claim>. Fix: <suggestion> (stable ids)
```

A reply with skipped files and no reason, or coverage below total, is not
approval. Send it back for the missing files.

## 3. Answer findings

Check each finding and severity yourself. Another path, rare case, or
small cost is a nit, whatever label it came with. OCR's critical/high is not
automatically a blocker.

- Agree: smallest fix. Prefer delete or narrow over new lock, cache,
  helper, or option. Fix the case that happened. Add one focused test
  that fails without the fix, if intent needs it.
- Disagree: say why, with evidence (file:line, test, output).
- Out of scope or nit: decline in one line. List as follow-up.

Update the ledger. Re-run checks and `git diff --shortstat <base>`. If
review grew the diff, cut first. Do not agree just to end the loop.

## 4. Next round

Start one only if budget remains and a blocker or should is open, behavior code
changed, or approval does not cover the current behavior diff. Commit the fixes
locally first. Use a fresh `delegate_task` in T3, or follow up the saved native
ID. Send the round 1 brief, the ledger (`fixed:` / `disagree:` / `declined:` +
evidence), the same base, and the commits since last round. Tell the reviewer:
open ids and changed code only. Mark each old id `resolved` or
`still open: <reason>`.

## 5. Stop and ship

Agreed: APPROVE on the current behavior diff, full coverage, no blocker or
should open, no concern of your own. Stuck: an item stays disputed after two
replies, or budget ends. Show both sides in 2-3 lines each. The user decides.
Budget end is not approval.

Do not push during the loop. Push after agreement only if the user asked
to push, ship, or update the PR. If you push to a PR, add above the model
blurb:

`Reviewed with ocr-loop: <N> rounds with <model>. OCR verdict: approved`.

Add `on <effort>` only if verified.

Report: rounds, model, OCR version, what the reviewer changed, diff size
before/after, fixed and declined findings with reasons, coverage, usage if
exposed (else say not), pushed or not.
