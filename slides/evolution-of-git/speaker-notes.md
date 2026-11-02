# Evolution of Git — speaker cues

Planned at about 37 minutes for a 30–45 minute slot · 36 slides. Each slide has three short cues, and build slides also have one cue per step.

## Run of show

| Slide | Start | Length | Title |
| --- | --- | --- | --- |
| 01 | 00:00 | 00:20 | Evolution of Git |
| 02 | 00:20 | 00:50 | 2005: built for one very unusual project |
| 03 | 01:10 | 01:00 | Everything in Git is an object |
| 04 | 02:10 | 01:10 | Content is its own address |
| 05 | 03:20 | 01:20 | A commit is a snapshot that shares structure |
| 06 | 04:40 | 01:00 | Refs give names to commits |
| 07 | 05:40 | 01:30 | Where objects live: loose files and packs |
| 08 | 07:10 | 01:30 | How a delta is stored |
| 09 | 08:40 | 00:45 | Further down the rabbit hole |
| 10 | 09:25 | 00:40 | Two programs on each side of the wire |
| 11 | 10:05 | 02:00 | git fetch, end to end |
| 12 | 12:05 | 02:00 | git push, end to end |
| 13 | 14:05 | 01:20 | Branches and tags at scale |
| 14 | 15:25 | 00:45 | A push is two different things |
| 15 | 16:10 | 00:40 | Stage 0: one box |
| 16 | 16:50 | 01:00 | Why not just distribute the storage? |
| 17 | 17:50 | 01:05 | Stage 1: RAID + DRBD pairs |
| 18 | 18:55 | 01:25 | Stage 2: Spokes — replicate Git, not disks |
| 19 | 20:20 | 01:15 | Stage 2b: packs in blobs, refs in a database |
| 20 | 21:35 | 02:15 | Stage 3: WAL-Git — the log is the repo |
| 21 | 23:50 | 01:30 | What a push writes to the object store |
| 22 | 25:20 | 00:55 | Five answers to one question |
| 23 | 26:15 | 00:25 | Storage scales. Every client still clones. |
| 24 | 26:40 | 00:45 | The cost of a local checkout at Uber |
| 25 | 27:25 | 00:55 | GitFarm: Git as a service |
| 26 | 28:20 | 00:55 | One request, six steps |
| 27 | 29:15 | 00:55 | Stateful sessions: many commands, one repo state |
| 28 | 30:10 | 00:50 | Identity, freshness, isolation |
| 29 | 31:00 | 00:45 | Who needs Git without a clone? |
| 30 | 31:45 | 01:25 | Agents need the monorepo, not a clone |
| 31 | 33:10 | 01:00 | Code review and PR automation |
| 32 | 34:10 | 00:40 | Compliance: audit every bypass |
| 33 | 34:50 | 00:40 | Results: give the resources back |
| 34 | 35:30 | 00:35 | What's next |
| 35 | 36:05 | 00:25 | Git became a distributed system |
| 36 | 36:30 | 00:10 | Thank you |

## Speaking cues

### 01 · Evolution of Git

00:00–00:20

- Git was designed with no server. Today it is the center of a distributed system.
- Four parts: the data model, fetch and push on the wire, how servers evolved, and GitFarm.
- Goal: you leave knowing what actually happens underneath git push.

### 02 · 2005: built for one very unusual project

00:20–01:10

- Git was written for the kernel's decentralized workflow: many maintainers, patches and pulls between peers.
- The original design has no privileged copy. Every clone holds the entire history.
- Keep that in mind: everything later in this talk is about making one copy special again.

### 03 · Everything in Git is an object

01:10–02:10

- Everything Git stores is an object: type, size, and content, compressed and named by its SHA-1. Commit, tree, and blob are the chain that matters; tag is the fourth, rarer type.
- Objects never change. Editing a file creates a new blob; the old one is still there under its old name.
- On top sits a ref: not an object, just a file holding a SHA, and the only mutable thing in Git. That split drives the rest of the talk.

### 04 · Content is its own address

02:10–03:20

- A blob is the file's bytes with a small header; its name is the SHA-1 of that. No filename, no metadata.
- A tree maps names and modes to blob or tree hashes. Identical content anywhere in history is stored exactly once.
- Because names are derived from content, any copy can verify any object. That is what makes replication safe later.

