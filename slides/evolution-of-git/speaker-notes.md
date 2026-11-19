# Evolution of Git — speaker cues

Planned at 24:25 (about 24 minutes) for a 30–45 minute slot · 27 slides. Speaking cues match the HTML deck; build slides also have one cue per step.

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
| 16 | 14:00 | 01:05 | Stage 1: RAID + DRBD pairs |
| 17 | 15:05 | 01:25 | Stage 2: Quorum-based Git |
| 18 | 16:30 | 02:25 | Stage 3: log-based Git storage |
| 19 | 18:55 | 00:10 | Part 4: A Git client in the cloud? |
| 20 | 19:05 | 00:45 | The cost of a local checkout |
| 21 | 19:50 | 00:40 | Warm checkouts: remote devboxes and GitFarm |
| 22 | 20:30 | 00:55 | GitFarm: Git Client as a service |
| 23 | 21:25 | 00:55 | One request, end to end |
| 24 | 22:20 | 00:45 | Identity, freshness, isolation |
| 25 | 23:05 | 00:45 | Where GitFarm fits |
| 26 | 23:50 | 00:25 | Git stayed distributed. Hosting evolved. |
| 27 | 24:15 | 00:10 | Thank you |

## Speaking cues

### 01 · Evolution of Git

00:00–00:20

- Git was designed with no required central server. Today, teams often rely on a large hosting system.
- Four parts: the data model, fetch and push on the wire, how servers evolved, and GitFarm.
- Goal: you leave knowing what actually happens underneath git push.

### 02 · About me

00:20–00:35

- One line: Core Developer Experience at LinkedIn; before that Developer Platform at Uber, where GitFarm comes from; before that Amazon.
- Outside of work: hiking, climbing, skiing, camping. One beat, then move on.
- The QR code goes to LinkedIn for anyone who wants to connect.

### 03 · Built for one very unusual project: Linux

00:35–01:25

- Linus Torvalds wrote Git for the kernel's decentralized workflow: many maintainers, patches and pulls between peers.
- The original design has no privileged copy. A full clone carries the history reachable from its fetched refs; shallow and partial clones deliberately take less.
- Hosting makes one copy authoritative by convention, not by changing Git's distributed object model.

### 04 · Part 1: What's actually in a repository?

01:25–01:35

- Start with a local repository. What's actually in it?
- Four kinds of object, refs that name them, and how they're stored.

### 05 · Git's history is made of objects

01:35–02:35

- History has four object types: blob, tree, commit and tag. Annotated tags use tag objects and can optionally be signed; lightweight tags are refs. These examples use SHA-1. Git hashes the type, space, size, NUL byte and content before compression; SHA-256 repositories use the same object model.
- Object contents do not change. Editing content creates a new object ID; repacking can change its physical representation, and garbage collection can remove unneeded objects.
- A ref is a mutable name, not a Git object. Git has other mutable metadata, including the index and reflogs; refs are the publication mechanism we follow through the talk.
- Git 3.0 plans to default new repositories to SHA-256. There is no release date yet; this is not an automatic migration of existing SHA-1 repositories.

### 06 · Files are blobs, directories are trees

02:35–03:45

- A blob holds file contents, not a filename or file mode. Its object ID hashes the content with the type and size header.
- A tree maps names and modes to object IDs (mode 100644 = file, 100755 = executable, 040000 = directory). Identical blob contents share an ID and can be reused across paths and commits; physical duplicates may still exist across packs or replicas.
- Because IDs are derived from content, any copy can verify an object. That is useful when we replicate it later.

### 07 · A commit is a snapshot that shares structure

03:45–05:05 · 3 builds

- A commit is a root tree plus parent pointers plus metadata. Not a diff.
- A new commit only adds objects on the path from the root to what changed; everything else is shared.
- It is a Merkle DAG: trees hash their children's hashes, commits hash their tree and parents. Anyone can verify what they receive, and an identical subtree hash means you can skip the whole directory. Walking it still means following pointers one object at a time; remember that for the server section.

Builds:

1. **One commit, one snapshot.** A commit points to one root tree. Walk the tree and you have every file.
2. **Second commit shares structure.** Edit server.ts. Only the path from root to that file gets new objects; README and pack.ts are reused.
3. **Count the new objects.** In this example, one changed file means one blob, two trees, one commit: the new hash ripples up the path. Hashes of hashes make this a Merkle DAG, so one commit ID covers its snapshot and reachable history.

