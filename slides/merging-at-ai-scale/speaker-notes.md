# Merging at AI Scale — speaker cues

30-minute prerecorded talk · 30 slides. Three short cues per slide.

## Run of show

| Slide | Start | Length | Title |
| --- | --- | --- | --- |
| 01 | 00:00 | 00:20 | Merging at AI Scale |
| 02 | 00:20 | 00:35 | About us |
| 03 | 00:55 | 00:20 | Outline |
| 04 | 01:15 | 00:40 | Uber builds in monorepos |
| 05 | 01:55 | 00:40 | Two green PRs. One untested combination. |
| 06 | 02:35 | 00:40 | A lands green. B turns main red. |
| 07 | 03:15 | 00:55 | SubmitQueue speculates over futures |
| 08 | 04:10 | 01:20 | How SubmitQueue scales |
| 09 | 05:30 | 00:15 | Then came AI |
| 10 | 05:45 | 00:45 | AI changed the pace of integration |
| 11 | 06:30 | 00:35 | Green trunk. Hours to land |
| 12 | 07:05 | 00:30 | What we changed |
| 13 | 07:35 | 00:55 | Admission gating |
| 14 | 08:30 | 00:50 | Admission gating: the tradeoff |
| 15 | 09:20 | 01:00 | Deferrals |
| 16 | 10:20 | 00:45 | Deferrals: the tradeoff |
| 17 | 11:05 | 01:15 | Any target overlap could serialize work |
| 18 | 12:20 | 00:55 | Conflict relaxation |
| 19 | 13:15 | 01:20 | Jaccard scoring for dependency overlap |
| 20 | 14:35 | 01:40 | Buy parallelism by dropping the weakest bridges |
| 21 | 16:15 | 00:45 | Assign new arrivals. Leave in-flight work alone. |
| 22 | 17:00 | 00:55 | Conflict relaxation: the tradeoff |
| 23 | 17:55 | 00:45 | The cost: trunk can turn red |
| 24 | 18:40 | 05:15 | Stovepipe: one history, two moving refs |
| 25 | 23:55 | 01:20 | Recovering a broken trunk |
| 26 | 25:15 | 01:40 | More throughput. Shorter waits. |
| 27 | 26:55 | 01:05 | SubmitQueue and Stovepipe in the open |
| 28 | 28:00 | 01:10 | What comes next |
| 29 | 29:10 | 00:30 | A hybrid approach to validation |
| 30 | 29:40 | 00:20 | Thank you |

## Speaking cues

### 01 · Merging at AI Scale

00:00–00:20

- AI-driven change volume forced us to rethink merging.
- Follow three interventions, then the shift to post-merge confidence.
- Correctness still matters. When and where we validate is what changed.

### 02 · About us

00:20–00:55

- Preetam Dwivedi: Staff Software Engineer.
- Manjari Akella: Senior Software Engineer.
- Both on Developer Platform at Uber.

### 03 · Outline

00:55–01:15

- Merging at Uber: the shared integration problem.
- AI-driven growth and how we adapted our integration model.
- The results and what comes next.

### 04 · Uber builds in monorepos

01:15–01:55

- Six major monorepos span Go, Java, Web, Python, iOS, and Android.
- 5K+ contributors, 36K+ commits/week, 27K PRs/week, and 440M+ lines of code.
- Shared code enables collaboration, but integration has a shared cost.

### 05 · Two green PRs. One untested combination.

01:55–02:35

- A and B branch from the same base H.
- Both PRs pass their own CI. Neither has landed.
- Green results in isolation do not establish that the changes work together.

### 06 · A lands green. B turns main red.

02:35–03:15

- A lands: H+A passes.
- B lands on top: H+A+B fails, despite B’s green CI on H.
- We need confidence in the integrated state, not just isolated PRs.

### 07 · SubmitQueue speculates over futures

03:15–04:10

- For A followed by B, consider H+A, H+B, and H+A+B.
- Build plausible futures before preceding requests finish.
- Land using passing outcomes consistent with what actually happened.

### 08 · How SubmitQueue scales

04:10–05:30

- A target is a build or test unit; target analysis separates independent work.
- Probabilistic speculation prioritizes likely futures.
- Bypassing lets ready work pass slow predecessors when required results agree.

### 09 · Then came AI

05:30–05:45

- AI-assisted development accelerated code production.
- Changes arrived faster than SubmitQueue could land them.
- More arrivals exposed the limits of the existing integration model.

### 10 · AI changed the pace of integration

05:45–06:30

- January: about 90 lands per hour at peak.
- March: about 130, roughly 44% above January.
- March was the tipping point. What did that feel like for developers?

### 11 · Green trunk. Hours to land

06:30–07:05

- Land-time p95 was around 30 minutes before the growth.
- During peak periods, p95 stretched to 3 to 4 hours.
- Protecting main meant developers waited. Next, shape traffic and change the merge model.

### 12 · What we changed

07:05–07:35

- First, shape incoming submission traffic with admission gating.
- Then, make the platform own waiting and release through deferrals.
- Finally, relax conflict constraints to unlock more independent merge progress.

