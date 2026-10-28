# Merging at AI Scale

**From Green to Broken Trunk**

Speakers: Preetam Dwivedi and Manjari Akella.

A self-contained HTML presentation for a 30-minute prerecorded talk, with 30 slides and a Thank you ending.

## Files

- `index.html`: slides, diagrams, chart, short speaker cues, and presentation controls.
- `speaker-notes.md`: three concise speaking cues per slide, with planned timing.
- `research.md`: talk outline, metric interpretation, and public resources.

Open `index.html` in a browser. No installation, CDN, or network connection is needed to present the deck. Public links open separately.

## Controls

| Key or control | Action |
| --- | --- |
| Left / Right, Space, Page Up / Down | Previous / next build or slide |
| Home / End | First / last slide |
| N | Speaker cues and planned timing |
| O | Slide overview |
| F | Fullscreen |
| B | Blackout |
| T / Shift+T | Start or pause / reset timer |
| ? | Keyboard help |
| Presenter ↗ | Synchronized second window with speaker cues |
| Browser Print | Slide-only PDF |

Slide position is stored in the URL hash, for example `#slide=15`. The Stovepipe example has four manual builds, advanced with the same controls. Its URL also records the build, for example `#slide=24&step=3`. Presenter navigation and build state synchronize from either window. Printing includes a static summary of all four builds.

## Content

The speaker introduction presents both speakers with matching portraits, names, and roles, followed by a high-level outline. A short intervention overview bridges developer impact and the detailed mechanisms. The story moves from monorepo integration and speculative validation to AI-driven workload growth, admission gating, deferral, conflict relaxation, post-merge confidence, and results.

The notes are prompts for the speaker, not a full narration script. The tactical interventions are paced more briskly so post-merge validation begins earlier. One Git-style commit graph introduces Stovepipe, its validation baseline, and the main and verified/main refs. Fixed commit positions and four manual builds show how landing, failed validation, pending validation, and passing validation affect the two pointers. The July throughput milestone is shown after relaxation; the final peak result remains in the results section. Numerical distinctions, such as averages versus peaks, are retained where they matter. See `research.md` for a concise outline and interpretation guide.

Both speaker portraits and all three QR codes are embedded: one for the open-source project and two for the recordings. The closing slide links the WeAreDevelopers and Seattle Systems recordings alongside the open-source project.

## Public resources

- [SubmitQueue](https://github.com/uber/submitqueue)
- [WeAreDevelopers](https://www.youtube.com/watch?v=mPssttsZknY&t=144s)
- [Seattle Systems](https://www.youtube.com/watch?v=O4sCvZAMC9w)
- [LinkedIn](https://www.linkedin.com/in/behinddwalls/)