### 08 · Refs give names to commits

05:05–06:20 · 3 builds

- Branches are mutable names; tags conventionally stay put; remote-tracking refs remember the last fetched state. HEAD usually names a branch, but can point directly to a commit when detached. A loose SHA-1 direct ref is a 41-byte file; symbolic refs, packed refs and reftable have different representations.
- With the files backend, updating a packed branch writes a loose override. Deleting a packed ref rewrites packed-refs; a normal branch update does not. Reftable uses sorted binary tables: updates add a table and compaction merges tables. Git 3.0 plans to make it the default for new repos, with no release date yet and no automatic conversion of existing repos.
- Lock, compare and rename implements the old-value check in this files-backend example. Ref backends can implement it differently; not every repository write updates a ref.
- Refs protect reachable objects, but reflogs, the index and retention rules also protect objects. Unreachable reflog entries default to 30-day expiry, other entries to 90 days; pruning has its own age cutoff. These are separate rules, not a guaranteed 30 days plus two weeks.

Builds:

1. **Kinds of ref.** Refs give names to object IDs; symbolic refs name another ref. These examples show loose files. Branches move with commits; tags conventionally stay put; remote-tracking refs remember the last fetched state. HEAD usually names a branch, but can point directly to a commit when detached.
2. **Commit E: main moves.** With the files backend, committing writes the objects first, then updates the branch: lock main, check it still points at D, write E, rename the lock into place. HEAD follows because it names main. A branch push uses the same old-value check on the server; other ref backends implement it differently.
3. **Reachability and retention.** X was committed on a branch that has since been deleted. No branch or tag reaches it, but reflogs and the index may still protect it. Garbage collection can remove it only once those protections and retention rules permit it.

### 09 · Where objects live: loose files and packs

06:20–07:50 · 5 builds

- Local edits can write objects loose, one file each. gc and repack collect objects into packs; large fetches and pushes can also arrive as packs, with a matching .idx.
- Pack entries are compressed whole objects or deltas. Base selection is heuristic, and the default delta-depth limit of 50 is configurable; neither delta direction nor physical order is fixed by history.
- The .idx maps object IDs to offsets. This diagram follows a per-pack lookup; a multi-pack index can avoid searching each pack separately. Remember A–E: the fetch slide uses the same history.
- Database analogy: immutable packs and repacking resemble SSTables and compaction. Unlike an SSTable's sorted key/value layout, the pack's objects need not be sorted by key; the separate .idx is sorted by object ID.

Builds:

1. **Every commit starts loose.** Each new object is its own zlib file under .git/objects, named by its hash: git add writes the blob, git commit writes the trees and the commit.
2. **gc rolls them into a pack.** Auto gc runs after commit or fetch once there are about 6,700 loose objects or 50 packs (or on a schedule with git maintenance). A fetch of 100+ objects is kept as a pack; a big push lands as one on the server. The id in the name is a checksum of the contents. D and E came later, so they are still loose.
3. **Inside the pack.** Pack entries contain zlib-compressed whole objects or deltas against a base. Newer-whole and older-delta is a heuristic, not a rule. The default delta-depth limit is 50 and is configurable. Physical layout need not follow commit order.
4. **The .idx finds things.** The .idx is sorted by hash: a 256-entry fanout narrows the search by the first byte, binary search finds the row, and the row gives the offset (plus a CRC to check the stored bytes), not a stored length. The pack's physical order is different from the index's hash order.
5. **Expand the pack entry.** At offset 12, read the entry header, then its zlib stream. This illustrative whole-commit entry declares type commit and an uncompressed size of 180 bytes, not 180 bytes to read from disk. Feed compressed bytes to zlib until it reports end-of-stream and check the output size. For a delta entry, the header size describes the uncompressed instructions; a base reference follows the header, and reconstructing the object also requires its base.

### 10 · How a delta is stored

07:50–08:50