### 05 · A commit is a snapshot that shares structure

03:20–04:40 · 3 builds

- A commit is a root tree plus parent pointers plus metadata. Not a diff.
- A new commit only adds objects on the path from the root to what changed; everything else is shared.
- It is a Merkle DAG: trees hash their children's hashes, commits hash their tree and parents. Anyone can verify what they receive, and an identical subtree hash means you can skip the whole directory. Walking it still means following pointers one object at a time; remember that for the server section.

Builds:

1. **One commit, one snapshot.** A commit points to one root tree. Walk the tree and you have every file.
2. **Second commit shares structure.** Edit server.ts. Only the path from root to that file gets new objects; README and pack.ts are reused.
3. **Count the new objects.** One changed file means one blob, two trees, one commit: the new hash ripples up the path. Hashes of hashes make this a Merkle DAG, so one commit hash covers every file and all of history.

### 06 · Refs give names to commits

04:40–05:40 · 3 builds

- A branch is a 41-byte file holding a SHA. HEAD names the current branch.
- Every write to a repo ends with one small ref update. That update is the only mutable thing in Git.
- Reachability from refs defines what the repo contains. Unreachable objects are eventually garbage-collected.

Builds:

1. **main points at D.** A branch is a file holding one SHA. HEAD is a symbolic ref naming the branch.
2. **Commit E: main moves.** Committing writes objects first, then rewrites one 41-byte file: main goes from D (9d2a) to E (c8f3). That rewrite is the whole 'update'.
3. **Reachability defines the repo.** X was committed on a branch that has since been deleted. No ref reaches it, so it is no longer part of the repo and gc eventually prunes it. A–E stay, because main reaches them.

### 07 · Where objects live: loose files and packs

05:40–07:10 · 4 builds

- New objects are written loose, one file each. gc, repack, or a large fetch or push rolls many objects into one packfile plus its .idx.
- Inside a pack, older versions are deltas against newer ones, placed for size rather than by commit.
- To read any object, Git checks loose files first, then binary-searches each .idx for the offset. Remember A–E: the fetch slide uses the same history.

Builds:

1. **Every commit starts loose.** git commit writes each new object (commit, trees, blob) as its own zlib file under .git/objects, named by its hash.
2. **gc rolls them into a pack.** Auto gc runs after commit or fetch once there are about 6,700 loose objects or 50 packs (or on a schedule with git maintenance). A fetch of 100+ objects is kept as a pack; a big push lands as one on the server. The id in the name is a checksum of the contents. D and E came later, so they are still loose.
3. **Inside the pack.** Every object is zlib-compressed. Blobs and trees can be deltas against a similar nearby object, usually newest whole and older as deltas. Chains are capped at 50. Entries are placed for size, not by commit.
4. **The .idx finds things.** The .idx is sorted by hash: a 256-entry fanout jumps to rows sharing the first byte, binary search finds the row, and the row gives the offset (plus a CRC to check the stored bytes). The .pack itself stays in write order.

### 08 · How a delta is stored

07:10–08:40 · 2 builds

- A delta is a list of COPY and INSERT instructions against a base object. A tree is just a list of mode, name, hash entries, so one changed file changes 20 bytes of each tree on its path, and the older tree becomes a ~26-byte delta against the newer one.
- server.ts edited 100 times is 100 objects, all kept. Packed, that's one or two whole copies plus about 200-byte deltas; depth is capped at 50, so Git branches or restarts chains as needed.
- Deltas are made when packing, reused when sending, and resolved whenever someone reads an old version.

Builds:

1. **A tree delta.** src/ is a tree: two entries (pack.ts, server.ts) of mode, name, and file hash, 72 bytes. Editing server.ts in commit E changes only its hash. Matching the newest-whole rule, E's src/ is stored whole and D's src/ becomes: copy bytes 0–51, insert the old 20-byte hash. About 26 bytes instead of 72. A commit is mostly new hashes and timestamps, so it usually stays whole.
2. **server.ts × 100 edits.** 100 versions, all kept. Loose they are 100 full copies; packed they are a whole base plus small deltas that branch, or restart from another whole copy, so no rebuild needs more than 50 hops. Recent versions read fast; old ones cost a few hops.

### 09 · Further down the rabbit hole

