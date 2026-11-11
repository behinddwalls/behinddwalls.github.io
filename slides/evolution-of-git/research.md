# Research notes and sources

## Story

The thread through the talk is a single observation about pushes:

- A push carries a **packfile**: large, immutable, content-addressed, idempotent.
- A push also carries a **ref transaction**: tiny, mutable, compare-and-swap.

Each server generation answers the same two questions: *where do packs live, and who agrees on refs?* GitFarm then moves *execution* to the repository.

1. Git's origin (2005) and its serverless design.
2. The data model:
   - Everything is one of four object types (blob, tree, commit, tag): `type + size + content`, zlib-compressed, named by its SHA-1, and immutable. Refs are the only mutable state.
   - Blobs and trees are content-addressed (`sha1("blob 6\0hello\n")` = `ce0136…`, checked with `git hash-object`).
   - Commits are snapshots that share structure.
   - Refs are 41-byte files, and reachability decides what a repository contains.
   - Packfiles hold delta chains and have an `.idx` index.
   - Where objects live: new objects are loose files; gc, repack, or a large fetch or push writes a packfile named by a checksum of its contents, with a `.idx` (hash → offset) sharing the name. The deck uses one A–E history: A–C packed, D and E loose, client at B.
   - Deltas: every object is zlib-compressed; blobs and trees can be COPY/INSERT deltas against a similar object, picked greedily within a window of about 10 and capped at depth 50. A tree delta for one changed file is about 28 bytes. 100 edits to a file stay 100 objects; packed, they become one or two whole bases plus small deltas.
   - The `.idx` is sorted by hash: a 256-entry fanout, then binary search, then offset, with a CRC32 per entry so reused bytes can be checked without inflating. The `.pack` is in write order.
   - One example repo runs through Parts 1–2: files README.md, src/pack.ts, src/server.ts; commits A→E on main (B 5ac0, C 3f7d, D 9d2a, E c8f3); D and E roots e8c4/f021, src/ 7a19/6c81, server.ts d431/f7a9; A–C packed, D–E loose; the client pushes F.
   - Refs at scale (background for Q&A; no longer its own slide): branches and tags are refs (annotated tags go through a tag object). Storage goes loose files → packed-refs (one sorted file; updating one ref rewrites it) → reftable (sorted binary blocks; updates append, compaction merges; Git 2.45+ `--ref-format=reftable`). Protocol v0 advertises every ref up front; v2 `ls-refs` takes `ref-prefix`. Hosts hide internal refs such as `refs/pull/*` (hideRefs). Every ref is a GC root.
   - want/have: a want is the tip of each ref the client lacks; haves are the client's tips, then older commits in batches, until the server ACKs a common one.
3. The wire:
   - fetch-pack talks to upload-pack: ls-refs, then want/have negotiation, then the reachability walk, then pack-objects over sideband, then index-pack and the connectivity check.
   - send-pack talks to receive-pack: ref advertisement, then the commands and the pack, then quarantine, the connectivity check, pre-receive, the ref transaction, post-receive, and report-status.
4. Server evolution: one box → RAID + DRBD pairs → Spokes (3PC on the ref transaction) → blob + DB → WAL-Git (object-store WAL + CAS index).
5. GitFarm: architecture, request flow, sessions, design choices, and use cases (results are in the speaker notes).

## Figures and how to read them

### GitFarm (arXiv 2604.11977)

**Problem and evaluation setup**

- Go monorepo clone: about 15 minutes, 4 cores, 16 GB of RAM, and 40+ GB of disk.
- A service node that syncs several monorepos needs 12 cores, 48 GB of RAM, and 64 GB of disk.
- Evaluation cluster: 4 backend nodes, each with 48 cores, 250 GB of RAM, and 1.5 TB of SSD.
- Pool sizes per node: 100 sandboxes, 30 Go checkouts, and 10 checkouts for each other monorepo.