- A delta is a recipe: COPY bytes from a base, INSERT changed bytes. In this SHA-1 example, src/ has two entries totaling 72 bytes. We choose E's tree as the whole base and encode D as copy bytes 0–51, then insert the old 20-byte hash: 25 bytes (2 for the sizes, 2 for COPY, 21 for INSERT).
- The 144 B versus 97 B comparison counts uncompressed tree payloads and delta instructions only. Both forms are zlib-compressed in a pack and need entry headers; a delta also needs a base reference. It is not a measured on-disk saving.
- Git chooses bases heuristically. Newer-whole is not a rule, and 50 is a configurable default depth, not a format limit. Reading a delta reconstructs its base even when that base is another delta.
- Database analogy: reconstruction resembles an MVCC undo chain, but Git's delta direction is chosen for compression, not transaction order.
- If asked why 52: bytes 0–34 are "100644 pack.ts\0" plus pack.ts's hash (6 + 1 + 7 + 1 + 20 = 35 B); bytes 35–51 are "100644 server.ts\0" (6 + 1 + 9 + 1 = 17 B); bytes 52–71 are server.ts's hash (20 B).

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

1. **1 · Connect, server advertises refs.** The client goes first: it connects over ssh or HTTPS and asks the server to run receive-pack. From then on the server speaks first, telling the client where every ref points right now.
2. **2 · Client builds the pack.** The client committed F on top of E. On push the client is the sender, so it builds: only objects reachable from F but not from E (F, its new trees, the changed blob), possibly as thin deltas against E's objects.
3. **3 · Commands + pack.** Each command is old, new, ref: here E F refs/heads/main. The pack follows on the same stream.
4. **4 · Server resolves in quarantine.** Now the server is the receiver, so it resolves: index-pack rebuilds every delta, SHA-1s each object, fills thin bases from E's objects, and writes the .idx, all inside a quarantine directory. Then the connectivity check on F. If anything fails, the quarantine is deleted and nothing becomes visible.
5. **5 · Ref transaction.** pre-receive can reject. Then lock main, check it still points at E, write F, commit. If someone else pushed G first, main is no longer E and this push is rejected. This is compare-and-swap.
6. **6 · Report status.** The client gets per-ref ok or ng, then post-receive fires (webhooks, CI).

### 14 · Part 3: How do you serve it to everyone?

13:00–13:20

- Git has no required central server. Hosting makes one copy authoritative by convention.
- A push separates immutable object data from ref updates. The data can be copied without ordering competing ref updates, but must be durable before refs publish it. The expected old ref values must still match; atomic multi-ref pushes require --atomic.
- Every stage answers the same two questions: where do packs live, and who decides which push wins? These are design choices, not a universal timeline.

### 15 · Stage 0: one box

13:20–14:00

- The simplest Git server is one machine: sshd or git-daemon runs upload-pack and receive-pack against bare repos on local disk.
- Packs and refs both live on that disk, and the box agrees with itself.
- It is fast and simple until the disk, the CPU, or the machine runs out.

### 16 · Stage 1: RAID + DRBD pairs

14:00–15:05 · 3 builds

- RAID 10 (Redundant Array of Independent Disks) mirrors disks in pairs inside one server; DRBD (Distributed Replicated Block Device) mirrors every disk write over the network to a second server. GitHub's pre-2016 design: an active fileserver with RAID, mirrored block-by-block to a dedicated hot spare with DRBD.
- Packs and refs both live on the active disk; DRBD copies bytes without knowing they are Git.
- Failover was manual, the spare served nothing, and a pair outage took every repo on it offline.

Builds:

1. **Active + hot spare.** RAID inside each server, DRBD mirrors every block to a dedicated spare. Four copies of every byte.
2. **Failover.** Active dies. A human confirms, promotes the spare, traffic moves. Now the spare has no spare.
3. **The limits.** Half the hardware idles, one server serves every read for its repos, and losing the pair takes them all offline.

### 17 · Stage 2: Quorum-based Git

15:05–16:30 · 3 builds

- This diagram is GitHub DGit, later Spokes: real Git repos on multiple local disks, with 3 replicas at launch and 5 for most repos in GitHub's 2026 account.
- Spokes fans out object data, then uses three-phase commit for the ref transaction. Healthy replicas can serve reads. GitLab Gitaly Cluster and Bitbucket Mesh also replicate repositories, but do not assume they use this exact protocol or have identical guarantees.
- The costs include push latency, a minimum replica count even for idle repos, and ongoing checksums and replica repair. Adding read replicas also increases coordination work on writes.

Builds:

1. **Fan out the pack.** Packs are idempotent, so send them to every replica in parallel: 3 at launch, 5 for most repos today. No agreement needed yet.
2. **3PC on the ref transaction.** Only the tiny ref update needs agreement. Phase 1 asks every replica: can you apply these ref updates? Each runs Git's own ref transaction: lock the refs, check the old values. Phase 2: a majority said yes, so get ready. Phase 3: commit, or roll back everywhere. Spokes then compares a checksum of every replica's refs; one that disagrees is marked unhealthy and repaired. About four round trips per push.
3. **Reads go to any replica.** Any replica can serve a fetch; a routing DB says which servers hold each repo. Say the cost out loud: every push waits for the slowest replica, even idle repos need 3+ copies, adding replicas slows pushes, and the replicas need checksums, repair jobs and that routing DB.

### 18 · Stage 3: log-based Git storage

16:30–18:55 · 6 builds

- Start with Cursor's Continuity: durable object data and WAL entries live in object storage. A successful conditional write on the WAL index publishes a push. Local Git repos are disposable caches, not the durable truth.
- The index fields shown follow walgit's concrete implementation: version, log head, live packs and checkpoint pointer. Current refs are derived from checkpoint plus log. This is not a diagram of Rubicon's storage format.
- Any node can attempt a push; the preferred writer reduces retries. Reads validate the index before serving. There is no Git-server quorum here, but correctness still depends on the object store's strongly consistent conditional writes. The publication point is not the only mutable object: leases and other controls also change.
- Spoken aside: Cursor publishes pushes through one shared index per repository. GitHub uses a separate Ref-CAS quorum service for refs and the pack manifest, allowing updates to different refs in parallel. Both store the objects before updating refs.
- If asked about throughput: Cursor reports 120 pushes/s on S3 Standard and 300+ on Express; GitHub reports 7,200 in separate internal Rubicon tests. Different workloads, not a head-to-head benchmark, and not walgit measurements.

Builds:

1. **1 · Read the index first.** Before advertising refs to the pusher, the node does the same conditional GET as any read. Now it knows the log is at version e1 with head 41, so this push will be entry #42. The version is the object's ETag on S3 or its generation on GCS.
2. **2 · Persist the push as WAL entry #42.** Persist the new object data and log entry in object storage. Those uploads alone do not publish the push; the index still names the previous state.
3. **3 · Publish with compare-and-swap.** Check main's expected old value against the validated state, then PUT the index with If-Match: e1. Success publishes e2: the push is acknowledged and linearized. A failed CAS requires re-reading and re-checking.
4. **4 · Gossip is best effort.** Node 2 receives the hint, reads the index and catches up to e2 before the next fetch. Receiving gossip alone does not make its cache current. Node 3 misses the message and stays cold. Correctness does not depend on gossip delivery; every fetch still checks the authoritative index.
5. **5 · Concurrent fetches check independently.** Nodes 2 and 3 now handle concurrent fetches. Node 2 has caught up to e2 and sends If-None-Match: e2; with the index unchanged, it gets 304 and serves its warm cache. Cold node 3 has no cached index version, so its unconditional GET returns 200 with the index. It reconstructs refs from the checkpoint and log, fetching packs as needed before serving. A stale warm node would also get 200 and catch up first. Cursor reports under 10 ms on average for the metadata check, not for the whole fetch.
6. **6 · A push loses the race.** Any node can attempt a push. Node 2 read e2, but another push publishes first, so its conditional write fails with 412. Unpublished uploads do not change the committed repository. It re-reads the index and re-checks every ref's old value: reject if a required old value changed, otherwise retry against the new version. The preferred writer reduces contention but is not required for correctness.

### 19 · Part 4: A Git client in the cloud?

18:55–19:05

- Hosting handles durable repository storage. Workflows still need somewhere to run Git.
- Git client as a service: GitFarm manages clones and warm checkouts, and runs Git commands for agents, bots and services.

### 20 · The cost of a local checkout

19:05–19:50

- This baseline is Uber's Go monorepo in the GitFarm paper: roughly 15 minutes, 4 cores, 16 GB of RAM and 40+ GB of disk for a cold clone. It is not a universal Git or agent cost.
- Short Git operations can be dominated by preparing the checkout. Cached, shallow, partial or sparse approaches may help; they are not the baseline shown here.
- Each automation service maintaining its own checkouts multiplies client resources, synchronization work and upstream load.

