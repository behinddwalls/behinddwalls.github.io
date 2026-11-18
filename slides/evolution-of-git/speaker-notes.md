# Evolution of Git — speaker cues

Planned at about 26 minutes for a 30–45 minute slot · 29 slides. Each slide has three short cues, and build slides also have one cue per step.

## Run of show

| Slide | Start | Length | Title |
| --- | --- | --- | --- |
| 01 | 00:00 | 00:20 | Evolution of Git |
| 02 | 00:20 | 00:15 | About me |
| 03 | 00:35 | 00:50 | Built for one very unusual project: Linux |
| 04 | 01:25 | 00:10 | Part 1: What's actually in a repository? |
| 05 | 01:35 | 01:00 | Git's history is made of objects |
| 06 | 02:35 | 01:10 | Files are blobs, directories are trees |
| 07 | 03:45 | 01:20 | A commit is a snapshot that shares structure |
| 08 | 05:05 | 01:15 | Refs give names to commits |
| 09 | 06:20 | 01:30 | Where objects live: loose files and packs |
| 10 | 07:50 | 01:00 | How a delta is stored |
| 11 | 08:50 | 00:10 | Part 2: How does it move between machines? |
| 12 | 09:00 | 02:00 | git fetch origin main |
| 13 | 11:00 | 02:00 | git push origin main |
| 14 | 13:00 | 00:20 | Part 3: How do you serve it to everyone? |
| 15 | 13:20 | 00:40 | Stage 0: one box |
| 16 | 14:00 | 00:45 | Why not just put the repos on NFS? |
| 17 | 14:45 | 01:05 | Stage 1: RAID + DRBD pairs |
| 18 | 15:50 | 01:25 | Stage 2: Distributed Git — replicate Git, not disks |
| 19 | 17:15 | 02:00 | Stage 3: WAL-Git — a write-ahead log is the repo |
| 20 | 19:15 | 01:15 | WAL-Git: where refs live and how nodes catch up |
| 21 | 20:30 | 00:10 | Part 4: What if clients never clone? |
| 22 | 20:40 | 00:45 | The cost of a local checkout |
| 23 | 21:25 | 00:40 | Warm checkouts: remote devboxes and GitFarm |
| 24 | 22:05 | 00:55 | GitFarm: Git as a service |
| 25 | 23:00 | 00:55 | One request, end to end |
| 26 | 23:55 | 00:45 | Identity, freshness, isolation |
| 27 | 24:40 | 00:45 | Who needs Git without a clone? |
| 28 | 25:25 | 00:25 | The infrastructure went distributed |
| 29 | 25:50 | 00:10 | Thank you |

## Speaking cues

### 01 · Evolution of Git

00:00–00:20

- Git was designed with no server. Today it is the center of a distributed system.
- Four parts: the data model, fetch and push on the wire, how servers evolved, and GitFarm.
- Goal: you leave knowing what actually happens underneath git push.

### 02 · About me

00:20–00:35

- One line: Core Developer Experience at LinkedIn; before that Developer Platform at Uber, where GitFarm comes from; before that Amazon.
- Keep it to ten seconds and move on.
- The QR code goes to LinkedIn for anyone who wants to connect.

### 03 · Built for one very unusual project: Linux

00:35–01:25

- Linus Torvalds wrote Git for the kernel's decentralized workflow: many maintainers, patches and pulls between peers.
- The original design has no privileged copy. Every clone holds the entire history.
- Keep that in mind: everything later in this talk is about making one copy special again.

### 04 · Part 1: What's actually in a repository?

01:25–01:35

- Every clone is the whole repository. So what's actually in it?
- Four kinds of object, a few refs, and how they're stored.

### 05 · Git's history is made of objects

01:35–02:35

