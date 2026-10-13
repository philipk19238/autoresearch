# Fork-and-Flush

How a metric loop gets out of an idea basin. Source: Cai et al., "Fork-and-Flush: Escaping Idea Basins in
Autoresearch Agents" ([arXiv:2610.07447](https://arxiv.org/abs/2610.07447)).

## Contents
- Why
- When to run a round
- One round
- Orchestrator
- Cost
- Opt out

## Why

A single loop keeps improving inside one family of ideas (an idea basin) and plateaus there. Its own chat
context holds it there: every new idea is reasoned from the old ones. Compaction does not fix this.

Two moves get it out, and they work best together:

- **Flush**: start a new agent from the files only, with no chat history. It has to work out the intent again
  from the artifacts, which is what lets it land in a different basin.
- **Fork**: run several of those fresh agents from the same state and keep the best one.

Paper results, 13 tasks, matched budget (normalized score):

| Single run | Best-of-4 | Flush only | Fork only | Fork-and-flush |
|---|---|---|---|---|
| 0.47 | 0.54 | 0.55 | 0.57 | 0.78 |

The flush-only and fork-only numbers come from the ablation, which ran on 11 of the 13 tasks.

## When to run a round

Run rounds on a **fixed schedule**, not when a plateau is detected. The paper tried plateau triggers and
dropped them: small gains keep resetting the stall clock, so a loop creeping upward inside one basin never
triggers, and 15 of 25 plateau-triggered runs used up their budget before the trigger fired.

| Mode | Rounds |
|---|---|
| Bounded, N ≥ 12 iterations | after 33% and 66% of N |
| Bounded, N < 12 | one round, at 50% of N |
| Unbounded | every 20 iterations |

**Early trigger:** also run a round when the last 5 iterations produced no `keep`. Each early round replaces
the next scheduled one, so the total number of rounds does not grow.

## One round

1. **Protect the incumbent.** The incumbent is the commit with the best metric so far. Tag it
   (`git tag -f autoresearch/incumbent <sha>`). No branch may revert or overwrite it.
2. **Fork K branches.** Default K = 4. Use K = 2 when one Verify run takes more than about 5 minutes, or when
   it needs a single shared resource such as one local stack or one test database. In that case run the
   branches one after another.
   - Each branch is a fresh-context sub-agent in its own git worktree, starting at the incumbent. In Claude
     Code: the `Agent` tool with `isolation: "worktree"`, one call per branch, all in one message when they can
     run in parallel. On other platforms: whatever starts a sub-agent with an empty conversation, on its own
     checkout.
   - Branch budget: `max(2, round(0.06 × N))` iterations, where N is the loop's total iterations. Unbounded
     loops use 2.
3. **Flush: what a branch is told.**
   - Give it: Goal, Scope, Metric, Direction, Verify, Guard, its iteration budget, the path of the results TSV,
     and the rule that the incumbent tag is protected.
   - Do not give it: the leader's hypotheses, plans, ranking of ideas, or a summary of the conversation. The
     results TSV and `git log` are artifacts, so the branch may read them. That is the "files" half of
     fork-and-flush.
   - Do not tell some branches to "try something completely different". In the paper that directive had no
     measurable effect on which branch won.
4. **Each branch runs the core loop** (Phases 1–7) on its own branch, and returns its best metric and its kept
   commits.
5. **Select.** The branch with the best metric wins. On a tie with the incumbent, the branch still wins,
   because its context is fresh.
   - Bring the winner's kept commits onto the loop branch (fast-forward or cherry-pick). Discard the other
     branches and remove their worktrees.
   - If no branch beat the incumbent, the loop branch stays at the incumbent, and the winner's context still
     becomes the leader.
6. **Hand over leadership.** The winning branch's agent continues as the leader until the next round. In Claude
   Code, resume it with `SendMessage` so its context carries on. If the platform can't resume it, the main agent
   continues, but only after re-reading the TSV and `git log`, and it treats its own earlier hypotheses as
   stale.
7. **Log.**
   - One TSV row per branch with status `fork-{k}`, carrying that branch's best metric.
   - One row with status `fork-select`, naming the winner and whether it beat the incumbent.

The loop's final result is the best metric any branch or the leader ever reached, not the leader's last value.

## Orchestrator

`orchestrate.sh verdict` answers `FORK` instead of `PLATEAU` while the state's `fork_rounds_since_progress` is
below 2. On `FORK`, run one round from the incumbent, add 1 to `fork_rounds_since_progress`, and continue the
loop. Reset it to 0 whenever units improve. Once 2 rounds pass with no progress, `verdict` answers `PLATEAU`
and the loop stops.

## Cost

A round costs about K × 6% of the budget. The paper spends roughly 72% of the budget on branches (an opening
round plus two scheduled rounds) and still beats a single run at the same total cost. When Verify is expensive,
use K = 2 rather than skipping rounds: flushing alone recovers about half of the gain.

## Opt out

Use `--no-fork` when the loop is a mechanical sweep with no ideas to get stuck on, for example a parameter grid
or a list of known fixes worked through in order.
