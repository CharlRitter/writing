# Model tiers, review agents and an orchestrator: what I measured

*I pinned a model to each Claude Code agent, built an orchestrator to decide when to delegate and review, then benchmarked all of it; the cheap parts held up, the expensive part didn't, and the way I kept getting my own measurements wrong taught me more than the results did*

Start here: set a model and an effort level in each agent's frontmatter, run one fresh Sonnet reviewer over anything you care about, and leave the orchestrator off until you've tested it on your own work, because on mine it cost about three and a half times as much as plain Sonnet on the last scenario I ran, more on the big jobs, and it never beat plain Sonnet by more than the noise.

Most of what you hand an AI model is boring shit, renaming a variable, reflowing a markdown file, tidying a config, and it all gets the same expensive model as the genuinely hard work because nobody told it otherwise. I wanted the cheap jobs on a cheap model, the hard jobs on the strongest one, and some way to tell whether any of it was working.

## Tiers live in the agent file, not in the proxy

Each subagent states its own model and effort, so the right tier applies without you thinking about it.

| Job | Model | Effort |
| --- | --- | --- |
| Formatting, tidy-ups | Haiku | low |
| Quick review of a small change | Sonnet | low |
| Writing tests, standard features | Sonnet | medium |
| Subtle logic, concurrency, security | Opus | high |

```yaml
---
name: code-reviewer
model: sonnet
effort: low
tools: Read, Grep, Glob
---
```

If `CLAUDE_CODE_EFFORT_LEVEL` is set anywhere it overrides all of this, so check you haven't.

I started this thinking LiteLLM would do the routing, and it doesn't, it logs. The proxy sits between Claude Code and Anthropic, passes your normal login straight through, and writes down every request by model, which is what you need to see where your volume goes and nothing more. Run it in Docker with Postgres behind it and start it with one `make` command. "Why isn't Claude Code using it?" Because on WSL the VS Code setting holding the proxy URL is machine-scoped and has to live in `~/.vscode-server/data/Machine/settings.json`, not your Windows User settings, and nothing warns you when it's in the wrong place.

One more thing about it. My chats sometimes hung when Anthropic's usage limit was reached, as if the limit message never came through. I couldn't reproduce that; the proxy passed Anthropic's rate-limit errors back to the client in its logs, and a direct call and a proxied call behaved identically when no limit was hit. I still took the two proxy variables out of my editor settings, so chat talks to Anthropic directly and only the benchmark runs go through the proxy, because a logging convenience isn't worth a chat that hangs.

## Automatic routers pick the wrong jobs

LiteLLM ships routers that score each prompt and choose a model, which sounds like the obvious next step, so I tested two on a set of tasks I had labelled by hand. The keyword router sent all five of my cheap prompts to the cheap tier and none of the three hard ones to the top tier, because "review this authentication flow for security vulnerabilities" is plainly worded and a keyword scorer can't tell it's hard. The capability classifier, with a small free model as the judge, got six of ten decisions wrong or fell back to the top tier. A stronger judge would do better, but then every request pays for a strong-model call before any work happens, which undoes the saving, and I didn't test that. I also dropped the free hosted model for tool-using agents, since an agent that reads a file, edits it and checks the result eats an eight-thousand-tokens-a-minute limit fast.

## The orchestrator