### 21 · Warm checkouts: remote devboxes and GitFarm

19:50–20:30

- A remote devbox (or devpod) is a dev machine in the data center that already has the monorepo checked out and kept in sync in the background. A developer connects with their IDE or shell and starts working: no 15-minute clone.
- GitFarm does the same for automation. Each node keeps a bare clone fresh from push events plus a periodic fetch, and a pool of warm checkouts.
- An agent, bot or service borrows one checkout per request, runs its git commands, and the checkout goes back to the pool.

### 22 · GitFarm: Git Client as a service

20:30–21:25

- Clients call a gRPC API. The gateway authenticates, authorizes per repo, and places the request.
- Every node runs three containers: backend serves the request; sandbox keeps a pool of sandboxes and executes the commands; repo-manager keeps the bare clone, synced by push events plus a 5-minute fetch, and a pool of pre-warmed checkouts.
- Separate clusters per heavy use case, plus a shared cluster for everyone else.

### 23 · One request, end to end

21:25–22:20 · 5 builds

- Request, authenticate, place on the node with the most free checkouts, or throttle.
- The backend takes a sandbox from the sandbox container's pool and a checkout from repo-manager's pool, mounts one into the other, and runs the commands as the caller.
- Pools are the whole trick: cold start moves off the request path.
- A request can be a session: several commands run in order on one repo state (the paper's example is merge-base, rev-parse, push), and the push goes through the same receive-pack and ref compare-and-swap from Part 2.

Builds:

1. **Request.** Client sends repo id, workspace type, and a list of commands.
2. **Authenticate + authorize.** Gateway identifies the caller and checks they may access this repo.
3. **Place.** Gateway picks the node with the most free checkouts for that repo, marks one occupied, and hands the request to that node's backend. None free → throttle.
4. **Acquire sandbox + checkout.** The backend first asks repo-manager for a pre-warmed checkout from its pool, then mounts it in one pre-started sandbox borrowed from the node's sandbox pool. The overlapping boxes show multiple sandboxes; only the front one serves this request.
5. **Execute as caller.** Commands run inside the sandbox with the caller's identity, not the service's. Fetches and pushes go from the sandbox straight to upstream; repo-manager also fetches from upstream to keep its bare clone fresh. When the commands finish, the backend releases the sandbox and the checkout back to their pools.

### 24 · Identity, freshness, isolation

22:20–23:05

- Identity: the gateway checks the caller's access to the repo, and the sandbox runs with the caller's privileges, not the client host's.
- Freshness: push events and a fetch every 5 minutes refresh the bare clones. A session can fetch to refresh refs, but this neither updates every working tree nor freezes upstream for the whole session. Resolve the intended commit and explicitly use it when the workflow needs a stable input.
- Checkout freshness is separate from the upstream host's storage consistency; GitFarm does not weaken the host's write guarantees. Sessions are capped at 5 minutes in the paper, which matters for long-running agents.
- Isolation: heavy workloads get dedicated clusters; a shared cluster serves the rest. Pools and limits are sized per use case.

### 25 · Where GitFarm fits

23:05–23:50

- Agent exploration and research read code, diffs and history. Approval checks inspect changed paths and ownership rules to determine required reviewers; they do not need to modify Git refs.
- Replication and mirroring synchronize objects and refs; a merge queue prepares integration commits and advances the target branch.
- These are workflow examples, not a claim that every workflow has been deployed or benchmarked. All benefit from running Git near a warm repository instead of managing a clone per client.

### 26 · Git stayed distributed. Hosting evolved.

23:50–24:15

- Git's distributed object model stayed intact while hosting changed where durable data lives and how competing updates are ordered. These are design choices, not a mandatory upgrade path for every host.
- GitFarm is a complementary execution layer, not a successor storage architecture. It can borrow warm checkouts from an upstream host regardless of that host's implementation.
- The thread is immutable object data and coordinated ref updates; automation increases pressure on both hosting and client execution.

### 27 · Thank you

24:15–24:25

- Thank you.
- Open the floor: questions on Git internals, the server designs, or GitFarm.
- If asked for reading: the GitFarm paper, Cursor's post, walgit, and Git's own docs on pack format and protocol.
