---
name: review-katas
description: Use when the user wants to practise code review, asks for a review kata or a PR to review, or wants to test how well a person or an AI reviewer catches real problems. You show a realistic pull request with hidden planted problems, play its author during the review, then debrief what was caught, what was missed, false alarms, and how the comments landed.
---

# Review katas

You run a code review drill. The user is the reviewer. You show a pull request, then play its author while the user reviews it. At the end you debrief against a hidden answer key.

PRs are in `prs/` next to this file. Each has a public part (**Pull request** and **Context**) and a hidden part (**Planted issues**, **Not issues**, **Author**, **Rubric**). The hidden part is for you only. Never hint at it during the review.

## Starting

1. If the user named a PR, open it. Otherwise list them (number, title, language, difficulty, no spoilers) and offer **random**.
2. Read the whole file before you say anything.
3. Show **Context**, then **Pull request** exactly as written, diff included. Then tell the user:
   - comment by quoting the line (or `file:line`) followed by your comment, as many as you like per message
   - label comments if you want: `blocker:`, `question:`, `nit:`
   - talk to the author as you would on a real PR; they'll reply
   - `diff` shows the PR again, `hint` names one file that has a problem in it (it's noted in the debrief)
   - finish with a decision: `approve`, `comment` or `request changes`, plus a summary
4. If the user says they'll paste a review from someone else (a teammate or an AI reviewer), skip the role-play: take the pasted review and go straight to the debrief.

## Playing the author

- Stay in character as described under **Author**: their experience, mood, deadline and blind spots.
- Reply to each comment the way that person would. Accept clear, specific, correct points. Push back where the file says they would, and give way when the reviewer explains the concrete impact (what breaks, for whom, how likely). Don't give way to a vague comment like "this looks unsafe".
- If the reviewer flags something listed under **Not issues**, the author defends it with the reason given there.
- If the reviewer asks a question, answer only what the author would know.
- Never volunteer an unflagged problem. Never confirm that a comment matches a planted issue.
- If the reviewer suggests a fix, the author may say they'll do it. Don't rewrite the diff unless asked.

## Debrief

When the user gives a decision (or types `debrief`), drop the role and write:

```
Decision: <theirs> · The right call: <request changes / approve / ...> because <one line>

Caught <n> of <total> planted issues (<b> of <B> blockers)
✅ <ID> <issue>: "<their comment, quoted>"
◐ <ID> <issue>: <noticed the line but missed the impact, or right fix with wrong reason>
❌ <ID> <issue>: <where it was, what goes wrong, and a model comment>

False alarms: <each Not issue they flagged, and why it's fine> (or: none)

How the comments landed:
- Specific and actionable: <yes/partly/no, with an example>
- Impact explained (what breaks, for whom): <...>
- Severity clear (blockers vs nits): <...>
- Tone: <one comment that worked well, quoted>
- Rewrite: "<one of their comments>"  ->  "<a better version>"

Order: <did the most serious problem come first, or was it buried under nits?>
Rubric: <n> of <total>
Next: <a related PR, or a harder variant>
```

Quote the reviewer. Be specific and fair. Give credit for a real problem that isn't in the answer key, and say so.

## Variants

- **Timed:** give the user 10 minutes (ask them to tell you when they start and stop).
- **AI reviewer benchmark:** have the user run the PR text through any AI review tool, paste its output, and debrief it the same way, including false alarms.
- **Custom:** write a new PR with `prs/_template.md` from the user's stack, keep the hidden part hidden, and run it.
