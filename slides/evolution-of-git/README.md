# Git at Scale

**From Kernel to Codebase**

Speaker: Preetam Dwivedi. Event: Seattle Systems, "From Kernel to Codebase: Engineering Systems at Scale".

This is a self-contained HTML presentation for a 30–45 minute slot, planned at about 32 minutes. It has 31 slides, 13 of which have step-by-step builds.

## Files

- `index.html`: slides, SVG diagrams, builds, short speaker cues, and presentation controls.
- `speaker-notes.md`: the run of show, three speaking cues per slide, and one cue per build step.
- `research.md`: the outline, the figures and how to read them, and sources.

Open `index.html` in a browser. Nothing needs to be installed, and the deck works offline. The LinkedIn QR code and the Git logo (by Jason Long, CC BY 3.0, from git-scm.com) are embedded.

## Controls

| Key or control | Action |
| --- | --- |
| Left / Right, Space, Page Up / Down | Previous / next build or slide |
| Home / End | First / last slide |
| N | Speaker cues, build cue, and planned timing |
| O | Slide overview |
| F | Fullscreen |
| B | Blackout |
| T / Shift+T | Start or pause / reset timer |
| ? | Keyboard help |
| Presenter ↗ | Synchronized second window with speaker cues |
| Browser Print | Slide-only PDF; build slides print in their final state with a one-line step summary |

The URL records the slide and build step, for example `#slide=9&step=5`.

## Builds

Diagram elements opt in to builds with step specs:

- `data-show`: the element is visible only at the listed steps.
- `data-hl`: the element is highlighted at the listed steps.
- `data-dim`: the element is dimmed at the listed steps.

Spec examples: `"2"` means step 2 only, `"2-"` means step 2 onwards, `"1-3"` means steps 1 to 3, and `"1,4"` means steps 1 and 4. To add a build, give a slide a `builds` array of `{label, cue}` and tag its elements. No per-slide render code is needed.

## Structure

1. **The data model.** What an object is, blobs, trees, commits, refs, reachability, and where objects live: loose files, packfiles, and the pack index. How a delta is stored (a tree delta byte by byte, and server.ts edited 100 times), then one skim slide of advanced topics. One A–E history runs through the pack, fetch, and push slides.
2. **Fetch and push.** Both sides of the wire, with the client and server steps shown separately. This part ends on the key split: a push is a packfile plus a ref transaction.
3. **Server evolution.** One box, then RAID + DRBD pairs, then Spokes, then packs in blob storage with refs in a database (JGit DFS, Azure DevOps, CodeCommit), then WAL-Git (Cursor Continuity) and what a push writes to the object store (walgit's format), with GitHub's Rubicon as the same pattern using a per-ref CAS service.
4. **GitFarm.** The problem, how remote devboxes keep a warm monorepo for developers and GitFarm does it for automation, the architecture, the request flow, stateful sessions, and the design choices. Then a map of use cases, with the measured results in the speaker notes.
