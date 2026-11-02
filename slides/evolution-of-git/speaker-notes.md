# Git at Scale — speaker cues

Planned at about 31 minutes for a 30–45 minute slot · 31 slides. Each slide has three short cues, and build slides also have one cue per step.

## Run of show

| Slide | Start | Length | Title |
| --- | --- | --- | --- |
| 01 | 00:00 | 00:20 | Git at Scale |
| 02 | 00:20 | 00:15 | About me |
| 03 | 00:35 | 00:50 | 2005: built for one very unusual project |
| 04 | 01:25 | 01:00 | Everything in Git is an object |
| 05 | 02:25 | 01:10 | Files are blobs, directories are trees |
| 06 | 03:35 | 01:20 | A commit is a snapshot that shares structure |
| 07 | 04:55 | 01:15 | Refs give names to commits |
| 08 | 06:10 | 01:30 | Where objects live: loose files and packs |
| 09 | 07:40 | 01:30 | How a delta is stored |
| 10 | 09:10 | 00:45 | Further down the rabbit hole |
| 11 | 09:55 | 00:40 | Two programs on each side of the wire |
| 12 | 10:35 | 02:00 | git fetch, end to end |
| 13 | 12:35 | 02:00 | git push, end to end |
| 14 | 14:35 | 00:45 | A push is two different things |
| 15 | 15:20 | 00:40 | Stage 0: one box |
| 16 | 16:00 | 01:00 | Why not just put the repos on NFS? |
| 17 | 17:00 | 01:05 | Stage 1: RAID + DRBD pairs |
| 18 | 18:05 | 01:25 | Stage 2: Spokes — replicate Git, not disks |
| 19 | 19:30 | 01:15 | Stage 2b: packs in blobs, refs in a database |
| 20 | 20:45 | 02:15 | Stage 3: WAL-Git — the log is the repo |
| 21 | 23:00 | 01:30 | What a push writes to the object store |
| 22 | 24:30 | 00:25 | Storage scales. Every client still clones. |
| 23 | 24:55 | 00:45 | The cost of a local checkout |
| 24 | 25:40 | 00:40 | Devboxes for developers, GitFarm for automation |
| 25 | 26:20 | 00:55 | GitFarm: Git as a service |
| 26 | 27:15 | 00:55 | One request, end to end |
| 27 | 28:10 | 00:55 | Stateful sessions: many commands, one repo state |
| 28 | 29:05 | 00:50 | Identity, freshness, isolation |
| 29 | 29:55 | 00:45 | Who needs Git without a clone? |
| 30 | 30:40 | 00:25 | Git became a distributed system |
| 31 | 31:05 | 00:10 | Thank you |

## Speaking cues

### 01 · Git at Scale

00:00–00:20

- Git was designed with no server. Today it is the center of a distributed system.
- Four parts: the data model, fetch and push on the wire, how servers evolved, and GitFarm.
- Goal: you leave knowing what actually happens underneath git push.

### 02 · About me

00:20–00:35

- One line: Developer Platform at LinkedIn; before that Developer Platform at Uber, where GitFarm comes from; before that Amazon.
- Keep it to ten seconds and move on.
- The QR code goes to LinkedIn for anyone who wants to connect.

### 03 · 2005: built for one very unusual project

00:35–01:25

- Linus Torvalds wrote Git for the kernel's decentralized workflow: many maintainers, patches and pulls between peers.
- The original design has no privileged copy. Every clone holds the entire history.
- Keep that in mind: everything later in this talk is about making one copy special again.

### 04 · Everything in Git is an object

01:25–02:25

- Everything Git stores is an object: type, size, and content, compressed and named by its SHA-1. Commit, tree, and blob are the chain that matters; tag is the fourth, rarer type. The stored bytes are a small header (type, a space, the size, a NUL byte) followed by the content; the SHA-1 of those bytes is the object's name.
- Objects never change. Editing a file creates a new blob; the old one is still there under its old name.
- On top sits a ref: not an object, just a file holding a SHA, and the only mutable thing in Git. That split drives the rest of the talk. (Try it: git cat-file -t ce0136 prints blob; git cat-file -t HEAD prints commit.)

### 05 · Files are blobs, directories are trees

02:25–03:35

- A blob is the file's bytes with a small header; its name is the SHA-1 of that. No filename, no metadata.
- A tree maps names and modes to blob or tree hashes. Identical content anywhere in history is stored exactly once.
- Because names are derived from content, any copy can verify any object. That is what makes replication safe later.