**Startup latency.** Keep these two figures separate:

- Under 1 s: acquiring a ready checkout from the pool.
- Under 1 min: starting a client service.
- Unpooled for comparison: a sandbox takes 1–2 s to start, and a checkout takes up to 3 min to materialize from the bare clone.

**Production cases**

- *Compliance auditing.* About 10–20k events per hour across about 9,000 repositories. p50 dropped from 110–160 s on Buildkite to 20–30 s, a reduction of more than 80%. Only p50 is reported.
- *PR base-branch change.* About 1 request/s with a p50 of about 25 s (fetch about 5 s, push about 12 s). This case has **no baseline**.
- *Code-ownership inspection.* About 100 requests/s over 350K+ directories.
  - Per node: 12 → 2 cores, 48 → 4 GB of RAM, and 64 → 2 GB of disk.
  - Fleet: 90.1 → 16 cores (−82%) and about 472 → 32 GB of RAM (−93%).
  - Node startup: 15–20 min → under 1 min.

**Other claims and limits**

- Headline: client-side resource use is down more than 70%. In production since early 2025, with Git invoked millions of times a day.
- Consistency: eventually consistent with upstream. Bare clones are updated by push events plus a fetch every 5 minutes, and a session can fetch first if it needs the latest state.
- Sessions are capped at 5 minutes.
- Future work: streaming output, sparse workspaces, longer sessions, SubmitQueue merge-to-trunk, and upstream mirroring.
- Agent use cases and AI review are presented as **patterns or direction**, not measured results.

### Cursor, "Git at any scale" (Vicent Martí, Aug 2026)

- Covers the JGit DHT at Google; NFS, GFS, and DRBD attempts at GitHub; Spokes using 3PC on the ref transaction; and the "floor too high, ceiling too low" critique.
- Describes Azure DevOps as storing packfiles in blob storage and refs in SQL Server.
- Describes Continuity:
  - A WAL in S3, with linearizable pushes published through a CAS on the WAL index.
  - UDP gossip, plus a conditional GET that returns 304 in under 10 ms.
  - No primary role: any node can accept a push (the CAS on the WAL index in the object store decides; a node that loses the race re-reads and retries) and any node can serve a read, rebuilding from the WAL on demand. Rendezvous hashing only picks a preferred node, which avoids CAS retries and does compaction.
- Numbers: 120 pushes/s on S3 Standard, 300+ pushes/s on S3 Express One Zone, and reads that scale linearly up to 100 replicas in synthetic tests. These are **Cursor's own claims** for their monorepo.

### GitHub, "Introducing DGit" (2016)

- Before DGit, each active fileserver had a dedicated hot spare. RAID plus DRBD gave four copies of every disk.
- Failover was manual.
- DGit replicates at the application layer and keeps three copies. It was later renamed Spokes.

### walgit (github.com/tobi/walgit)

- An open-source Rust implementation of Cursor's Continuity design; the deck's "What a push writes to the object store" slide uses its layout and log format.
- Per repo under `repos/<owner>/<repo>/`:
  - `manifest.pb`: tiny, rewritten only by CAS. It holds the head sequence, live pack set, checkpoint pointer and settings. This is the linearization point.
  - `log/<seq>.pb`: immutable entries (PUSH, COMPACT, REF_UPDATE, CHECKPOINT, SETTINGS).
  - `wal/<checksum>.pack|.idx|.rev|.bitmap`: immutable, content-addressed packs.
  - `checkpoints/`: ref snapshot plus pack set.
  - `leases/`: CAS with a TTL, the only cross-instance mutex.
- A PUSH log entry carries `seq`, a `PackRef` (pack checksum, sizes, tier) and a `RefTransaction` (name, old oid, new oid per ref), plus writer and meta.
- Push path:
  1. `git index-pack --fix-thin --rev-index` in a scratch directory, plus connectivity and policy checks.
  2. Upload `pack ∥ idx ∥ log entry`.
  3. CAS the manifest.
  4. On a 412: re-read, re-validate every ref's old value, retry. Concurrent pushes on one instance are group-committed into one CAS.
  5. The client sees `ok` only after the bucket does.
