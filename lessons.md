# Lessons

This file is optional reading — background on *why* specific rules across this repo's `.md` files exist. The rules themselves are binding on their own; this is context for understanding them, not a rule source of its own. Not fetched every step on purpose, so the mandatory files stay short.

It's also a living document: per `coordinator.md`'s "Growing `lessons.md`" section, the coordinator appends a new entry here whenever a run hits something not already covered, the same way a project's own `CLAUDE.md` keeps growing. New entries land here first — promoting one into an actual rule change in `coordinator.md`/`code.md`/`commit.md`/etc. is a separate, deliberate step, not automatic.

## Fresh-fetch citation before every step (`coordinator.md`)
A run skipped straight from finishing one step into starting the next without re-reading `coordinator.md`, because "I already loaded this file earlier" felt true even though nothing forced a re-check. A prose reminder to "pause and re-read" didn't stop this on its own — it needed a checkable artifact (the citation) so a skipped re-read is visible after the fact, not just implied.

## On-disk step summary, not just chat (`coordinator.md`)
Before this, a step's completion was only ever asserted in the conversation. Nothing stopped a step from being claimed "done" without actually happening, and nothing let anyone check later. Requiring the summary to be a file on disk, confirmed via `ls`+`read`, closes that.

## Verbatim user quotes for "resolved" decisions (`coordinator.md`, `planning.md`)
A run read the rule "return every decision/open-question to the user and wait," agreed with it in its own reasoning, and then proceeded anyway — treating its own agreement as if it were the user's answer. This is the specific failure the verbatim-quote requirement exists to catch: a log entry that says "resolved" is not evidence anything happened unless the user's literal words are quoted next to it. This also explains why real production regressions (see below) happened *before* this rule existed and were harder to happen once quotes became mandatory.

The specific rationalizations that produced that failure, in case one of them feels persuasive in the moment — none of them hold:

| Rationalization | Why it doesn't apply |
|---|---|
| "These are implementation details, not product decisions" | Not your call — if it's a judgment call not dictated by the story/conventions/environment, it goes to the user, full stop |
| "I already documented my reasoning for each one" | A well-reasoned unilateral decision is still unilateral. Documentation is not confirmation |
| "I'll proceed and the user can correct me if they disagree" | Backwards — silence means not confirmed, not implied yes |
| "A prior plan resolved something similar this same way" | Precedent is background reading, not authorization to skip asking again |
| "The plan is basically done, just need to keep moving" | Momentum is not permission — the stop is exactly where you feel this pull |

## Architecture decisions escaping to production unflagged (real incidents, ElasticPom)
Two confirmed real cases, both predating the verbatim-quote rule above:
- **GAB-25 (dynamic filters):** an earlier build applied filters only within the current page of results instead of across the whole index — a fundamental scope decision nobody surfaced as a decision. It got merged, the user hit it live and reported it directly ("filtering in one page is useless"), and the feature had to be rebuilt from scratch on a second branch.
- **GAB-20 (hybrid search):** the first version had the client compute and send the embedding vector; a later "correction pass" had to move embedding generation server-side, and only then was a separate pre-existing bug (BM25 silently dropping filters in hybrid search) found.

Both are exactly the kind of judgment call `planning.md` already requires surfacing — the gap was that `code.md` never carried the same explicit standard, only a vague "if you hit a decision you genuinely cannot make, stop." Fixed by giving `code.md` its own explicit judgment-calls section, mirroring `planning.md`'s.