### 13 · Admission gating

07:35–08:30

- First, keep expensive submissions out during peak hours.
- Smaller changes enter; rejected authors must resubmit later.
- This protects active capacity, but puts scheduling onto developers.

### 14 · Admission gating: the tradeoff

08:30–09:20

- Business-hour average landing rate: about 130 → 160 lands/hour; +23% after gating.
- These are business-hour averages, not peak landing rates. The percentage uses unrounded values.
- The cost: developers must return and resubmit. Deferrals automate that waiting.

### 15 · Deferrals

09:20–10:20

- Accept the costly request once and hold it durably.
- Recheck release conditions and admit it when eligible.
- The platform owns waiting and release, not the developer.

### 16 · Deferrals: the tradeoff

10:20–11:05

- Submit once; release automatically.
- The waiting and validation cost still exist.
- When deferred work enters the active queue, its dependencies still constrain progress.

### 17 · Any target overlap could serialize work

11:05–12:20

- The original rule treats relevant target overlap as a yes-or-no conflict.
- One bridge can couple otherwise separate groups of changes.
- We needed to distinguish strong dependency signals from weak ones.

### 18 · Conflict relaxation

12:20–13:15

- Rank overlaps instead of treating every conflict equally.
- Retain stronger links and relax selected weaker links.
- The overlap score helps choose which groups can make independent progress.

### 19 · Jaccard scoring for dependency overlap

13:15–14:35

- Jaccard similarity is shared targets divided by distinct targets across both changes.
- Direct overlap has weight 1; new dependency-edge overlaps have weight 0.5 in each direction.
- Take the strongest signal. This ranks overlap; it is not a failure probability.

### 20 · Buy parallelism by dropping the weakest bridges

14:35–16:15

- Six changes at 50% desired parallelism gives a target of three independent groups.
- Retain the strong links: 0.90, 0.80, and 0.70. Drop the weak bridges: 0.08 and 0.04.
- This is a graph-level choice, not one universal score cutoff.

### 21 · Assign new arrivals. Leave in-flight work alone.

16:15–17:00

- Assign each new arrival using the conflict policy.
- Preserve placement for work already being validated.
- Avoid reshuffling useful speculative builds. This is not a fixed global merge order.

### 22 · Conflict relaxation: the tradeoff

17:00–17:55

- July business-hour average after relaxation: about 160 to 250 lands/hour, roughly 56% observed growth.
- More independent progress, but some combinations land without being validated together.
- Checks are not waived. We need confidence in the states that actually land.

### 23 · The cost: trunk can turn red

17:55–18:40

- Individually green changes can still fail together after landing.
- The combined state can leave trunk red even though both PRs passed.
- Stovepipe validates landed states and tracks which ones are known good.

### 24 · Stovepipe: one history, two moving refs

18:40–23:55

- A is known green. Main advances to D, but failed validation keeps verified/main at A.
- Main reaches G. Validate the A–G range while verified/main still points to A.
- When G passes, advance verified/main to G. Passing G does not certify every intermediate state.

### 25 · Recovering a broken trunk

23:55–25:15

- D fails validation. Stovepipe alerts on-call.
- On-call prepares a fix or revert, gets approvals, and lands it through SubmitQueue.
- Stovepipe validates the repaired state and advances verified/main only after validation passes.

### 26 · More throughput. Shorter waits.

25:15–26:55

- As of October 2026: ~300 successful lands/hour at peak versus January’s ~90, about 233% growth. Branch requests excluded.
- Land-time p95 is around 30 minutes outside incidents, not a guaranteed upper bound.
- The next step is making this approach reusable and available in the open.

### 27 · SubmitQueue and Stovepipe in the open

26:55–28:00

- Both SubmitQueue and Stovepipe are being built in the open.
- SubmitQueue handles pre-merge validation; Stovepipe handles post-merge validation.
- Explore the code through the project link or QR code.

### 28 · What comes next

28:00–29:10

- Batch submissions in SubmitQueue to reduce repeated validation and merge work.
- Move toward project-scoped confidence instead of one global status.
- Invest in automated diagnosis, fixes, and reverts. These are future directions.

### 29 · A hybrid approach to validation

29:10–29:40

- Keep required checks and stronger dependency constraints before merge.
- Continuously validate integrated trunk states after merge.
- The hybrid model supports higher landing throughput with reasonable land times.

### 30 · Thank you

29:40–30:00

- Thank you for watching.
- SubmitQueue and Stovepipe are being built in the open.
- The WeAreDevelopers and Seattle Systems recordings provide more background.

## Public resources

- [Preetam Dwivedi · LinkedIn](https://www.linkedin.com/in/behinddwalls/)
- [WeAreDevelopers · Merging at Scale](https://www.youtube.com/watch?v=mPssttsZknY&t=144s)
- [Seattle Systems · Merging at Scale](https://www.youtube.com/watch?v=O4sCvZAMC9w)
- [Open-source SubmitQueue](https://github.com/uber/submitqueue)