- No leader: any instance may accept a push; a cold replica reads the log; every read revalidates with a conditional GET. Compaction is done by the lease holder and published into the log.

### GitHub Rubicon (Nov 2026)

- Spokes stays as the router. Underneath: disposable "virtual replicas" (local disk is only a cache), packs and read accelerators in Azure Blob Storage, and refs plus the pack manifest in Ref-CAS, a replicated quorum log service that stores only pointers.
- Write: index the pack, upload it to blob storage, then one Ref-CAS transaction lists the pack and moves the refs. Same order as WAL-Git: durable first, visible second.
- The CAS differs: Ref-CAS locks only the refs a transaction touches, and every commit gets an LSN that is the repo version. Reads send their LSN; an unchanged repo costs one comparison.
- GitHub first tried a WAL indexed by one blob (Cursor's pattern) and hit the mid-hundreds of writes/s; sharding refs across blobs made every read check every shard.
- Compaction and GC run on separate nodes that lease shards of packs; compacted output is verified to hold exactly the input objects.
- Numbers are GitHub's internal tests: 5,000 concurrent agents, 7,200 pushes/s to one repo, Ref-CAS alone in the tens of thousands of ops/s. The post does not describe the test setup (pack sizes, replica count, latency).
- Limited private preview; broader rollout early 2027.

### Bitbucket: from NFS to Mesh

- Bitbucket Data Center has kept Git repositories on a shared NFS file system ever since clustering was introduced.
- Atlassian's docs say NFS latency is fine for large operations like streaming a pack, but very costly for small ones like reading refs or loose objects.
- Bitbucket Mesh (Data Center 8.0, 2022) replaces NFS for Git storage. It moves Git processing next to local disks and replicates each repository across several Mesh nodes. Every replica serves both reads and writes.
- Git LFS objects still live on the shared home.

### Blob + DB pattern

- **JGit DFS** stores standard pack and index files on an abstract DFS. It replaced the slower DHT layer and was built at Google for hosted Gerrit.
- **Azure DevOps** stores packs in blobs and refs in SQL Server (per the Cursor post). It runs Microsoft's own Git server, not JGit.
- **AWS CodeCommit** stores repositories in S3 and DynamoDB, according to AWS's own pages. AWS does not publish how objects and refs are split between them, so the slide treats this as the same *pattern*.

## Sources

- [GitFarm: Git as a Service for Large-Scale Monorepos](https://arxiv.org/abs/2604.11977)
- [Uber Engineering · GitFarm as a Service](https://www.uber.com/blog/gitfarm-as-a-service/)
- [Cursor · Git at any scale](https://cursor.com/blog/git-at-any-scale)
- [GitHub · Introducing DGit](https://github.blog/engineering/architecture-optimization/introducing-dgit/)
- [GitHub · Stretching Spokes](https://github.blog/engineering/infrastructure/stretching-spokes/)
- [Pro Git · Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) and [Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
- [gitformat-pack](https://git-scm.com/docs/gitformat-pack), [gitprotocol-v2](https://git-scm.com/docs/gitprotocol-v2), [gitprotocol-pack](https://git-scm.com/docs/gitprotocol-pack), [git-receive-pack](https://git-scm.com/docs/git-receive-pack)
- [JGit DFS storage](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/internal/storage/dfs)
- [Atlassian · Bitbucket Mesh](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)
- [AWS CodeCommit features](https://aws.amazon.com/codecommit/features/)
- [walgit](https://github.com/tobi/walgit)
- [GitHub · Inside GitHub's new Git storage architecture](https://github.blog/engineering/architecture-optimization/inside-githubs-new-git-storage-architecture/)