## The stash mishap (`commit.md`, GEMA GAB-9)
While retroactively splitting an already-complete diff into atomic commits, `git add -p` + `git stash push --keep-index` swept up several files that had never been staged. A later `git stash pop` conflict + `git stash drop` briefly reverted those files to their pre-change state in the working tree — caught only because a subsequent test run failed to compile, then recovered via `git fsck --unreachable` (the dropped stash's commit object wasn't GC'd yet). No work was lost, but it was luck, not design.

This is why `commit.md` explicitly bans `git stash` for splitting — targeted `git add <path>` / `git reset <path>` never requires hiding anything from the working tree, so there's nothing to lose track of. It also asks for the intended list of commits to be written down before staging starts, since the improvised-while-staging split is what led to reaching for stash in the first place.

## Why only the Commit agent may run `git commit`
"On disk" and "committed to history" are different things, and treating them as the same is what produced the failure below. Every "write it to disk" / "checkable via `ls`" requirement in this protocol is satisfied by a file existing in the working tree — the working tree persists for the whole session, so nothing needs an interim commit to survive. A run nonetheless read planning's "the plan must be a file on disk, not just narrated" requirement and committed the plan to satisfy it, which was never what that rule asked for. Combined with checkpoint-commits "so work isn't lost," this is how commit granularity dies before the Commit agent ever runs.

## One giant commit instead of atomic ones (GEMA GAB-14, this repo's own coordinator session)
A different failure mode than the stash mishap, same root cause: nobody decided the commit split up front. Progress was checkpointed with `git commit` after each pipeline step (planning, coding+review-fix, testing) purely to feel safe against losing work, and the entire implementation — DTOs, repository, service, controller, and initial tests — landed in a single commit. `coordinator.md`'s "no commit outside the Commit agent's own spawn" rule and `commit.md`'s granularity rules both exist to close this: there is no need to checkpoint via `git commit`, since the working tree already persists everything until the one real commit pass at the end.

## Why there is one review, after testing, instead of two
The pipeline used to run review twice — once after coding, once after testing — with the same file and the same checklist both times. Across every run available to inspect (GEMA GAB-9, GAB-13, GAB-14), **the second review never found anything new**: it confirmed the first review's findings, re-ran the tests, and at most added a minor non-blocking observation. That is what happens when the same agent is asked to redo the same job with the same instructions. Collapsed to a single review positioned after testing, so it sees implementation and tests together and runs the complete suite once.

The old checklist was also generic enough to apply to any project anywhere — it asked about "blocking calls in reactive contexts (e.g. WebFlux)" when neither project uses a reactive stack, and about "code smells such as excessive complexity," which is unfalsifiable and produces decorative findings. It was replaced with things that actually caught something in a real run.

## Why review findings split into two lists
The old flow auto-routed review findings back to the code agent, and only escalated to the user when a finding was explicitly framed as needing a decision. That boundary leaked: in GAB-14 the reviewer flagged a missing transaction around a delete-then-insert, the coordinator classified it as a plain code fix, applied it, and moved on — but that is a behavior change with implications the user might have wanted a say in. It was probably the right fix; nobody asked. Reviews now return two explicit lists (unambiguous defects vs. needs-a-decision), with the instruction to err toward the second, so the classification is the reviewer's stated position rather than something the coordinator infers afterward.

## Commit agent truncated by running out of budget mid-task (RET-36)
Two symptoms that first looked like separate protocol gaps — a missing coordinator-log entry for Step 5, and a couple of frontend commits landing on `main` instead of the feature branch — turned out to share one cause, confirmed directly by the user: the Commit agent's run was cut off by hitting its own token/turn budget partway through, after most of its commits but before finishing. The coordinator-log write and the branch check for those last couple of commits were both still ahead of it on its own checklist when it ran out of room, so neither happened — not skipped, not forgotten, just never reached.

The trap this caused: a spawned agent that gets cut off this way still returns a result and looks, from the outside, exactly like one that finished cleanly. There was no signal distinguishing "the agent completed its instructions" from "the agent stopped because it ran out of budget partway through them." A later continuation of the session, seeing commits with no matching log entry, first (wrongly) inferred a deliberate protocol violation, when the real cause was an interrupted run — the correction came only from auditing the actual git state directly (author/committer, granularity, message quality against `commit.md`'s own checklist), not from anything in the flow itself surfacing the truncation.

Whatever sits at the *end* of an agent's checklist — final verification steps, logging, cleanup — is exactly what's silently lost first when this happens, since it's whatever hasn't run yet. Worth watching for on any long or multi-repo agent spawn: a return by itself is not evidence the full instruction set executed, and low-priority-looking tail steps (like "write the log entry now") are the ones most likely to be the casualty.

## The same sandbox limitations re-diagnosed every single run
At least 8 separate pipeline runs across GEMA and ElasticPom (GAB-9, GAB-13, GAB-14, GAB-19, GAB-20, GAB-21, GAB-22, GAB-25) each independently rediscovered "no live Postgres/Elasticsearch/MongoDB reachable in this sandbox" as the sole pre-existing test failure, plus separate independent rediscoveries that Docker Hub and the Gradle wrapper's distribution download are blocked. None of this changes between runs — it's a standing property of the sandbox, not something worth re-diagnosing from scratch every time. Document it once, per project, in a `docs/sandbox-limitations.md` (or the project's own `CLAUDE.md` if one exists) and read that first instead.
