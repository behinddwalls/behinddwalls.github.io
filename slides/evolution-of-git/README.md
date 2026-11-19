# Evolution of Git

**From Kernel to Codebase**

Speaker: Preetam Dwivedi. Event: Seattle Systems, "From Kernel to Codebase: Engineering Systems at Scale".

This is a self-contained HTML presentation for a 30–45 minute slot, planned at 24:25 (about 24 minutes). It has 27 slides, including a short divider that opens each part; 9 have step-by-step builds.

## Files

- `index.html`: slides, SVG diagrams, builds, short speaker cues, and presentation controls.
- `speaker-notes.md`: the run of show, speaking cues, and one cue per build step, synchronized with the HTML deck.
- `research.md`: the outline, the figures and how to read them, and sources.

Open `index.html` in a browser. Nothing needs to be installed, and the deck works offline. The LinkedIn QR code and Git logo are embedded. Applicable slides link to official [Git documentation](https://git-scm.com/doc) in their source footers; the Notes panel includes the full resource list.

The [Git logo](https://git-scm.com/community/logos) is © [Jason Long](https://bsky.app/profile/jasonlong.me), licensed under [Creative Commons Attribution 3.0 Unported (CC BY 3.0)](https://creativecommons.org/licenses/by/3.0/). The title slide includes this credit and links to the source and license.

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

The URL records the slide and build step, for example `#slide=12&step=5`.

## Builds

Diagram elements opt in to builds with step specs:

- `data-show`: the element is visible only at the listed steps.
- `data-hl`: the element is highlighted at the listed steps.
- `data-dim`: the element is dimmed at the listed steps.

Spec examples: `"2"` means step 2 only, `"2-"` means step 2 onwards, `"1-3"` means steps 1 to 3, and `"1,4"` means steps 1 and 4. To add a build, give a slide a `builds` array of `{label, cue}` and tag its elements. No per-slide render code is needed.

## Structure

1. **The data model.** What an object is, blobs, trees, commits, refs, reachability and retention, and where objects live: loose files, packfiles, and the pack index. The storage slide expands an entry to show its type, uncompressed size and zlib stream boundary. How a delta is stored (one tree stored whole vs. as a delta, before compression and pack-entry overhead). One A–E history runs through the pack, fetch, and push slides. Git 3.0 changes are labeled as planned defaults for new repositories.
2. **Fetch and push.** Both sides of the wire, with the client and server steps shown separately. Part 3 opens on the key split: a push is a packfile plus a ref update.
3. **Server evolution.** One box, RAID + DRBD pairs, then GitHub Spokes's application-level replication and 3PC. Cursor's object-store WAL diagram has six builds, separating best-effort gossip from concurrent fetches; a short spoken aside contrasts it with Rubicon's quorum service for refs and pack metadata.
4. **GitFarm.** Uber's cold-clone baseline, warm checkouts, architecture, request flow, and identity/freshness/isolation choices. “Where GitFarm fits” lists four concrete workflows without categories; detailed production measurements remain in `research.md`. The closing keeps client execution separate from the upstream hosting/storage layer.
