# review-katas

[ภาษาไทย](README.th.md) · Part of the katas series with [staff-katas](https://github.com/nuttakulsv/staff-katas), [incident-katas](https://github.com/nuttakulsv/incident-katas) and [system-design-katas](https://github.com/nuttakulsv/system-design-katas)

Most of us learned code review by doing it on real PRs, missing things, and finding out in production. Nobody tells you what you missed. Nobody tells you that your comment made the author defensive, or that you spent four comments on naming while a migration was about to lock the orders table.

review-katas gives you realistic pull requests with planted problems. Your AI agent plays the author: they answer your comments, push back when you're vague, defend the things that are actually fine, and give way when you explain the real impact. When you submit, you get a debrief: what you caught, what you missed and why it mattered, false alarms, and how your comments landed, with one of them rewritten.

And now that more and more PRs are written by agents, reviewing well is half the job.

## What a review looks like

An abridged example from PR 03:

```
Agent:  PR: "Retry PSP capture on failure" by Jordan
        > Fix for Tuesday's incident. If capture fails we now retry up to 5 times...
        [diff]

You:    capture.ts `psp.captures.create(...)` inside withRetry: blocker: if the first
        attempt times out after the PSP already captured, the retry charges the
        customer again. Can we pass an idempotency key per order?

Agent:  Jordan: The PSP has never double charged us in years though. Is this a real risk?

You:    Their docs say a timeout doesn't mean the request failed, and that's exactly
        what happened Tuesday: 30s responses with a 10s timeout. The key makes the
        retry safe, it's one header.

Agent:  Jordan: OK, that's fair. I'll add `Idempotency-Key: capture-${order.id}`.

You:    request changes

Agent:  Decision: request changes · The right call: request changes (double charge risk)
        Caught 3 of 6 planted issues (1 of 1 blockers)
        ✅ B1 double charge on retry: "if the first attempt times out after the PSP..."
        ❌ M3 secret key in logs: `logger.warn({ err })` logs the axios config,
           including the Authorization header. Model comment: ...
        Rewrite: "retries look risky" -> "Retrying a 402 declined card won't help..."
```

## The PRs

| # | PR | Language | Difficulty | You'll practise |
|---|---|---|---|---|
| 01 | Forgot password | Python, Flask | Easy | Auth flows, secrets in logs, a product trade-off |
| 02 | Order status | Ruby, Rails, Postgres | Medium | Migrations on big tables, rolling deploys, data loss |
| 03 | Retry the capture | TypeScript, Node | Medium | Idempotency, retries, secrets in error objects |
| 04 | Profile cache | Go | Medium | Concurrency, shared mutable state, cache staleness |
| 05 | Daily revenue | Python, Django | Medium | Time zones, cron, query counts |
| 06 | Pure refactor | TypeScript, Express | Hard | Reviewing big PRs, not rubber-stamping a senior |
| 07 | The agent wrote it | Python, FastAPI | Medium | Reviewing AI-generated code, tenant isolation, tests that test nothing |

Each PR also has things that look wrong but are fine. Flagging them counts as a false alarm, and the author will tell you why.

PR files live in [`skills/review-katas/prs/`](skills/review-katas/prs). They contain the answers, so don't open one before you review it.

## Install

Claude Code:

```
/plugin marketplace add nuttakulsv/review-katas
/plugin install review-katas@review-katas
```

Codex, Cursor, or anything else that reads SKILL.md:

```
npx skills add nuttakulsv/review-katas
```

Then ask for "a review kata", "PR 6" or "a random PR to review".

**No coding agent?** Paste [`SKILL.md`](skills/review-katas/SKILL.md) and a PR file into any chat assistant and say "start the review". Have a teammate paste it if you want the surprise.

## During a review

- Quote the line (or `file:line`) and write your comment. Several per message is fine.
- Label if you like: `blocker:`, `question:`, `nit:`.
- Talk to the author as you would on a real PR.
- `diff` shows the PR again. `hint` names a file with a problem in it (noted in the debrief).
- Finish with `approve`, `comment` or `request changes` and a summary.

## Test your AI reviewer

Every PR doubles as a small benchmark. Give the Context and Pull request sections to any AI review tool, paste its review back, and say "debrief this review". You get the same scorecard: what it caught, what it missed, and its false alarms. Seven PRs won't rank tools, but they'll show you what yours tends to miss.

## Use it with a team

- **Onboarding reviewers:** new people do 01, 03 and 07 before they review alone.
- **Review guild:** everyone reviews the same PR separately, then compare debriefs. The differences in what people notice are the interesting part.
- **Interviews:** a 30-minute review says more about seniority than a coding puzzle. What a candidate prioritizes, and how they say it, is most of the signal.
- **Your own stack:** describe your codebase and ask the agent to write a PR with [`_template.md`](skills/review-katas/prs/_template.md).

## Write a PR

Copy the template. A good kata PR looks like something a real colleague would open, has one or two problems that would cause a real incident, a few smaller ones, at least one thing that looks wrong but isn't, and an author with a personality and a reason to push back. PRs (the GitHub kind) welcome.

## References

- Google's engineering practices: [How to do a code review](https://google.github.io/eng-practices/review/reviewer/)
- [Conventional Comments](https://conventionalcomments.org/), for labelling comments

## License

MIT