### 06 · A commit is a snapshot that shares structure

03:35–04:55 · 3 builds

- A commit is a root tree plus parent pointers plus metadata. Not a diff.
- A new commit only adds objects on the path from the root to what changed; everything else is shared.
- It is a Merkle DAG: trees hash their children's hashes, commits hash their tree and parents. Anyone can verify what they receive, and an identical subtree hash means you can skip the whole directory. Walking it still means following pointers one object at a time; remember that for the server section.

Builds:

1. **One commit, one snapshot.** A commit points to one root tree. Walk the tree and you have every file.
2. **Second commit shares structure.** Edit server.ts. Only the path from root to that file gets new objects; README and pack.ts are reused.
3. **Count the new objects.** One changed file means one blob, two trees, one commit: the new hash ripples up the path. Hashes of hashes make this a Merkle DAG, so one commit hash covers every file and all of history.

### 07 · Refs give names to commits

04:55–06:10 · 4 builds

- A ref is a 41-byte file holding a SHA. Branches move with each commit, tags stay put, remote-tracking refs remember the server as of the last fetch, and HEAD names the current branch. As they multiply, Git moves them from loose files to one packed-refs file, or to reftable.
- Updating a ref is lock, compare, rename: a tiny compare-and-swap. Every write to a repo ends with one, and it is the only mutable thing in Git.
- Reachability from refs defines what the repo contains. Unreachable objects are eventually garbage-collected.

Builds:

1. **Kinds of ref.** A ref is a tiny file holding a SHA. Branches under refs/heads move with each commit; tags under refs/tags stay put; refs/remotes/origin/* remember where the server was at the last fetch. HEAD is symbolic: it names a branch, not a commit.
2. **How refs are stored as they grow.** Git starts with one loose file per ref: simple, but slow at 100k files. gc packs them into packed-refs, one sorted text file: fast lookups, but changing one ref rewrites the whole file. reftable (Git 2.45+) stores refs in sorted, prefix-compressed binary blocks; an update appends a small table and compaction merges them later.
3. **Commit E: main moves.** Committing writes the new objects first, then updates one ref: lock main, check it still points at D, write E, rename the lock into place. That compare-and-swap is the whole update. HEAD follows because it names main. Every push ends with the same step on the server.
4. **Reachability defines the repo.** X was committed on a branch that has since been deleted. No ref reaches it, so it is no longer part of the repo and gc eventually prunes it. Everything a ref can reach is kept: every ref is a garbage-collection root.

### 08 · Where objects live: loose files and packs

06:10–07:40 · 4 builds

- New objects are written loose, one file each. gc, repack, or a large fetch or push rolls many objects into one packfile plus its .idx.
- Inside a pack, older versions are deltas against newer ones, placed for size rather than by commit.
- To read any object, Git checks loose files first, then binary-searches each .idx for the offset. Remember A–E: the fetch slide uses the same history.

Builds:

1. **Every commit starts loose.** git commit writes each new object (commit, trees, blob) as its own zlib file under .git/objects, named by its hash.
2. **gc rolls them into a pack.** Auto gc runs after commit or fetch once there are about 6,700 loose objects or 50 packs (or on a schedule with git maintenance). A fetch of 100+ objects is kept as a pack; a big push lands as one on the server. The id in the name is a checksum of the contents. D and E came later, so they are still loose.
3. **Inside the pack.** Every object is zlib-compressed. Blobs and trees can be deltas against a similar nearby object, usually newest whole and older as deltas. Chains are capped at 50. Entries are placed for size, not by commit.
4. **The .idx finds things.** The .idx is sorted by hash: a 256-entry fanout jumps to rows sharing the first byte, binary search finds the row, and the row gives the offset (plus a CRC to check the stored bytes). The .pack itself stays in write order.

### 09 · How a delta is stored

07:40–09:10 · 2 builds

- A delta is a list of COPY and INSERT instructions against a base object. A tree is just a list of mode, name, hash entries, so one changed file changes 20 bytes of each tree on its path, and the older tree becomes a ~26-byte delta against the newer one.
- server.ts edited 100 times is 100 objects, all kept. Packed, that's one or two whole copies plus about 200-byte deltas; depth is capped at 50, so Git branches or restarts chains as needed.
- Deltas are made when packing, reused when sending, and resolved whenever someone reads an old version.

Builds:

1. **A tree delta.** src/ is a tree: two entries (pack.ts, server.ts) of mode, name, and file hash, 72 bytes. Editing server.ts in commit E changes only its hash. Matching the newest-whole rule, E's src/ is stored whole and D's src/ becomes: copy bytes 0–51, insert the old 20-byte hash. About 26 bytes instead of 72. A commit is mostly new hashes and timestamps, so it usually stays whole.
2. **server.ts × 100 edits.** 100 versions, all kept. Loose they are 100 full copies; packed they are a whole base plus small deltas that branch, or restart from another whole copy, so no rebuild needs more than 50 hops. Recent versions read fast; old ones cost a few hops.

### 10 · Further down the rabbit hole

09:10–09:55

- A quick skim, not a deep dive: these are the indexes and storage tricks big repos and big hosts depend on.
- commit-graph, bitmaps, and the multi-pack-index speed up walks and lookups; geometric repacking and cruft packs keep maintenance cheap; alternates, partial clones, reftable, and SHA-256 deal with scale and security.
- None of them changes the model from the last few slides. Happy to go deeper in Q&A.

### 11 · Two programs on each side of the wire

09:55–10:35

- Fetch is fetch-pack talking to upload-pack. Push is send-pack talking to receive-pack.
- Transport is ssh or smart HTTP; framing is pkt-line; protocol v2 adds explicit commands like ls-refs and fetch.
- The server is the same Git binary running against a bare repo. That is why hosting Git looks easy and isn't.

### 12 · git fetch, end to end

10:35–12:35 · 5 builds

- Same history as the pack slide: the server has A–E (A–C packed, D and E loose); the client stopped at B.
- want names ref tips; have walks back through the client's history until the server recognizes a commit. The server then walks from E back to B, finding objects loose or through the .idx.
- It ships C, D, E as one fresh pack; the client re-indexes it into its own pack, and only then moves origin/main. At scale, the first step is the expensive one: protocol v0 advertises every ref before any work (a million refs is tens of MB), so v2's ls-refs lets the client ask only for the prefixes it needs.

Builds:

1. **Ref advertisement.** The client asks which refs exist. The server answers from its ref files: main points at E. No objects are touched yet.
2. **want / have.** want = the tip of each ref the client is missing (E). have = the client's own tips first (B), then older commits in batches only if needed. Once the server ACKs B, it knows the client has everything behind B, including A.
3. **Server walks the graph.** Walk back from E until you hit what the client has. E and D are found loose; C is found through the pack's .idx; B is common, so stop.
4. **Server builds one pack.** The sender builds; the receiver resolves. pack-objects puts C, D, E and their new trees and blobs in one pack. C's bytes are reused from the existing pack (CRC-checked, never re-inflated); D and E are compressed now. Deltas against B's objects are allowed: a thin pack.
5. **Client resolves and moves the ref.** Whoever receives a pack resolves it. Pack entries carry no hashes, so index-pack rebuilds every delta from its base, SHA-1s the result, and writes the .idx. Thin bases come from the client's own store (B) and are appended. Only after the connectivity check does origin/main move to E.

### 13 · git push, end to end

12:35–14:35 · 6 builds

- Continuing the A–E story: the client commits F on top of E and pushes. It sends only what the server lacks, plus the command E F refs/heads/main.
- The server quarantines, verifies connectivity, runs hooks, then does a ref transaction: lock, check old, write.
- That ref transaction is a compare-and-swap. Keep it in mind for every architecture that follows.

Builds:

1. **Server advertises refs.** receive-pack opens by telling the client where every ref points right now.
2. **Client builds the pack.** The client committed F on top of E. On push the client is the sender, so it builds: only objects reachable from F but not from E (F, its new trees, the changed blob), possibly as thin deltas against E's objects.
3. **Commands + pack.** Each command is old, new, ref: here E F refs/heads/main. The pack follows on the same stream.
4. **Server resolves in quarantine.** Now the server is the receiver, so it resolves: index-pack rebuilds every delta, SHA-1s each object, fills thin bases from E's objects, and writes the .idx, all inside a quarantine directory. Then the connectivity check on F. If anything fails, the quarantine is deleted and nothing becomes visible.
5. **Ref transaction.** pre-receive can reject. Then lock main, check it still points at E, write F, commit. If someone else pushed G first, main is no longer E and this push is rejected. This is compare-and-swap.
6. **Report status.** post-receive fires (webhooks, CI), and the client gets per-ref ok or ng.

### 14 · A push is two different things

14:35–15:20

- The pack is bulk data that can be copied anywhere without coordination.
- The ref update is a few bytes that must be agreed on, atomically.
- Use this lens for every stage: one box, RAID pairs, Spokes, blob plus database, WAL.

### 15 · Stage 0: one box

15:20–16:00

- The simplest Git server is one machine: sshd or git-daemon runs upload-pack and receive-pack against bare repos on local disk.
- Packs and refs both live on that disk, and the box agrees with itself.
- It is fast and simple until the disk, the CPU, or the machine runs out.

### 16 · Why not just put the repos on NFS?

16:00–17:00

- The obvious next step after one box: keep stock Git, but put every repo on a shared NFS filer that all servers mount. Any server can serve any repo.
- Git assumes a local disk. Delta walks become many small network reads, and ref updates depend on lock files and atomic rename over NFS. Atlassian's own docs call NFS fine for streaming packs but punishing for refs and loose objects; GitHub tried it early, and Bitbucket Data Center ran on it for years before Bitbucket Mesh. I ran Phabricator's repositories on NFS at Uber, so this is first-hand: say what broke first.
- Conclusion: Git needs local disk, or you rewrite Git around remote storage. "Local disk" from here on means a fast local SSD (NVMe): microseconds per random read instead of a network round trip.

### 17 · Stage 1: RAID + DRBD pairs

17:00–18:05 · 3 builds

- RAID-1 (Redundant Array of Independent Disks) mirrors two disks inside one server; DRBD (Distributed Replicated Block Device) mirrors every disk write over the network to a second server. GitHub's pre-2016 design: an active fileserver with RAID, mirrored block-by-block to a dedicated hot spare with DRBD.
- Packs and refs both live on the active disk; DRBD copies bytes without knowing they are Git.
- Failover was manual, the spare served nothing, and a pair outage took every repo on it offline.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/)

Builds:

1. **Active + hot spare.** RAID inside each server, DRBD mirrors every block to a dedicated spare. Four copies of every byte.
2. **Failover.** Active dies. A human confirms, promotes the spare, traffic moves. Now the spare has no spare.
3. **The limits.** Half the hardware idles, one server serves every read for its repos, and losing the pair takes them all offline.

### 18 · Stage 2: Spokes — replicate Git, not disks

18:05–19:30 · 3 builds

- DGit, later Spokes: real Git repos on several servers (3 by default, up to around 20 for the hottest repos), replicated by Git-aware software, not by blocks.
- Packs fan out freely; only the ref transaction goes through three-phase commit. Reads go to any replica.
- Costs: latency bound by the slowest replica; a floor of 3 copies even for idle repos; a ceiling, because every extra replica slows every push, so even ~20 is tight for monorepo CI; and repos as pets. Atlassian landed in the same place later: Bitbucket Mesh (Data Center 8.0, 2022) replaced NFS with repos replicated on local disk across nodes.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/) · [GitHub · Stretching Spokes (2017)](https://github.blog/engineering/infrastructure/stretching-spokes/) · [Atlassian · Bitbucket Mesh (replaces NFS for Git storage)](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)

Builds:

1. **Fan out the pack.** Packs are idempotent, so send them to every replica in parallel: 3 by default, up to around 20 for the hottest repos. No agreement needed yet.
2. **3PC on the ref transaction.** Only the tiny ref update needs agreement. Phase 1 asks every replica: can you apply these ref updates? Each runs Git's own ref transaction: lock the refs, check the old values. Phase 2: a majority said yes, so get ready. Phase 3: commit, or roll back everywhere. Spokes then compares a checksum of every replica's refs; one that disagrees is marked unhealthy and repaired. About four round trips per push.
3. **Reads, and the bill.** Any replica can serve a fetch. The price: tail latency, a fixed floor and ceiling on replicas, and repos you have to babysit.

### 19 · Stage 2b: packs in blobs, refs in a database

19:30–20:45 · 3 builds

- The other road: put immutable packs in blob storage and refs in a database. JGit DFS, Azure DevOps, and CodeCommit all follow this pattern, and so does Jigit at Uber: built on JGit, with MySQL and a blob store.
- Compute becomes stateless and storage durable, with no replica babysitting.
- The price: you no longer run stock Git on a local repo, so you own a Git implementation, a cache, and a database.

Sources: [JGit · DFS storage layer](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/internal/storage/dfs) · [AWS CodeCommit features (S3 + DynamoDB storage)](https://aws.amazon.com/codecommit/features/) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

Builds:

1. **Pack → blob store.** Packs are immutable files. Any stateless frontend can write one to blob storage.
2. **Ref → database transaction.** The ref update becomes a conditional write in a database: old value must match.
3. **Reads through a cache.** Git reads packs in small random pieces: the .idx, then each object and its delta bases. A blob read is 10–50 ms versus about 0.1 ms on a local SSD, so each frontend keeps recently used pack and index blocks in memory (JGit's DfsBlockCache, 64 KB blocks by default). Packs never change, so cached blocks never go stale; only misses go to the blob store.

### 20 · Stage 3: WAL-Git — the log is the repo

20:45–23:00 · 6 builds

- Cursor's Continuity: a push is a WAL entry in the object store; it becomes visible only when a compare-and-swap on the WAL index succeeds. Pushes are linearizable.
- The local Git repo is a warm cache. Replicas learn from lossy gossip but verify every read with a conditional GET, so reads are consistent.
- There is no primary: any node can take a push (the CAS on the object store decides) and any node can serve a read, rebuilding from the WAL if it has no copy. Rendezvous hashing just picks a preferred node to avoid CAS retries, and that node does compaction; idle repos are dropped and rebuilt on demand. Kept from Spokes: real Git on local disk. Kept from blob + DB: object storage as truth. Dropped: quorum, primary, routing DB, database. Cursor reports about 120 pushes/s on S3 Standard and 300+ on S3 Express One Zone, with reads scaling linearly to 100 replicas.

Sources: [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale) · [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit)

Builds:

1. **Read the index first.** Before advertising refs to the pusher, the node does the same conditional GET as any read. Now it knows the log is at version e1 with head 41, so this push will be entry #42. The version is the object's ETag on S3 or its generation on GCS.
2. **Persist the push as WAL entry #42.** Write the pack to local disk and upload entry #42 to the object store at the same time. Durable, but nothing points at it yet.
3. **Publish with compare-and-swap.** Lock main and check it is still E, then PUT the index with If-Match: e1, the version read in step 1. Success gives e2: the push is acknowledged and linearized.
4. **Gossip to other nodes.** UDP gossip tells other nodes there is news. It can be lost; correctness doesn't depend on it.
5. **Any node can serve a read.** A fetch starts with the same GET index as a push. A warm node sends If-None-Match with the version it has: 304 in under 10 ms means serve now; 200 means pull the new WAL entries first. Cold node 3 has no version yet, so it does a plain GET, learns e2 and head 42, rebuilds the repo from the WAL, then serves.
6. **Any node can take a push.** There is no primary role. Node 2 read e2 and tries to publish #43 with If-Match: e2. If another node got there first, it gets a 412: re-read the index, re-check the ref, retry. Pushes usually go to the rendezvous #1 node only because that avoids retries.

### 21 · What a push writes to the object store

23:00–24:30 · 3 builds

- The contract, as implemented by walgit (an open-source take on Cursor's design): per repository, one tiny mutable WAL index and everything else immutable: log entries, packs, checkpoints. walgit calls the index manifest.pb; Cursor calls it gitwal.pb.
- A push writes a pack, its .idx, and a PUSH log entry holding the pack reference and the ref transaction (main: E to F). Only the conditional PUT on the manifest makes it visible; git hears ok after that.
- After the push: the lease holder compacts packs and publishes a COMPACT entry; a checkpoint snapshots all refs and packs so a cold node starts from it plus the log tail. (Races were covered on the previous slide.)

Sources: [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

Builds:

1. **The contract.** Five kinds of objects per repository. Only the WAL index is mutable, and only by compare-and-swap. Entries, packs, and checkpoints are written once and never change.
2. **What one push writes.** Resolve and verify the pack locally (index-pack --fix-thin, connectivity, policy). Write pack P42 and entry #42 (kind PUSH, pack P42, refs main: E to F, plus metadata: writer node, client, pusher); they are written in parallel and are durable but invisible. Then CAS the WAL index: only if it is still at version e1, set head 42 and add P42. That is the commit point; git hears ok only after it.
3. **Compaction and checkpoints.** Later, the node holding the compaction lease merges P1, P2 and P42 into P-all, writes entry #43 COMPACT, and CASes the index to e3; other nodes download P-all instead of repacking. A checkpoint then snapshots all refs and the pack set at #43, so a cold node loads it and replays only newer entries.

### 22 · Storage scales. Every client still clones.

24:30–24:55

- Everything up to WAL-Git makes the server side scale.
- But each automation still pulls a full checkout before it can run one git command.
- GitFarm moves the execution to where the repo already is.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 23 · The cost of a local checkout

24:55–25:40

- Cloning a large monorepo takes about 15 minutes and a meaningful slice of a machine: 40+ GB of disk, plus cores and memory.
- Each automation system kept and synced its own checkouts, multiplying upstream load.
- CI caches reduce clone time but add cold starts, drift, and upkeep.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 24 · Devboxes for developers, GitFarm for automation

25:40–26:20

- A devbox (or devpod) is a remote dev machine that already has the monorepo checked out and kept in sync in the background. A developer connects with their IDE or shell and starts working: no 15-minute clone.
- GitFarm does the same for automation. Each node keeps a bare clone fresh from push events plus a periodic fetch, and a pool of warm checkouts.
- An agent, bot or service borrows one checkout per request, runs its git commands, and the checkout goes back to the pool.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 25 · GitFarm: Git as a service

26:20–27:15

- Clients call a gRPC API. The gateway authenticates, authorizes per repo, and places the request.
- Every node runs three containers: backend serves the request; sandbox keeps a pool of sandboxes and executes the commands; repo-manager keeps the bare clone, synced by push events plus a 5-minute fetch, and a pool of pre-warmed checkouts.
- Separate clusters per heavy use case, plus a shared cluster for everyone else.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 26 · One request, end to end

27:15–28:10 · 5 builds

- Request, authenticate, place on the node with the most free checkouts, or throttle.
- The backend takes a sandbox from the sandbox container's pool and a checkout from repo-manager's pool, mounts one into the other, and runs the commands as the caller.
- Pools are the whole trick: cold start moves off the request path.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **Request.** Client sends repo id, workspace type, and a list of commands.
2. **Authenticate + authorize.** Gateway identifies the caller and checks they may access this repo.
3. **Place.** Gateway picks the node with the most free checkouts for that repo, marks one occupied, and hands the request to that node's backend. None free → throttle.
4. **Acquire sandbox + checkout.** The backend first asks repo-manager for a pre-warmed checkout from its pool, then mounts it on a pre-started sandbox.
5. **Execute as caller.** Commands run inside the sandbox with the caller's identity, not the service's. Fetches and pushes go from the sandbox straight to upstream; repo-manager also fetches from upstream to keep its bare clone fresh. When the commands finish, the backend releases the sandbox and the checkout back to their pools.

### 27 · Stateful sessions: many commands, one repo state

28:10–29:05 · 3 builds

- The API takes a repo, a workspace type, and a list of commands; a session can chain commands that depend on earlier output.
- Guarantees: isolation, one consistent repo state, ordered execution. Fatal errors end the session.
- The example from the paper: merge-base, rev-parse, push.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **merge-base.** First command: where did this branch diverge from main?
2. **rev-parse.** Same session, same repo state: resolve the tip. Output streams back; the next step can use it.
3. **push.** Push from the sandbox. It goes through exactly the receive-pack path we walked earlier: quarantine, connectivity, ref compare-and-swap.

### 28 · Identity, freshness, isolation

29:05–29:55

- Identity is decoupled from the client environment: the caller's identity flows into the sandbox.
- Freshness is eventual with a 5-minute backstop; clients that need the newest state fetch first.
- Workload isolation by cluster avoids noisy neighbours.

### 29 · Who needs Git without a clone?

29:55–30:40

- Two axes: does the workload write, and is someone waiting on it. Agents and review automation fill every quadrant, which is why a shared execution layer pays off.
- Filled dots are measured production cases in the paper; hollow dots are patterns. Security scanning (secrets, vulnerable dependencies) is a read-only batch job: it needs the code at a commit, not a clone.
- Measured in production: compliance audits went from 110–160 s p50 on Buildkite to 20–30 s, more than 80% lower, and the code ownership service dropped from about 90 to 16 cores and 472 to 32 GB of memory. Agents open a session in under a second instead of a 15-minute, 40 GB clone.

### 30 · Git became a distributed system

30:40–31:05

- Storage, agreement, and execution each moved from a single machine to a distributed service.
- The thread the whole way: big immutable packs, tiny atomic ref updates.
- Agents are driving the next round, at both the storage and the execution layer.

### 31 · Thank you

31:05–31:15

- Thank you.
- The GitFarm paper, the Cursor post, and walgit are the deep references.
- Git's own docs on pack format and protocol cover everything from Part 1 and 2.
