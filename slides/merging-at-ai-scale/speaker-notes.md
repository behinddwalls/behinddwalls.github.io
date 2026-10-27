# Merging at AI Scale — speaker cues

30-minute prerecorded talk · 31 slides. Three short cues per slide.

## Run of show

| Slide | Start | Length | Title |
| --- | --- | --- | --- |
| 01 | 00:00 | 00:20 | Merging at AI Scale |
| 02 | 00:20 | 00:35 | About us |
| 03 | 00:55 | 00:40 | Uber builds in monorepos |
| 04 | 01:35 | 00:40 | Two green PRs. One untested combination. |
| 05 | 02:15 | 00:40 | A lands green. B turns main red. |
| 06 | 02:55 | 00:55 | SubmitQueue speculates over futures |
| 07 | 03:50 | 01:20 | How SubmitQueue scales |
| 08 | 05:10 | 00:15 | Then the arrival rate changed. |
| 09 | 05:25 | 00:45 | AI changed the pace of integration |
| 10 | 06:10 | 00:35 | Green trunk. Hours to land |
| 11 | 06:45 | 00:30 | What we changed |
| 12 | 07:15 | 00:55 | Admission gating |
| 13 | 08:10 | 00:50 | Admission gating: the tradeoff |
| 14 | 09:00 | 01:00 | Deferrals |
| 15 | 10:00 | 00:45 | Deferrals: the tradeoff |
| 16 | 10:45 | 01:15 | Any target overlap could serialize work |
| 17 | 12:00 | 00:55 | Conflict relaxation |
| 18 | 12:55 | 01:20 | Jaccard scoring for dependency overlap |
| 19 | 14:15 | 01:40 | Buy parallelism by dropping the weakest bridges |
| 20 | 15:55 | 00:45 | Assign new arrivals. Leave in-flight work alone. |
| 21 | 16:40 | 00:55 | Conflict relaxation: the tradeoff |
| 22 | 17:35 | 00:45 | The cost: trunk can turn red |
| 23 | 18:20 | 01:50 | Stovepipe: post-merge validation |
| 24 | 20:10 | 02:00 | Validate from the last known-good baseline |
| 25 | 22:10 | 01:25 | Separate the integration tip from the trusted ref |
| 26 | 23:35 | 01:20 | Detection is only the start of recovery |
| 27 | 24:55 | 02:00 | More throughput. Shorter waits. |
| 28 | 26:55 | 01:05 | SubmitQueue and Stovepipe in the open |
| 29 | 28:00 | 01:10 | What comes next |
| 30 | 29:10 | 00:30 | Keep confidence green, even when trunk isn’t |
| 31 | 29:40 | 00:20 | Thank you |

## Speaking cues

### 01 · Merging at AI Scale

00:00–00:20

- AI-driven change volume forced us to rethink merging.
- Follow three interventions, then the shift to post-merge confidence.
- Correctness still matters. When and where we validate is what changed.

### 02 · About us

00:20–00:55

- Preetam Dwivedi: Staff Software Engineer.
- Manjari Akella: Senior Engineer.
- Both on Developer Platform at Uber.

### 03 · Uber builds in monorepos

00:55–01:35

- Six major monorepos span Go, Java, Web, Python, iOS, and Android.
- 5K+ contributors, 36K+ commits/week, 27K PRs/week, and 440M+ lines of code.
- Shared code enables collaboration, but integration has a shared cost.

### 04 · Two green PRs. One untested combination.

01:35–02:15

- A and B branch from the same base H.
- Both PRs pass their own CI. Neither has landed.
- Green results in isolation do not establish that the changes work together.

### 05 · A lands green. B turns main red.

02:15–02:55

- A lands: H+A passes.
- B lands on top: H+A+B fails, despite B’s green CI on H.
- We need confidence in the integrated state, not just isolated PRs.

### 06 · SubmitQueue speculates over futures

02:55–03:50

- For A followed by B, consider H+A, H+B, and H+A+B.
- Build plausible futures before preceding requests finish.
- Land using passing outcomes consistent with what actually happened.

### 07 · How SubmitQueue scales

03:50–05:10

- A target is a build or test unit; target analysis separates independent work.
- Probabilistic speculation prioritizes likely futures.
- Bypassing lets ready work pass slow predecessors when required results agree.

### 08 · Then the arrival rate changed.

05:10–05:25

- We had optimized how changes land.
- AI changed how quickly they arrive.
- The same integration machinery faced a very different workload.

### 09 · AI changed the pace of integration

05:25–06:10

- January: about 90 changes per hour at peak.
- March: about 130. Growth arrived in months, not years.
- March was the tipping point. What did that feel like for developers?

### 10 · Green trunk. Hours to land

06:10–06:45

- Landing at peak could take 3 to 4 hours.
- The cost was developer waiting, despite existing optimizations.
- Our response: manage the load first, then change the merge model.

### 11 · What we changed

06:45–07:15

- First, shape incoming submission traffic with admission gating.
- Then, make the platform own waiting and release through deferrals.
- Finally, relax conflict constraints to unlock more independent merge progress.

### 12 · Admission gating

07:15–08:10