08:40–09:25

- A quick skim, not a deep dive: these are the indexes and storage tricks big repos and big hosts depend on.
- commit-graph, bitmaps, and the multi-pack-index speed up walks and lookups; geometric repacking and cruft packs keep maintenance cheap; alternates, partial clones, reftable, and SHA-256 deal with scale and security.
- None of them changes the model from the last few slides. Happy to go deeper in Q&A.

### 10 · Two programs on each side of the wire

09:25–10:05

- Fetch is fetch-pack talking to upload-pack. Push is send-pack talking to receive-pack.
- Transport is ssh or smart HTTP; framing is pkt-line; protocol v2 adds explicit commands like ls-refs and fetch.
- The server is the same Git binary running against a bare repo. That is why hosting Git looks easy and isn't.

### 11 · git fetch, end to end

10:05–12:05 · 5 builds

- Same history as the pack slide: the server has A–E (A–C packed, D and E loose); the client stopped at B.
- want names ref tips; have walks back through the client's history until the server recognizes a commit. The server then walks from E back to B, finding objects loose or through the .idx.
- It ships C, D, E as one fresh pack; the client re-indexes it into its own pack, and only then moves origin/main.

Builds:

1. **Ref advertisement.** The client asks which refs exist. The server answers from its ref files: main points at E. No objects are touched yet.
2. **want / have.** want = the tip of each ref the client is missing (E). have = the client's own tips first (B), then older commits in batches only if needed. Once the server ACKs B, it knows the client has everything behind B, including A.
3. **Server walks the graph.** Walk back from E until you hit what the client has. E and D are found loose; C is found through the pack's .idx; B is common, so stop.
4. **Server builds one pack.** The sender builds; the receiver resolves. pack-objects puts C, D, E and their new trees and blobs in one pack. C's bytes are reused from the existing pack (CRC-checked, never re-inflated); D and E are compressed now. Deltas against B's objects are allowed: a thin pack.
5. **Client resolves and moves the ref.** Whoever receives a pack resolves it. Pack entries carry no hashes, so index-pack rebuilds every delta from its base, SHA-1s the result, and writes the .idx. Thin bases come from the client's own store (B) and are appended. Only after the connectivity check does origin/main move to E.

### 12 · git push, end to end

12:05–14:05 · 6 builds

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

### 13 · Branches and tags at scale

14:05–15:25 · 3 builds

- Branches and tags are just refs: names pointing at SHAs. Branches move, lightweight tags don't, annotated tags point through a tag object. Real repos have thousands of them, hosts far more.
- Ref storage grows up in three steps: one loose file per ref, then one sorted packed-refs file, then reftable's sorted binary blocks, where an update is a small append instead of a full rewrite.
- On the wire, protocol v0 advertises every ref up front; v2's ls-refs with ref-prefix fixes that, and hosts hide internal refs. Every ref is also a GC root.

Builds:

1. **Branches and tags are just refs.** A branch is a ref that moves as you commit. A lightweight tag is a ref that never moves. An annotated tag is a ref to a tag object (tagger, message, signature) that points at a commit. Real repos have thousands: CI branches, agent branches, release tags, and on hosts, a ref per pull request.
2. **How refs are stored as they grow.** One loose file per ref is simple but slow at 100k files. packed-refs is one sorted text file: fast lookups, but changing one ref rewrites the whole file. reftable stores refs in sorted, prefix-compressed binary blocks; an update appends a small table and compaction merges them later.
3. **What thousands of refs cost.** Protocol v0 advertises every ref before any work, so a million refs means tens of MB per fetch. v2's ls-refs asks only for the prefixes you need. Hosts hide internal refs like refs/pull/*. And every ref is a garbage-collection root, so more refs means more to walk.

### 14 · A push is two different things

15:25–16:10

- The pack is bulk data that can be copied anywhere without coordination.
- The ref update is a few bytes that must be agreed on, atomically.
- Use this lens for every stage: one box, RAID pairs, Spokes, blob plus database, WAL.

### 15 · Stage 0: one box

16:10–16:50

- The first Git hosts were one big machine: web app, Git daemons, and repos on local disk.
- Packs and refs both live on that disk, and the box agrees with itself.
- It is fast and simple until the disk, the CPU, or the machine runs out.

