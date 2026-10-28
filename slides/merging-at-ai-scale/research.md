# Talk outline and public resources

## Story

The speaker introductions lead into a high-level outline of the talk.

1. Uber’s monorepo scale and the shared integration problem.
2. Two individually green PRs can produce a broken integrated build.
3. SubmitQueue validates speculative futures; target analysis, probabilistic speculation, and bypassing make that practical.
4. AI-assisted development increases the arrival rate. March’s growth exposes long landing waits.
   A brief overview introduces the response: shape the traffic, automate the wait, and unlock parallelism.
5. Admission gating protects capacity but makes developers schedule around peak hours.
6. Deferral automates waiting and release without removing dependency constraints.
7. Conflict relaxation creates independent progress by retaining stronger overlap signals and dropping selected weaker links.
8. Post-merge validation detects failures and establishes known-good states. On-call is alerted, prepares a fix/revert, gets approvals, and lands it through SubmitQueue. Stovepipe validates the repaired state before advancing verified/main.
9. Results and the open-source SubmitQueue (pre-merge) and Stovepipe (post-merge) systems.
10. Future investments: SubmitQueue batching, project-scoped confidence, and automated fixes/reverts.

## Interpreting the figures

- The monorepo figures describe organization-wide scale, not go-code-only throughput.
- January’s approximately 90 and March’s approximately 130 lands/hour are historical peak milestones. Those rounded milestones imply approximately 44% growth.
- During the early scale pressure, land-time p95 rose from approximately 30 minutes to 3–4 hours at peak.
- The admission-gating comparison is a business-hour average, shown as approximately 130 to 160 lands/hour and a 23% increase. The percentage uses unrounded values. This is a before/after observation, not an isolated causal estimate.
- No separate throughput improvement is attributed to deferral alone.
- The early throughput milestone after relaxation in July 2026 is approximately 240–250 lands/hour, displayed as ~250. This business-hour average is compared with the post-gating ~160/hour. The displayed ~56% observed increase uses the rounded milestones, not an isolated causal estimate of relaxation alone.
- As of October 2026, approximately 300 successful lands/hour is an overall peak result, not a sustained average. Compared with January’s approximately 90/hour peak, the rounded milestones imply approximately 233% growth. It is not part of a like-for-like staircase with the preceding business-hour averages.
- The approximately 30-minute land-time p95 describes operation outside incidents, not a guaranteed upper bound.

## Technical distinctions

- Dependency overlap is a ranking signal, not a calibrated failure probability.
- Relaxing dependency constraints is not permission to ignore failed required checks or textual conflicts.
- Post-merge validation can coalesce intermediate heads. Passing a later head does not certify every earlier state.
- `main` advances as changes land. `verified/main` advances after eligible passing validation, staying at the last known-good commit while validation is pending or failing.
- A known-good ref helps only consumers that choose to use it.
- Detection, confidence recording, and recovery are different responsibilities.
- Future directions and the operating approach discussed here are not a promise that every capability is already available in the open-source project.

## Public resources

- [Open-source SubmitQueue](https://github.com/uber/submitqueue)
- [WeAreDevelopers recording](https://www.youtube.com/watch?v=mPssttsZknY&t=144s)
- [Seattle Systems recording](https://www.youtube.com/watch?v=O4sCvZAMC9w)
- [Preetam Dwivedi on LinkedIn](https://www.linkedin.com/in/behinddwalls/)