- First, keep expensive submissions out during peak hours.
- Smaller changes enter; rejected authors must resubmit later.
- This protects active capacity, but puts scheduling onto developers.

### 13 · Admission gating: the tradeoff

08:10–09:00

- Business-hour average landing rate: about 130 → 160 diffs/hour; +23% after gating.
- These are business-hour averages, not peak landing rates. The percentage uses unrounded values.
- The cost: developers must return and resubmit. Deferrals automate that waiting.

### 14 · Deferrals

09:00–10:00

- Accept the costly request once and hold it durably.
- Recheck release conditions and admit it when eligible.
- The platform owns waiting and release, not the developer.

### 15 · Deferrals: the tradeoff

10:00–10:45

- Submit once; release automatically.
- The waiting and validation cost still exist.
- When deferred work enters the active queue, its dependencies still constrain progress.

### 16 · Any target overlap could serialize work

10:45–12:00

- The original rule treats relevant target overlap as a yes-or-no conflict.
- One bridge can couple otherwise separate groups of changes.
- We needed to distinguish strong dependency signals from weak ones.

### 17 · Conflict relaxation

12:00–12:55

- Rank overlaps instead of treating every conflict equally.
- Retain stronger links and relax selected weaker links.
- The overlap score helps choose which groups can make independent progress.

### 18 · Jaccard scoring for dependency overlap

12:55–14:15

- Jaccard similarity is shared targets divided by distinct targets across both changes.
- Direct overlap has weight 1; new dependency-edge overlaps have weight 0.5 in each direction.
- Take the strongest signal. This ranks overlap; it is not a failure probability.

### 19 · Buy parallelism by dropping the weakest bridges

14:15–15:55

- Six changes at 50% desired parallelism gives a target of three independent groups.
- Retain the strong links: 0.90, 0.80, and 0.70. Drop the weak bridges: 0.08 and 0.04.
- This is a graph-level choice, not one universal score cutoff.

### 20 · Assign new arrivals. Leave in-flight work alone.

15:55–16:40

- Assign each new arrival using the conflict policy.
- Preserve placement for work already being validated.
- Avoid reshuffling useful speculative builds. This is not a fixed global merge order.

### 21 · Conflict relaxation: the tradeoff

16:40–17:35

- After relaxation in July, throughput was roughly 240 to 250 landed requests/hour.
- More independent progress, but some combinations land without being validated together.
- Checks are not waived. We need confidence in the states that actually land.

### 22 · The cost: trunk can turn red

17:35–18:20

- Individually green changes can still fail together after landing.
- The combined state can leave trunk red even though both PRs passed.
- Stovepipe validates landed states and tracks which ones are known good.

### 23 · Stovepipe: post-merge validation

18:20–20:10

- Observe landed states, choose the head and baseline, build and test, and record the result.
- Validate from a known-good baseline; update it only after a passing result.
- The question: at which commit do we know this branch is green?

### 24 · Validate from the last known-good baseline

20:10–22:10

- A is known green. Validation at D fails, so the baseline stays at A.
- Next validate G relative to A, including the still-unverified range.
- If G passes, advance the baseline. That does not certify every intermediate state.

### 25 · Separate the integration tip from the trusted ref

22:10–23:35

- Main is the newest integrated state; a verified ref is the latest eligible known-good state.
- Failed or pending validation does not advance the trusted ref.
- Consumers must choose to honor that confidence signal.

### 26 · Detection is only the start of recovery

23:35–24:55

- A broken mainline triggers an on-call alert.
- On-call prepares a fix or revert, gets approvals, and lands the change.
- Stovepipe keeps validating and records when main is green again.

### 27 · More throughput. Shorter waits.

24:55–26:55

- As of October 2026: around 300 successful lands per hour at peak, excluding branch requests.
- Land-time p95 is around 30 minutes outside incidents, not a guaranteed upper bound.
- The next step is making this approach reusable and available in the open.

### 28 · SubmitQueue and Stovepipe in the open

26:55–28:00

- Both SubmitQueue and Stovepipe are being built in the open.
- SubmitQueue handles pre-merge validation; Stovepipe handles post-merge validation.
- The open-source repository contains both systems.

### 29 · What comes next

28:00–29:10

- Batch submissions in SubmitQueue to reduce repeated validation and merge work.
- Move toward project-scoped confidence instead of one global status.
- Invest in automated diagnosis, fixes, and reverts. These are future directions.

### 30 · Keep confidence green, even when trunk isn’t

29:10–29:40

- Shape demand when validation capacity is scarce.
- Spend serialization on the strongest dependency signals.
- Make known-good state explicit. Confidence still needs an owner.

### 31 · Thank you

29:40–30:00

- Thank you for watching.
- SubmitQueue and Stovepipe are being built in the open.
- The WeAreDevelopers and Seattle Systems recordings provide more background.

## Public resources

- [Preetam Dwivedi · LinkedIn](https://www.linkedin.com/in/behinddwalls/)
- [WeAreDevelopers · Merging at Scale](https://www.youtube.com/watch?v=mPssttsZknY&t=144s)
- [Seattle Systems · Merging at Scale](https://www.youtube.com/watch?v=O4sCvZAMC9w)
- [Open-source SubmitQueue](https://github.com/uber/submitqueue)