Sources: [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

### 16 · Why not just distribute the storage?

16:50–17:50

- Objects in a distributed key-value store: every hop of the DAG walk becomes a network round trip. Google's JGit DHT worked for reads, but clone performance sank it.
- Repos on NFS or GFS: Git's random delta reads and its local-filesystem assumptions make it slow and buggy. GitHub tried NFS early; Bitbucket Data Center kept every repo on a shared NFS mount for years, and Atlassian's own docs call NFS latency fine for streaming packs but punishing for refs and loose objects.
- Conclusion: Git needs local disk, or you rewrite Git around remote storage. "Local disk" from here on means a fast local SSD (NVMe): microseconds per random read instead of a network round trip.

Sources: [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale) · [Atlassian · Bitbucket Mesh (replaces NFS for Git storage)](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)

### 17 · Stage 1: RAID + DRBD pairs

17:50–18:55 · 3 builds

- GitHub's pre-2016 design: an active fileserver with RAID, mirrored block-by-block to a dedicated hot spare with DRBD.
- Packs and refs both live on the active disk; DRBD copies bytes without knowing they are Git.
- Failover was manual, the spare served nothing, and a pair outage took every repo on it offline.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/)

Builds:

1. **Active + hot spare.** RAID inside each server, DRBD mirrors every block to a dedicated spare. Four copies of every byte.
2. **Failover.** Active dies. A human confirms, promotes the spare, traffic moves. Now the spare has no spare.
3. **The limits.** Half the hardware idles, one server serves every read for its repos, and losing the pair takes them all offline.

### 18 · Stage 2: Spokes — replicate Git, not disks

18:55–20:20 · 3 builds