- History is made of objects. Each is a type, a size, and content, stored compressed and named by the SHA-1 of the uncompressed bytes. Commit, tree, and blob are the chain that matters; the fourth type is the tag object, used by annotated and signed tags (the tag ref points to it; a lightweight tag is just a ref, like a branch). The stored bytes are a small header (type, a space, the size, a NUL byte) followed by the content; the SHA-1 of those bytes is the object's name.
- Objects never change. Editing a file creates a new blob; the old one is still there under its old name.
- On top sits a ref: not an object, just a file holding a SHA, and the only mutable thing in Git. That split drives the rest of the talk. (Try it: git cat-file -t ce0136 prints blob; git cat-file -t HEAD prints commit.)
- Coming in Git 3.0 (no release date yet): new repositories default to SHA-256 instead of SHA-1. Existing repos keep SHA-1, and the object model is the same; only the hash, and so the names, get longer (64 hex digits instead of 40).

### 06 · Files are blobs, directories are trees

02:35–03:45

- A blob is the file's bytes with a small header; its name is the SHA-1 of that. No filename, no metadata.
- A tree maps names and modes to blob or tree hashes (mode 100644 = file, 100755 = executable, 040000 = directory). Identical content anywhere in history is stored exactly once.
- Because names are derived from content, any copy can verify any object. That is what makes replication safe later.

### 07 · A commit is a snapshot that shares structure

03:45–05:05 · 3 builds

- A commit is a root tree plus parent pointers plus metadata. Not a diff.
- A new commit only adds objects on the path from the root to what changed; everything else is shared.
- It is a Merkle DAG: trees hash their children's hashes, commits hash their tree and parents. Anyone can verify what they receive, and an identical subtree hash means you can skip the whole directory. Walking it still means following pointers one object at a time; remember that for the server section.

Builds:

1. **One commit, one snapshot.** A commit points to one root tree. Walk the tree and you have every file.
2. **Second commit shares structure.** Edit server.ts. Only the path from root to that file gets new objects; README and pack.ts are reused.
3. **Count the new objects.** One changed file means one blob, two trees, one commit: the new hash ripples up the path. Hashes of hashes make this a Merkle DAG, so one commit hash covers every file and all of history.

### 08 · Refs give names to commits

05:05–06:20 · 3 builds

- A ref is a 41-byte file holding a SHA. Branches move with each commit, tags stay put, remote-tracking refs remember the server as of the last fetch, and HEAD names the current branch. As they multiply, Git moves them from loose files to one packed-refs file, or to reftable. Git 3.0 plans to make reftable the default for new repos.
- If asked about reftable: Git starts with one loose file per ref: simple, but 100k refs means 100k files, and on macOS or Windows two names that differ only in case collide. gc packs them into packed-refs, one sorted text file: fast lookups, but deleting a ref rewrites the whole file, which can be tens of megabytes, and an update goes back to a loose file until the next gc. reftable (in Git since 2.45; designed at Google for JGit) keeps refs in a small stack of binary files. Inside each file, refs are sorted by name and grouped into blocks of about 4 KB; neighboring names share their prefix (refs/heads/feature-…), so it is stored once. Finding a ref is a binary search, not a scan. An update never edits an existing file: it writes a new tiny table holding only the changed refs and adds it to the top of the stack in one atomic step, so updating many refs at once is all-or-nothing. Readers check the newest table first. In the background, small tables are merged into bigger ones (like compaction in a log-structured store).
- Updating a ref is lock, compare, rename: a tiny compare-and-swap. Every write to a repo ends with one, and it is the only mutable thing in Git.
- Reachability from refs defines what the repo contains. Unreachable objects are eventually garbage-collected; reflogs keep them alive for a while first (by default about 30 days, then a two-week grace period).

Builds:

1. **Kinds of ref.** A ref is a tiny file holding a SHA. Branches under refs/heads move with each commit; tags under refs/tags stay put; refs/remotes/origin/* remember where the server was at the last fetch. HEAD is symbolic: it names a branch, not a commit.
2. **Commit E: main moves.** Committing writes the new objects first, then updates one ref: lock main, check it still points at D, write E, rename the lock into place. That compare-and-swap is the whole update. HEAD follows because it names main. Every push ends with the same step on the server.
3. **Reachability defines the repo.** X was committed on a branch that has since been deleted. No ref reaches it, so it is no longer part of the repo and gc eventually prunes it. Everything a ref can reach is kept: every ref is a garbage-collection root.

### 09 · Where objects live: loose files and packs

06:20–07:50 · 4 builds

- New objects are written loose, one file each. gc, repack, or a large fetch or push rolls many objects into one packfile plus its .idx.
- Inside a pack, older versions are deltas against newer ones, placed for size rather than by commit.
- To read any object, Git binary-searches each pack's .idx for the offset, then falls back to a loose file. Remember A–E: the fetch slide uses the same history.
- Database hook: a pack is like an SSTable in an LSM tree (RocksDB, Cassandra). Small writes land loose, gc merges them into one immutable file the way compaction does, and the .idx is the index that finds a key without a scan.

Builds:

1. **Every commit starts loose.** Each new object is its own zlib file under .git/objects, named by its hash: git add writes the blob, git commit writes the trees and the commit.
2. **gc rolls them into a pack.** Auto gc runs after commit or fetch once there are about 6,700 loose objects or 50 packs (or on a schedule with git maintenance). A fetch of 100+ objects is kept as a pack; a big push lands as one on the server. The id in the name is a checksum of the contents. D and E came later, so they are still loose.
3. **Inside the pack.** Every object is zlib-compressed. Blobs and trees can be deltas against a similar nearby object, usually newest whole and older as deltas. Chains are capped at 50. Entries are placed for size, not by commit.
4. **The .idx finds things.** The .idx is sorted by hash: a 256-entry fanout jumps to rows sharing the first byte, binary search finds the row, and the row gives the offset (plus a CRC to check the stored bytes). The .pack itself stays in write order.

### 10 · How a delta is stored

07:50–08:50

- A delta is a recipe: COPY byte ranges from a base object, INSERT the bytes the base doesn't have. src/ is a tree: two entries (pack.ts, server.ts) of mode, name, and file hash, 72 bytes. Editing server.ts in commit E changes only its hash. Matching the newest-whole rule, E's src/ is stored whole and D's src/ becomes: copy bytes 0–51, insert the old 20-byte hash. 25 bytes instead of 72 (2 for the two sizes, 2 for the COPY, 21 for the INSERT). Left vs right: without a delta both trees are stored whole, 144 B, and the first 52 bytes are stored twice; with a delta it is 72 + 25 = 97 B. A commit is mostly new hashes and timestamps, so it usually stays whole.
- Same idea for file contents: server.ts (~12 KB) edited 100 times is 100 objects, about 1.2 MB of content (each loose file is zlib-compressed, so less on disk). Packed, the newest is whole and the rest are ~200-byte deltas, a few tens of KB in total; Git starts a new base only if a chain would pass 50 steps.
- Deltas are made when packing, reused when sending, and resolved whenever someone reads an old version.
- Database hook: deltas are like MVCC undo chains in InnoDB or Oracle. The newest row is stored whole; older versions are rebuilt by walking the undo chain backward, so history costs a walk and long chains get slow (hence the cap at 50). PostgreSQL keeps full row versions instead, much like loose objects, with VACUUM in roughly gc's role.
- If asked why 52: bytes 0–34 are "100644 pack.ts\\0" plus pack.ts's hash (6 + 1 + 7 + 1 + 20 = 35 B); bytes 35–51 are "100644 server.ts\\0" (6 + 1 + 9 + 1 = 17 B); bytes 52–71 are server.ts's hash (20 B).

  | Bytes | Contents | Size |
  |---|---|---|
  | 0–34 | `100644 pack.ts\0` + pack.ts hash | 35 B (6 + 1 + 7 + 1 + 20) |
  | 35–51 | `100644 server.ts\0` | 17 B (6 + 1 + 9 + 1) |
  | 52–71 | server.ts hash | 20 B |

### 11 · Part 2: How does it move between machines?

08:50–09:00

- That's the data on one machine. Now move it between two.
- Fetch, then push, with the same A–E history.

### 12 · git fetch origin main

09:00–11:00 · 5 builds

- Both ends are just Git: fetch is fetch-pack talking to upload-pack, push is send-pack talking to receive-pack, over ssh or HTTPS.
- Same history as the pack slide: the server has A–E (A–C packed, D and E loose); the client stopped at B.
- want names ref tips; have walks back through the client's history until the server recognizes a commit. The server then walks from E back to B, finding objects loose or through the .idx.
- It ships C, D, E as one fresh pack; the client re-indexes it into its own pack, and only then moves origin/main, with a local compare-and-swap (still B? then E). At scale, the first step is the expensive one: protocol v0 advertises every ref before any work (a million refs is tens of MB), so v2's ls-refs lets the client ask only for the prefixes it needs.

Builds:

1. **Ref advertisement.** The client asks which refs exist. The server answers from its ref files: main points at E. No objects are touched yet.
2. **want / have.** want = the tip of each ref the client is missing (E). have = the client's own tips first (B), then older commits in batches only if needed. Once the server ACKs B, it knows the client has everything behind B, including A.
3. **Server walks the graph.** Walk back from E until you hit what the client has. E and D are found loose; C is found through the pack's .idx; B is common, so stop.
4. **Server builds one pack.** The sender builds; the receiver resolves. pack-objects puts C, D, E and their new trees and blobs in one pack. C's bytes are reused from the existing pack (CRC-checked, never re-inflated); D and E are compressed now. Deltas against B's objects are allowed: a thin pack.
5. **Client resolves and moves the ref.** Whoever receives a pack resolves it. Pack entries don't carry their own hashes, so index-pack rebuilds every delta from its base, SHA-1s the result, and writes the .idx. Thin bases come from the client's own store (B) and are appended. Only after the connectivity check does origin/main move to E. origin/main moves with the same local compare-and-swap as slide 8: lock, check it's still B, write E, rename. It only guards your own repo; the server's main is never touched.

### 13 · git push origin main

11:00–13:00 · 6 builds

- Continuing the A–E story: the client commits F on top of E and pushes. It sends only what the server lacks, plus the command E F refs/heads/main.
- The server quarantines, verifies connectivity, runs hooks, then does a ref transaction: lock, check old, write.
- That ref transaction is a compare-and-swap. Keep it in mind for every architecture that follows.

Builds:

1. **Connect, server advertises refs.** The client goes first: it connects over ssh or HTTPS and asks the server to run receive-pack. From then on the server speaks first, telling the client where every ref points right now.
2. **Client builds the pack.** The client committed F on top of E. On push the client is the sender, so it builds: only objects reachable from F but not from E (F, its new trees, the changed blob), possibly as thin deltas against E's objects.
3. **Commands + pack.** Each command is old, new, ref: here E F refs/heads/main. The pack follows on the same stream.
4. **Server resolves in quarantine.** Now the server is the receiver, so it resolves: index-pack rebuilds every delta, SHA-1s each object, fills thin bases from E's objects, and writes the .idx, all inside a quarantine directory. Then the connectivity check on F. If anything fails, the quarantine is deleted and nothing becomes visible.
5. **Ref transaction.** pre-receive can reject. Then lock main, check it still points at E, write F, commit. If someone else pushed G first, main is no longer E and this push is rejected. This is compare-and-swap.
6. **Report status.** The client gets per-ref ok or ng, then post-receive fires (webhooks, CI).

### 14 · Part 3: How do you serve it to everyone?

13:00–13:20

- Git was built with no server. Hosting made one copy special.
- A push is two different things. The pack is immutable and content-addressed: write it twice and nothing breaks, so it can land on any disk before anyone agrees. The ref update is mutable and must be atomic: it is the only part that needs agreement, a compare-and-swap.
- So every stage that follows answers the same two questions: where do packs live, and who decides which push wins? Use that lens for one box, RAID pairs, distributed Git, and WAL.

### 15 · Stage 0: one box

13:20–14:00

- The simplest Git server is one machine: sshd or git-daemon runs upload-pack and receive-pack against bare repos on local disk.
- Packs and refs both live on that disk, and the box agrees with itself.
- It is fast and simple until the disk, the CPU, or the machine runs out.

### 16 · Why not just put the repos on NFS?

14:00–14:45

- The obvious next step after one box: keep stock Git, but put every repo on a shared NFS filer that all servers mount. Any server can serve any repo.
- Git assumes a local disk. Delta walks become many small network reads, and ref updates depend on lock files and atomic rename over NFS. Atlassian's own docs call NFS fine for streaming packs but punishing for refs and loose objects; GitHub tried it early, and Bitbucket Data Center ran on it for years before Bitbucket Mesh. I ran Phabricator's repositories on NFS at Uber, so this is first-hand: say what broke first.
- Conclusion: Git needs local disk, or you rewrite Git around remote storage. "Local disk" from here on means a fast local SSD (NVMe): microseconds per random read instead of a network round trip.

### 17 · Stage 1: RAID + DRBD pairs

14:45–15:50 · 3 builds

- RAID 10 (Redundant Array of Independent Disks) mirrors disks in pairs inside one server; DRBD (Distributed Replicated Block Device) mirrors every disk write over the network to a second server. GitHub's pre-2016 design: an active fileserver with RAID, mirrored block-by-block to a dedicated hot spare with DRBD.
- Packs and refs both live on the active disk; DRBD copies bytes without knowing they are Git.
- Failover was manual, the spare served nothing, and a pair outage took every repo on it offline.

Sources: [GitHub · How We Made GitHub Fast (2009)](https://github.blog/news-insights/the-library/how-we-made-github-fast/) · [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/)

Builds:

1. **Active + hot spare.** RAID inside each server, DRBD mirrors every block to a dedicated spare. Four copies of every byte.
2. **Failover.** Active dies. A human confirms, promotes the spare, traffic moves. Now the spare has no spare.
3. **The limits.** Half the hardware idles, one server serves every read for its repos, and losing the pair takes them all offline.

### 18 · Stage 2: Distributed Git — replicate Git, not disks

15:50–17:15 · 3 builds

- Distributed Git: GitHub DGit, later Spokes (2016); GitLab Gitaly Cluster (2020); Bitbucket Mesh (2022). Real Git repos on several servers (3 at launch, 5 for most repos today per GitHub), replicated by Git-aware software, not by blocks.
- Packs fan out freely; only the ref transaction goes through three-phase commit. Reads go to any replica.
- Costs: latency bound by the slowest replica; a floor of 3 copies even for idle repos; a ceiling, because every extra replica slows every push, so adding replicas for read load makes pushes slower; and repos as pets. Atlassian landed in the same place later: Bitbucket Mesh (Data Center 8.0, 2022) replaced NFS with repos replicated on local disk across nodes.

Sources: [GitHub · Introducing DGit (2016)](https://github.blog/engineering/architecture-optimization/introducing-dgit/) · [GitHub · Stretching Spokes (2017)](https://github.blog/engineering/infrastructure/stretching-spokes/) · [GitHub · Building Git infrastructure for agent-scale development (Oct 2026)](https://github.blog/engineering/architecture-optimization/building-git-infrastructure-for-agent-scale-development/) · [Atlassian · Bitbucket Mesh (2022, replaces NFS for Git storage)](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)

Builds:

1. **Fan out the pack.** Packs are idempotent, so send them to every replica in parallel: 3 at launch, 5 for most repos today. No agreement needed yet.
2. **3PC on the ref transaction.** Only the tiny ref update needs agreement. Phase 1 asks every replica: can you apply these ref updates? Each runs Git's own ref transaction: lock the refs, check the old values. Phase 2: a majority said yes, so get ready. Phase 3: commit, or roll back everywhere. Spokes then compares a checksum of every replica's refs; one that disagrees is marked unhealthy and repaired. About four round trips per push.
3. **Reads go to any replica.** Any replica can serve a fetch; a routing DB says which servers hold each repo. Say the cost out loud: every push waits for the slowest replica, even idle repos need 3+ copies, adding replicas slows pushes, and the replicas need checksums, repair jobs and that routing DB.

### 19 · Stage 3: WAL-Git — a write-ahead log is the repo

17:15–19:15 · 5 builds

- From Stage 2, two things change: packs leave the replicas for a blob store, and the vote becomes one compare-and-swap.
- Cursor's Continuity: a push is a WAL entry in the blob store; it becomes visible only when a compare-and-swap on the WAL index succeeds. Pushes are linearizable.
- The local Git repo is a warm cache. Replicas learn from lossy gossip but verify every read with a conditional GET, so reads are consistent.
- There is no primary: any node can take a push (the CAS on the blob store decides) and any node can serve a read, rebuilding from the WAL if it has no copy. Rendezvous hashing just picks a preferred node to avoid CAS retries, and that node does compaction; idle repos are dropped and rebuilt on demand. Kept from distributed Git: real Git on local disk. New: blob storage is the truth. Dropped: quorum, primary, routing DB. Cursor reports about 120 pushes/s on S3 Standard and 300+ on S3 Express One Zone, with reads scaling linearly to 100 replicas.

- The WAL index is one small object, overwritten only by compare-and-swap: a version (what the CAS checks), the head entry number, the list of live packs (here base, an earlier compaction, plus one pack per push since; push #42 adds P42), and a pointer to the latest checkpoint (a snapshot of every ref, here as of entry #40). There is no refs field: current refs are the checkpoint plus the entries after it. A cold node reads the checkpoint plus the entries after it, never the whole log.
- Where the WAL index lives is the design choice. Cursor keeps it in the blob store and publishes with one CAS on it, so every push to the repo goes through that one object. GitHub's Rubicon (2026) keeps packs in blob storage too but moves the log into Ref-CAS, a separate quorum service that locks only the refs a push touches, so pushes to different branches don't wait on each other.
- If asked about refs in a database instead of a log: JGit DFS (Google's Gerrit hosting), Azure DevOps (blob + SQL Server), AWS CodeCommit (S3 + DynamoDB), and Entire (2026: packs on replicas, refs in PostgreSQL) keep one row per ref and move it with UPDATE … WHERE value = old. Same compare-and-swap, without the log.
Sources: [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale) · [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit) · [GitHub · Inside GitHub's new Git storage architecture (Nov 2026)](https://github.blog/engineering/architecture-optimization/inside-githubs-new-git-storage-architecture/)

Builds:

1. **1 · Read the index first.** Before advertising refs to the pusher, the node does the same conditional GET as any read. Now it knows the log is at version e1 with head 41, so this push will be entry #42. The version is the object's ETag on S3 or its generation on GCS.
2. **2 · Persist the push as WAL entry #42.** Write the pack to local disk and upload entry #42 to the blob store at the same time. Durable, but nothing points at it yet.
3. **3 · Publish with compare-and-swap.** Lock main and check it is still E, then PUT the index with If-Match: e1, the version read in step 1. Success gives e2: the push is acknowledged and linearized.
4. **4 · Any node can serve a read.** UDP gossip tells other nodes there is news; it can be lost, and correctness doesn't depend on it. A fetch starts with the same GET index as a push. A warm node sends If-None-Match with the version it has: 304 in under 10 ms means serve now; 200 means pull the new WAL entries first. Cold node 3 has no version yet, so it does a plain GET, learns e2 and head 42, rebuilds the repo from the WAL, then serves.
5. **5 · A push loses the race.** There is no primary role: node 2 takes a push too. It read e2, so its push should become #43, and it tries the CAS: only if still e2. But another push already published #43, so the index is e3 and node 2 gets a 412. There is only ever one #43: entries are created only if absent, and whatever node 2 uploaded is never published, so no reader sees it. Node 2 re-reads the index and re-checks the old value of every ref it is updating: if one moved, the push is rejected, like a non-fast-forward; if not, it retries at the next number. Pushes usually go to the rendezvous #1 node only because that avoids these retries.

### 20 · WAL-Git: where refs live and how nodes catch up

19:15–20:30 · 3 builds

- Truth is only the index, the log and the checkpoint. With 100k refs, the index holds none, a log entry holds only the refs its push changed, and the checkpoint holds them all (a few MB for 100k refs, an estimate). walgit's rule: cost must not scale with ref count on any hot path.
- The latest value per ref is each node's own packed-refs, folded from checkpoint plus log: a cache. The pushing node updates its cache before acking; everyone else catches up when asked, with a conditional GET (304: current; 200: apply the new entries). Gossip may nudge them sooner, best effort. Idle nodes can stay behind for hours, harmlessly.
- If asked: walgit can skip the check for a short window (wal.freshness_ttl), trading a little staleness for fewer requests; Cursor checks on every read. walgit's reference monorepo has 466k refs; refs from a cold node in under a second is a target, not a measurement.

Sources: [walgit · open-source Git server on object storage (Continuity design)](https://github.com/tobi/walgit) · [Cursor · Git at any scale (Vicent Martí, Aug 2026)](https://cursor.com/blog/git-at-any-scale)

Builds:

1. **Truth lives in the blob store.** Three kinds of object, and only one of them holds every ref. The index holds no refs, so it stays a few KB. A log entry holds only the refs its push changed: main E to F. The checkpoint holds all 100k refs: one full snapshot, rewritten every N entries, on a timer, or when the tail gets long. A push touching one branch costs the same with 10 refs or 100k.
2. **Each node keeps its own cache.** Nothing in the bucket says main = F right now. Each node folds checkpoint plus log into its own packed-refs: a sorted file with the latest value of every ref. That is the per-ref index, and it is a cache. Right after push #42 they disagree: node 1 took the push and updated its refs before it said ok, so it is at e2. Node 2 is still at e1. Node 3 has nothing yet.
3. **Caches catch up on read.** Nobody broadcasts the push. Each node checks before it serves: a conditional GET on the index. Node 1 gets 304 and serves at once. Node 2 gets 200, reads only entry #42, applies main E to F, and serves. Node 3 reads the checkpoint and the tail in parallel, folds them into packed-refs, and serves refs in two rounds; packs come later, only if a fetch needs them. In Cursor's design gossip can nudge a node to catch up early, best effort. That is why the per-ref check in a CAS is safe: a node checks refs at a version, and the CAS only succeeds if that version is still current.

### 21 · Part 4: What if clients never clone?

20:30–20:40

- Storage scales now, but every client still clones.
- Agents, review bots and automation don't need a copy of the repo; they need to run Git against it.

### 22 · The cost of a local checkout

20:40–21:25

- Cloning a large monorepo takes about 15 minutes and a meaningful slice of a machine: 40+ GB of disk, plus cores and memory.
- Say the contrast: an agent that needs a 2-second git command still pays the whole clone first. Checkout sits on the critical path.
- If asked: each automation system kept and synced its own checkouts, multiplying upstream load. CI caches reduce clone time but add cold starts, drift, and upkeep.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 23 · Warm checkouts: remote devboxes and GitFarm

21:25–22:05

- A remote devbox (or devpod) is a dev machine in the data center that already has the monorepo checked out and kept in sync in the background. A developer connects with their IDE or shell and starts working: no 15-minute clone.
- GitFarm does the same for automation. Each node keeps a bare clone fresh from push events plus a periodic fetch, and a pool of warm checkouts.
- An agent, bot or service borrows one checkout per request, runs its git commands, and the checkout goes back to the pool.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 24 · GitFarm: Git as a service

22:05–23:00

- Clients call a gRPC API. The gateway authenticates, authorizes per repo, and places the request.
- Every node runs three containers: backend serves the request; sandbox keeps a pool of sandboxes and executes the commands; repo-manager keeps the bare clone, synced by push events plus a 5-minute fetch, and a pool of pre-warmed checkouts.
- Separate clusters per heavy use case, plus a shared cluster for everyone else.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

### 25 · One request, end to end

23:00–23:55 · 5 builds

- Request, authenticate, place on the node with the most free checkouts, or throttle.
- The backend takes a sandbox from the sandbox container's pool and a checkout from repo-manager's pool, mounts one into the other, and runs the commands as the caller.
- Pools are the whole trick: cold start moves off the request path.
- A request can be a session: several commands run in order on one repo state (the paper's example is merge-base, rev-parse, push), and the push goes through the same receive-pack and ref compare-and-swap from Part 2.

Sources: [GitFarm: Git as a Service for Large-Scale Monorepos (arXiv 2604.11977)](https://arxiv.org/abs/2604.11977) · [Uber Engineering · GitFarm as a Service (Jul 2026)](https://www.uber.com/blog/gitfarm-as-a-service/)

Builds:

1. **Request.** Client sends repo id, workspace type, and a list of commands.
2. **Authenticate + authorize.** Gateway identifies the caller and checks they may access this repo.
3. **Place.** Gateway picks the node with the most free checkouts for that repo, marks one occupied, and hands the request to that node's backend. None free → throttle.
4. **Acquire sandbox + checkout.** The backend first asks repo-manager for a pre-warmed checkout from its pool, then mounts it on a pre-started sandbox.
5. **Execute as caller.** Commands run inside the sandbox with the caller's identity, not the service's. Fetches and pushes go from the sandbox straight to upstream; repo-manager also fetches from upstream to keep its bare clone fresh. When the commands finish, the backend releases the sandbox and the checkout back to their pools.

### 26 · Identity, freshness, isolation

23:55–24:40

- Identity: the gateway checks the caller's access to the repo, and the sandbox runs with the caller's privileges, not the client host's.
- Freshness: push events plus a periodic fetch every 5 minutes keep bare clones current. Need the newest state? Start the session with git fetch. Can tolerate lag? Skip it and save the latency. Contrast with WAL-Git: strong consistency at the storage layer; GitFarm lets each request choose.
- Isolation: heavy or latency-sensitive workloads get their own cluster; a shared cluster onboards the rest. New use cases go through a sizing review: pool sizes, limits, sync policy.

### 27 · Who needs Git without a clone?

24:40–25:25

- Two axes: does the workload write, and is someone waiting on it. Agents and review automation fill every quadrant, which is why a shared execution layer pays off.
- Security scanning (secrets, vulnerable dependencies) is a read-only batch job: it needs the code at a commit, not a clone.
- Measured in production: compliance audits went from 110–160 s p50 on Buildkite to 20–30 s, more than 80% lower, and the code ownership service dropped from about 90 to 16 cores and 472 to 32 GB of memory. Agents open a session in under a second instead of a 15-minute, 40 GB clone.

### 28 · The infrastructure went distributed

25:25–25:50

- Storage, agreement, and execution each moved from a single machine to a distributed service. If you want the full path: storage went one disk, mirrored disks, replicas, a log in the blob store; agreement went the box, the active node, 3PC, one compare-and-swap; execution went every client's clone to a shared, pre-warmed service. Close on the arc: Git was built so every clone is a full repo, then the world hosted it centrally, and now that central host is itself a distributed system.
- The thread the whole way: big immutable packs, tiny atomic ref updates.
- Agents are driving the next round, at both the storage and the execution layer.

### 29 · Thank you

25:50–26:00

- Thank you.
- Open the floor: questions on Git internals, the server designs, or GitFarm.
- If asked for reading: the GitFarm paper, Cursor's post, walgit, and Git's own docs on pack format and protocol.