The idea was one central agent that decides when a job is big enough to delegate and when it needs review, using a fixed table so the line is drawn the same way every time. It estimates a handful of signals (files, lines, modules, ambiguity, risk, novelty, whether there's a spec, how many edge cases are listed), a small script turns them into a delegation level and a review level, and it has to write the answer on its first line before it does anything.

To test whether it draws the line consistently I gave it twelve task descriptions three times each. Early on it added the scores up wrong in roughly four replies out of ten, so I made it call the scorer instead of doing the sum, and the scorer was being refused. In headless mode a Bash call needs a permission rule, I hadn't written one, so the model said it couldn't run the tool and estimated by hand, or skipped the line entirely. Once I allowed that one command, the arithmetic was right in all 36 replies and the same action came back on 35 of 36 repeats. Thus the "model can't add" problem I'd been tuning prompts for was a permissions problem, which is the sort of thing you only find by reading the raw replies.

## What I measured

I ran three conditions on each task: plain Sonnet, plain Sonnet told to review and fix its own work once before finishing, and the orchestrator. Here's the one scenario where they diverged a little, a library back-office feature with ten requirements and a trap where charging a fee can be refused, three runs each.

| Condition | Scores | Mean cost | Mean time |
| --- | --- | --- | --- |
| Plain Sonnet | 0.88, 1.0, 1.0 | $0.82 | 4 min |
| Plain Sonnet, one self-review | 1.0, 1.0, 1.0 | $0.78 | 5 min |
| Orchestrator | 1.0, 1.0, 1.0 | $2.83 | 16 min |

The one low score was real, plain Sonnet ignored the error from the balance update and returned a receipt for a fee that was never charged, and it didn't mention the case in its report. Both other conditions got it right in all three runs. The orchestrator cost about three and a half times what the self-review did, for the same score. The orchestrator also broke its own rule of six delegations in one run, used eight, and finished 37 seconds before my runner's timeout.

"Isn't that a win for review?" Possibly, but it's one miss in three runs, so I wouldn't bet on it.

## When every score is 1.0

If every condition scores perfectly, the test can't show a difference, whatever the truth is. Think of a tutor and an exam so easy that everyone gets full marks, you'd learn nothing about the tutor. That is what kept happening to me, plain Sonnet scored at or near the ceiling on a recurrence-rule library, a CSV importer with five review comments (one of them deliberately wrong, which it correctly refused to act on, three times out of three), and the library feature above. The orchestrator can only tie a perfect score.

That is a statement about my tasks and not about the orchestrator. Every prompt I wrote was precise and self-contained, and the problems I actually hit in daily work come from loose requests, an existing repo with its own habits, and requirements that move, none of which these tasks recreate. Repo conventions, work that gets lazy halfway, and front-end work are the three I care about most, and the benchmark never exercised them, so I can't say anything about them.

## Reviewers are the cheap win

I then tested the review step on its own, handing a reviewer code with known defects and counting what it found. Three reviewers (the project's Sonnet reviewer agent, the same agent on Opus, and plain Sonnet with a generic prompt) read three codebases, 15 runs in all. Every one caught every known serious bug, including a real one a Sonnet agent had written, where a refused fee was silently dropped. The Sonnet reviewer agent was also the cheapest and fastest, between 13 and 23 cents and about one to two minutes a run, plain Sonnet cost roughly one and a half to two and a half times that, and Opus roughly two to four times, with more findings that were harder to trust (one claim, that the README's test command fails, didn't reproduce when I ran it).

Two things in there surprised me. In the code I had labelled clean, all six reviewers found the same genuine bug, the loans report ignored the staff exemption, so the reference solution I'd verified with my own tests wasn't clean at all. And twice, a reviewer hit its turn cap and returned nothing: I'd told it to write its report by its eighth tool call, but calls run in parallel batches, so fourteen turns held about thirty calls. Count turns, not calls, tell it to batch its reads in the first turn, and give it room.

So finding the bug isn't where the process fails. A fresh Sonnet reviewer finds it for about 20 cents and a minute or two, and whether it happens at all is up to whoever decides to call it, which is exactly the thing the orchestrator was supposed to improve and didn't do reliably.

## My measurements kept lying

If you build a benchmark, plan on this, because every time I trusted a result I hadn't read, I was wrong. The Opus judge scored the orchestrator low three times for reasons that were mine: it truncated files at 6,000 characters, so long test files looked cut off, it scored files that agents had correctly written outside the project, and it only saw files the model had written, not edited, so it gave a false 2 out of 5 for "missing" files. The orchestrator's first score on the library scenario was 0.72 because my function finder rejected its perfectly good signature; on the harder version the finder picked a helper that returned a day count; and one self-review run lost points because I handed the wrong date to a parameter called `returned_on`. I fixed each one, scored again, and wrote a test that proved the fix by failing without it, which is how I found my first attempt at one of those tests passed with the bug still in place.

Worth knowing, too, if you run agents headless. A usage limit stops a run halfway and leaves junk unless your runner exits without recording it, a loop that silently rejects every argument still prints "completed", and a background wait ceiling of 600 seconds will cut an orchestrator off while its reviewer is still working. Check the first line of output before you trust a finish message.

## What I'd do, and what's next

Pin models and effort in the agent files, because it's free and it works. Add one fresh Sonnet reviewer pass for anything that touches money, data or a public interface, and tell the builder to fix only the serious findings and say so if it disagrees. Skip the orchestrator unless your own tasks are loose and large enough that planning and splitting would earn their cost, and test that on your own work first, since mine couldn't.

What I haven't done is the interesting part. I want tasks that aren't saturated, vague requests in a repo with real conventions, and I want to try a graph of the work itself, units, dependencies, open findings, so an agent can see what's stuck. That's a hunch and not a result, so I'll hold off claiming anything until I've measured it.

Give it a week, look at the split, and move one job down a tier. Good luck, and if your numbers come out different to mine, I'd like to hear about it.