- DGit, later Spokes: real Git repos on several servers (3 by default, up to around 20 for the hottest repos), replicated by Git-aware software, not by blocks.
- Packs fan out freely; only the ref transaction goes through three-phase commit. Reads go to any replica.
- Costs: latency bound by the slowest replica; a floor of 3 copies even for idle repos; a ceiling, because every extra replica slows every push, so even ~20 is tight for monorepo CI; and repos as pets. Atlassian landed in the same place later: Bitbucket Mesh (Data Center 8.0, 2022) replaced NFS with repos replicated on local disk across nodes.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/) · [GitHub · Stretching Spokes (2017)](https://github.blog/engineering/infrastructure/stretching-spokes/) · [Atlassian · Bitbucket Mesh (replaces NFS for Git storage)](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)

Builds:

1. **Fan out the pack.** Packs are idempotent, so send them to every replica in parallel: 3 by default, up to around 20 for the hottest repos. No agreement needed yet.
2. **3PC on the ref transaction.** Only the tiny ref update needs consensus: prepare (lock + check old), pre-commit, commit. A majority must ack.
3. **Reads, and the bill.** Any replica can serve a fetch. The price: tail latency, a fixed floor and ceiling on replicas, and repos you have to babysit.

### 19 · Stage 2b: packs in blobs, refs in a database

20:20–21:35 · 3 builds

- The other road: put immutable packs in blob storage and refs in a database. JGit DFS, Azure DevOps, and CodeCommit all follow this pattern.
- Compute becomes stateless and storage durable, with no replica babysitting.
- The price: you no longer run stock Git on a local repo, so you own a Git implementation, a cache, and a database.

Sources: [JGit · DFS storage layer](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/internal/storage/dfs) · [AWS CodeCommit features (S3 + DynamoDB storage)](https://aws.amazon.com/codecommit/features/) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

Builds:

1. **Pack → blob store.** Packs are immutable files. Any stateless frontend can write one to blob storage.
2. **Ref → database transaction.** The ref update becomes a conditional write in a database: old value must match.
3. **Reads through a cache.** Reads come back through a cache because blob latency per object is too high for Git's random access.

### 20 · Stage 3: WAL-Git — the log is the repo

21:35–23:50 · 6 builds

- Cursor's Continuity: a push is a WAL entry in the object store; it becomes visible only when a compare-and-swap on the WAL index succeeds. Pushes are linearizable.
- The local Git repo is a warm cache. Replicas learn from lossy gossip but verify every read with a conditional GET, so reads are consistent.
- There is no primary: any node can take a push (the CAS on the object store decides) and any node can serve a read, rebuilding from the WAL if it has no copy. Rendezvous hashing just picks a preferred node to avoid CAS retries, and that node does compaction; idle repos are dropped and rebuilt on demand.

Sources: [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale) · [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit)

Builds:

1. **Read the index first.** Before advertising refs to the pusher, the node does the same conditional GET as any read. Now it knows the log is at version e1 with head 41, so this push will be entry #42.
2. **Persist the push as WAL entry #42.** Write the pack to local disk and upload entry #42 to the object store at the same time. Durable, but nothing points at it yet.
3. **Publish with compare-and-swap.** Lock main and check it is still E, then PUT the index with If-Match: e1, the version read in step 1. Success gives e2: the push is acknowledged and linearized.
4. **Gossip to other nodes.** UDP gossip tells other nodes there is news. It can be lost; correctness doesn't depend on it.
5. **Any node can serve a read.** A fetch starts with the same GET index as a push. A warm node sends If-None-Match with the version it has: 304 in under 10 ms means serve now; 200 means pull the new WAL entries first. Cold node 3 has no version yet, so it does a plain GET, learns e2 and head 42, rebuilds the repo from the WAL, then serves.
6. **Any node can take a push.** There is no primary role. Node 2 read e2 and tries to publish #43 with If-Match: e2. If another node got there first, it gets a 412: re-read the index, re-check the ref, retry. Pushes usually go to the rendezvous #1 node only because that avoids retries.

### 21 · What a push writes to the object store

23:50–25:20 · 3 builds

- The contract, as implemented by walgit (an open-source take on Cursor's design): per repository, one tiny mutable WAL index and everything else immutable: log entries, packs, checkpoints. walgit calls the index manifest.pb; Cursor calls it gitwal.pb.
- A push writes a pack, its .idx, and a PUSH log entry holding the pack reference and the ref transaction (main: E to F). Only the conditional PUT on the manifest makes it visible; git hears ok after that.
- After the push: the lease holder compacts packs and publishes a COMPACT entry; a checkpoint snapshots all refs and packs so a cold node starts from it plus the log tail. (Races were covered on the previous slide.)

Sources: [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

Builds:

1. **The contract.** Five kinds of objects per repository. Only the WAL index is mutable, and only by compare-and-swap. Entries, packs, and checkpoints are written once and never change.
2. **What one push writes.** Resolve and verify the pack locally. Write pack P42 and entry #42 (kind PUSH, pack P42, refs main: E to F); they are durable but invisible. Then CAS the WAL index: only if it is still at version e1, set head 42 and add P42. That is the commit point; git hears ok only after it.
3. **Compaction and checkpoints.** Later, the node holding the compaction lease merges P1, P2 and P42 into P-all, writes entry #43 COMPACT, and CASes the index to e3; other nodes download P-all instead of repacking. A checkpoint then snapshots all refs and the pack set at #43, so a cold node loads it and replays only newer entries.

### 22 · Five answers to one question

25:20–26:15

- Same two questions every time: where do packs live, who agrees on refs. 
- The trend: move truth to durable shared storage and treat local copies as cache.
- Throughput numbers are Cursor's own reports for their monorepo, not independent benchmarks.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale) · [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit)

### 23 · Storage scales. Every client still clones.

26:15–26:40

- Everything up to WAL-Git makes the server side scale.
- But each automation still pulls a full checkout before it can run one git command.
- GitFarm moves the execution to where the repo already is.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 24 · The cost of a local checkout at Uber

26:40–27:25

- Six monorepos. Cloning Go takes about 15 minutes and a meaningful slice of a machine.
- Each automation system kept and synced its own checkouts, multiplying upstream load.
- CI caches reduce clone time but add cold starts, drift, and upkeep.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 25 · GitFarm: Git as a service

27:25–28:20

- Clients call a gRPC API. The gateway authenticates, authorizes per repo, and places the request.
- Backend nodes keep a bare clone per repo, synced by push events plus a 5-minute fetch, and pools of pre-warmed checkouts and sandboxes.
- Separate clusters per heavy use case, plus a shared cluster for everyone else.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 26 · One request, six steps

28:20–29:15 · 6 builds

- Request, authenticate, place on the node with the most free checkouts, or throttle.
- A pooled sandbox gets a pooled checkout mounted; commands run as the caller.
- Pools are the whole trick: cold start moves off the request path.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **Request.** Client sends repo id, workspace type, and a list of commands.
2. **Authenticate + authorize.** Gateway identifies the caller and checks they may access this repo.
3. **Place.** Gateway picks the node with the most free checkouts for that repo and marks one occupied. None free → throttle.
4. **Acquire sandbox + checkout.** Backend takes a pre-started sandbox from its pool and mounts a pre-warmed checkout into it.
5. **Execute as caller.** Commands run inside the sandbox with the caller's identity, not the service's.
6. **Release.** Sandbox is released back to the pool; Redis state is updated.

### 27 · Stateful sessions: many commands, one repo state

29:15–30:10 · 3 builds

- The API takes a repo, a workspace type, and a list of commands; a session can chain commands that depend on earlier output.
- Guarantees: isolation, one consistent repo state, ordered execution. Fatal errors end the session.
- The example from the paper: merge-base, rev-parse, push.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **merge-base.** First command: where did this branch diverge from main?
2. **rev-parse.** Same session, same repo state: resolve the tip. Output streams back; the next step can use it.
3. **push.** Push from the sandbox. It goes through exactly the receive-pack path we walked earlier: quarantine, connectivity, ref compare-and-swap.

### 28 · Identity, freshness, isolation

30:10–31:00

- Identity is decoupled from the client environment: the caller's identity flows into the sandbox.
- Freshness is eventual with a 5-minute backstop; clients that need the newest state fetch first.
- Workload isolation by cluster avoids noisy neighbours.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 29 · Who needs Git without a clone?

31:00–31:45

- Two axes: does the workload write, and is someone waiting on it.
- Filled dots are measured production cases in the paper; hollow dots are patterns and planned work.
- Agents and review automation fill every quadrant, which is why a shared execution layer pays off.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 30 · Agents need the monorepo, not a clone

31:45–33:10 · 3 builds

- An agent that clones first spends 15 minutes and 40 GB before it reads a file, and every agent repeats it.
- With GitFarm an agent opens a session in under a second, reads with ordinary git commands, and pushes to a scoped ref.
- This is the direction, not a paper result. The 5-minute session cap matters for long agent loops.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **Clone first.** Cost of a clone-first agent.
2. **Session instead.** Same task through a GitFarm session.
3. **Fan-out.** Many agents share pools; identity scopes their writes.

### 31 · Code review and PR automation

33:10–34:10 · 3 builds

- A PR event starts a session: fetch the PR ref, merge-base, diff, read context files, look up owners.
- Outputs are comments, required reviewers, or writes such as a fix-up commit or base-branch update.
- Measured: ownership inspection at about 100 rps over 350K directories; base-branch changes at about 1 rps with p50 near 25 seconds, mostly Git itself.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **Event → session.** The trigger and the session.
2. **Read the change.** Ordinary git commands, no local clone.
3. **Act on it.** Comments, reviewers, or a push back through receive-pack.

### 32 · Compliance: audit every bypass

34:10–34:50

- Compliance auditing watches deploy and push events and publishes audit PRs for unreviewed commits.
- Median latency fell from roughly two minutes on CI to 20–30 seconds.
- Only p50 is reported; the paper notes tail latency is dominated by Git itself.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 33 · Results: give the resources back

34:50–35:30

- The code ownership service dropped local checkouts for all six monorepos.
- Fleet CPU fell from about 90 to 16 cores and memory from about 472 to 32 GB; node startup went from 15–20 minutes to under a minute.
- Caveat: results come from one shared four-node cluster and one service per case.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 34 · What's next

35:30–36:05

- Planned: SubmitQueue merge-to-trunk running in sandboxes, and mirroring across upstreams.
- Engine work: streaming output, sparse workspaces, longer sessions.
- The direction: agents become the main clients of the Git control plane.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 35 · Git became a distributed system

36:05–36:30

- Storage, agreement, and execution each moved from a single machine to a distributed service.
- The thread the whole way: big immutable packs, tiny atomic ref updates.
- Agents are driving the next round, at both the storage and the execution layer.

### 36 · Thank you

36:30–36:40

- Thank you.
- The GitFarm paper, the Cursor post, and walgit are the deep references.
- Git's own docs on pack format and protocol cover everything from Part 1 and 2.
