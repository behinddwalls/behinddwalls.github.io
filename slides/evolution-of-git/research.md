# Research notes and sources

## Story

The thread through the talk is a single observation about pushes:

- A push with new objects carries a **packfile**: immutable object data that can be copied before publication.
- A push also carries **ref updates**: small, mutable names with expected old values. An all-or-nothing multi-ref push requires `--atomic`.

Each server generation answers the same two questions: *where do packs live, and who decides which push wins?* GitFarm then moves *execution* to the repository.

1. Git's origin (2005) and its lack of a required central server. A full clone carries reachable history for fetched refs; shallow and partial clones differ.
2. The data model:
   - History uses four object types (blob, tree, commit, tag). The object ID hashes the type, size, NUL separator and content before compression. Examples use SHA-1; SHA-256 repos use the same object model. Contents are immutable, but repacking changes physical storage and GC can delete objects. Refs publish history; other mutable metadata includes the index and reflogs.
   - Blobs and trees are content-addressed (`sha1("blob 6\0hello\n")` = `ce0136…`, checked with `git hash-object`).
   - Commits are snapshots that share structure.
   - A loose direct SHA-1 ref is a 41-byte file; symbolic refs, packed refs and reftable use other representations. HEAD is usually symbolic, but can be detached. Tags are conventionally stable, not inherently immutable. GC considers refs, reflogs, the index and retention rules.
   - Packfiles hold delta chains and have an `.idx` index.
   - Where objects live: local edits can write loose objects; gc, repack, or a large fetch or push writes a packfile named by a checksum of its contents, with a `.idx` (hash → offset) sharing the name. Identical contents share an object ID; physical copies can still exist in multiple packs or replicas. The deck uses one A–E history: A–C packed, D and E loose, client at B.
   - Deltas: pack entries contain zlib-compressed whole objects or COPY/INSERT recipes against a base. The default candidate window is 10 and depth limit is 50; both are configurable (maximum supported depth 4095). Newer-whole/older-delta is a heuristic, not a rule. The illustrated 72-byte SHA-1 tree becomes a 25-byte delta (2 B sizes + 2 B COPY + 21 B INSERT). The 144 B versus 97 B comparison excludes zlib compression, entry headers and delta-base references; it is not an on-disk measurement. A delta chain resembles MVCC reconstruction, but base selection is not chronological.
   - The `.idx` is sorted by hash: a 256-entry fanout, then binary search, then offset, with a CRC32 per entry so reused bytes can be checked without inflating. The `.pack` is in write order.
   - The `.idx` does not store entry lengths. Slide 9 expands an illustrative whole-commit entry at offset 12: the entry header identifies its type and uncompressed size (180 B in the example), followed by a zlib stream. The inflater signals the stream's end. For delta entries, the header size is the uncompressed instruction size, not the reconstructed object size; a base reference precedes the compressed instructions.
   - One example repo runs through Parts 1–2: files README.md, src/pack.ts, src/server.ts; commits A→E on main (B 5ac0, C 3f7d, D 9d2a, E c8f3); D and E roots e8c4/f021, src/ 7a19/6c81, server.ts d431/f7a9; A–C packed, D–E loose; the client pushes F.
   - Refs at scale (background for Q&A): annotated tags point through tag objects. Updating a branch stored in `packed-refs` writes a loose override; deleting a packed ref rewrites the packed file. Reftable uses sorted binary tables; updates add tables and compaction merges them (Git 2.45+ `--ref-format=reftable`). Protocol v0 advertises visible refs up front; v2 `ls-refs` supports `ref-prefix`. Hosts can hide internal refs. Unreachable reflog entries default to 30-day expiry, other entries to 90 days; pruning has its own age cutoff, not an additive guaranteed wait.
   - Planned Git 3.0 defaults: SHA-256 and reftable for newly initialized repositories. No release date; existing repositories do not automatically migrate.
   - want/have: a want is the tip of each ref the client lacks; haves are the client's tips, then older commits in batches, until the server ACKs a common one.
3. The wire:
   - fetch-pack talks to upload-pack: ls-refs, then want/have negotiation, then the reachability walk, then pack-objects over sideband, then index-pack and the connectivity check.
   - send-pack talks to receive-pack: ref advertisement, then the commands and the pack, then quarantine, the connectivity check, pre-receive, the ref transaction, post-receive, and report-status.
4. Server evolution: one box → RAID + DRBD pairs → Spokes (3PC on the ref transaction) → log-based storage. A spoken aside contrasts Cursor's repository-wide object-store publication point with Rubicon's per-ref quorum service. These are design choices, not a universal timeline. Detailed walgit cache behavior, Entire, JGit DFS and other database-backed patterns below are background, not additional stages in the main deck.
5. GitFarm: architecture, request flow, sessions, design choices, and four concrete use cases presented as a simple list. Detailed production measurements and evidence distinctions below are background, not labels on the slide. Client execution complements upstream hosting; it does not replace the storage layer.

## Figures and how to read them

### GitFarm (arXiv 2604.11977)

**Problem and evaluation setup**

- Uber's Go monorepo cold clone: about 15 minutes, 4 cores, 16 GB of RAM, and 40+ GB of disk. Not a universal Git or agent baseline.
- A service node that syncs several monorepos needs 12 cores, 48 GB of RAM, and 64 GB of disk.
- Evaluation cluster: 4 backend nodes, each with 48 cores, 250 GB of RAM, and 1.5 TB of SSD.
- Pool sizes per node: 100 sandboxes, 30 Go checkouts, and 10 checkouts for each other monorepo.

**Startup latency.** Keep these two figures separate:

- Under 1 s: acquiring a ready checkout from the pool.
- Under 1 min: starting a client service.
- Unpooled for comparison: a sandbox takes 1–2 s to start, and a checkout takes up to 3 min to materialize from the bare clone.

**Production cases**

- *Compliance auditing.* About 10–20k events per hour across about 9,000 repositories. Git-work p50 dropped from 110–160 s on Buildkite to 20–30 s. This excludes PR creation; no p99 is reported.
- *PR base-branch change.* About 1 request/s with a p50 of about 25 s (fetch about 5 s, push about 12 s). This case has **no baseline**.
- *Code-ownership inspection.* About 100 requests/s over 350K+ directories.
  - Per node: 12 → 2 cores, 48 → 4 GB of RAM, and 64 → 2 GB of disk.
  - Client fleet: 90.1 → 16 cores (−82%) and about 472 → 32 GB of RAM (−93%). These exclude GitFarm's backend cost, so they are not net whole-system savings.
  - Node startup: 15–20 min → under 1 min.

**Other claims and limits**

- Headline: client-side resource use is down more than 70%. In production since early 2025, with Git invoked millions of times a day.
- Freshness: checkout caches can lag upstream. Bare clones are updated by push events plus a fetch every 5 minutes, and a session can fetch to refresh refs. Fetch does not freeze upstream for a session or automatically update every working tree; choose a specific commit for stable inputs. This is separate from the upstream host's storage consistency.
- Sessions are capped at 5 minutes.
- Future work: streaming output, sparse workspaces, longer sessions, SubmitQueue merge-to-trunk, and upstream mirroring.
- Agent use cases, AI review, scanning and analytics are **potential uses**, not measured results. Under 1 s means acquiring a ready checkout, not an entire agent session or workflow.

### Cursor, "Git at any scale" (Vicent Martí, Aug 2026)

- Covers the JGit DHT at Google; NFS, GFS, and DRBD attempts at GitHub; Spokes using 3PC on the ref transaction; and the "floor too high, ceiling too low" critique.
- Describes Azure DevOps as storing packfiles in blob storage and refs in SQL Server.
- Describes Continuity:
  - A WAL in S3, with linearizable pushes published through a CAS on the WAL index.
  - Best-effort UDP gossip, plus a conditional GET version check taking under 10 ms on average. This is metadata-check latency, not end-to-end fetch latency. The diagram separates gossip from fetches: node 2 receives the hint and catches up before its conditional GET returns 304; cold node 3 misses gossip and receives 200 on an unconditional GET. Gossip receipt alone does not establish freshness.
  - No primary role: any node can accept a push (the CAS on the WAL index in the object store decides; a node that loses the race re-reads and retries) and any node can serve a read, rebuilding from the WAL on demand. Rendezvous hashing only picks a preferred node, which avoids CAS retries and does compaction.
- No Git-server quorum is required, but agreement still depends on the object store's strongly consistent conditional writes.
- Numbers: 120 pushes/s on S3 Standard, 300+ pushes/s on S3 Express One Zone, and reads that scale linearly up to 100 replicas in synthetic tests. These are **Cursor's own claims**, not walgit measurements or a head-to-head comparison with Rubicon.

### GitHub, "Introducing DGit" (2016)

- Before DGit, each active fileserver had a dedicated hot spare. RAID plus DRBD gave four copies of every disk.
- Failover was manual.
- DGit replicates at the application layer and keeps three copies. It was later renamed Spokes.
- The deck's 3PC diagram is specifically Spokes. GitLab Gitaly Cluster and Bitbucket Mesh are related application-level replication designs, not claims of identical protocols. GitLab documents primary routing, asynchronous replication and selective strong consistency.

### walgit (github.com/tobi/walgit)

- An open-source Rust implementation inspired by Cursor's Continuity design; the main WAL diagram uses its manifest fields.
- Per repo under `repos/<owner>/<repo>/`:
  - `manifest.pb`: rewritten by CAS. It holds the head sequence, live pack set, checkpoint pointer and settings. This is the publication point; its size depends in part on the pack inventory, so it is not a fixed-size object.
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
- Any instance may accept a push. A cold replica reconstructs refs from checkpoint plus log. Per-read validation uses a conditional GET; an optional freshness TTL trades some staleness for fewer checks. Compaction is done by the lease holder and published into the log.

### walgit with many refs

- The manifest holds no ref values. Log entries record changed refs rather than rewriting the full ref snapshot for each push. This is a metadata-design property, not a guarantee of constant manifest size or total push cost.
- Each PUSH / REF_UPDATE log entry carries its ref transaction plus a pack pointer. A checkpoint (`checkpoints/<seq>/refs.pb`) is a full `RefSnapshot`, rewritten when `wal.snapshot_every_entries`, `wal.checkpoint_interval` or `wal.checkpoint_tail_bytes` fires.
- Cold refs sync: one manifest GET, then the checkpoint refs and log tail segments in parallel, folded into `packed-refs`; no packs. Target: refs in under a second. walgit's reference monorepo has 466k refs; the docs give the timing as a target, not a measurement. Snapshot compression or sharding is not described.

### GitHub Rubicon (Nov 2026)

- Spokes stays as the router. Underneath: disposable "virtual replicas" (local disk is only a cache), packs and read accelerators in Azure Blob Storage, and authoritative refs plus the pack manifest in Ref-CAS, a replicated quorum log service. Git object contents stay in blob storage.
- Write: index the pack, upload it to blob storage, then one Ref-CAS transaction lists the pack and moves the refs. Same order as WAL-Git: durable first, visible second.
- Ref-CAS locks only the refs a transaction touches, allowing disjoint ref updates to proceed concurrently. Every committed transaction gets an LSN that versions the repo. Reads check their LSN. This is not Cursor's single WAL index moved to another service.
- GitHub first tried a WAL indexed by one blob (Cursor's pattern) and hit the mid-hundreds of writes/s; sharding refs across blobs made every read check every shard.
- Compaction and GC run on separate nodes that lease shards of packs; compacted output is verified to hold exactly the input objects.
- Numbers are GitHub's internal tests: 5,000 concurrent agents, 7,200 pushes/s to one repo, Ref-CAS alone in the tens of thousands of ops/s. The post does not describe the test setup (pack sizes, replica count, latency).
- Limited private preview; broader rollout early 2027.

### Bitbucket: from NFS to Mesh

- Bitbucket Data Center has kept Git repositories on a shared NFS file system ever since clustering was introduced.
- Atlassian's docs say NFS latency is fine for large operations like streaming a pack, but very costly for small ones like reading refs or loose objects.
- Bitbucket Mesh (Data Center 8.0, 2022) replaces NFS for Git storage. It moves Git processing next to local disks and replicates each repository across several Mesh nodes. Every replica serves both reads and writes.
- Git LFS objects still live on the shared home.

### Background: Entire — refs in a database, packs on replicas (Oct 2026)

- A cell is one PostgreSQL database holding refs (one row per ref, current value) plus a set of EntireDB nodes holding loose objects and packfiles on local disk. Each repo has one home cell; regional mirror cells serve reads.
- Push: the receiving node stages the pack and streams it to the repo's other nodes; at least two nodes in distinct availability zones must confirm staging, then promotion to durable storage. A node checks that every object the new refs reach is present, then one compare-and-swap on PostgreSQL makes the push visible.
- No WAL or push log is described; only the current ref value. The posts don't give a node count per cell, PostgreSQL failover, or whether nodes sit on blob storage.
- Entire's own benchmark (forgemark) reports 418 pushes/s; vendor numbers, not independently checked. EntireDB is to be open-sourced later.

### Background: blob + DB patterns and limits of the evidence

- **JGit DFS** stores standard pack and index files through a storage abstraction. It does not prescribe one ref backend or imply one SQL row per ref. It replaced the slower DHT layer and was built at Google for hosted Gerrit.
- **Azure DevOps** stores packs in blobs and refs in SQL Server (per the Cursor post). It runs Microsoft's own Git server, not JGit.
- **AWS CodeCommit** stores repositories in S3 and DynamoDB, according to AWS's own pages. AWS does not publish the exact split or transaction protocol; do not infer a specific per-ref database schema.

## Sources

- [Pro Git · A Short History of Git](https://git-scm.com/book/en/v2/Getting-Started-A-Short-History-of-Git)
- [Pro Git · Git on the Server: The Protocols](https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols)
- [git-clone](https://git-scm.com/docs/git-clone) and [git-fetch](https://git-scm.com/docs/git-fetch)
- [GitFarm: Git as a Service for Large-Scale Monorepos](https://arxiv.org/abs/2604.11977)
- [Uber Engineering · GitFarm as a Service](https://www.uber.com/blog/gitfarm-as-a-service/)
- [Cursor · Git at any scale](https://cursor.com/blog/git-at-any-scale)
- [GitHub · Introducing DGit](https://github.blog/engineering/architecture-optimization/introducing-dgit/)
- [GitHub · Stretching Spokes](https://github.blog/engineering/infrastructure/stretching-spokes/)
- [Pro Git · Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) and [Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
- [Git planned breaking changes](https://git-scm.com/docs/BreakingChanges), [packed refs](https://git-scm.com/docs/git-pack-refs), [garbage collection](https://git-scm.com/docs/git-gc), [delta selection](https://git-scm.com/docs/git-pack-objects)
- [gitformat-pack](https://git-scm.com/docs/gitformat-pack), [gitprotocol-v2](https://git-scm.com/docs/gitprotocol-v2), [gitprotocol-pack](https://git-scm.com/docs/gitprotocol-pack), [git-receive-pack](https://git-scm.com/docs/git-receive-pack)
- [JGit DFS storage](https://github.com/eclipse-jgit/jgit/tree/master/org.eclipse.jgit/src/org/eclipse/jgit/internal/storage/dfs)
- [Atlassian · Bitbucket Mesh](https://confluence.atlassian.com/display/BitbucketServer/Bitbucket+Mesh)
- [GitLab · Gitaly Cluster](https://docs.gitlab.com/administration/gitaly/praefect/)
- [AWS CodeCommit features](https://aws.amazon.com/codecommit/features/)
- [walgit](https://github.com/tobi/walgit)
- [GitHub · Inside GitHub's new Git storage architecture](https://github.blog/engineering/architecture-optimization/inside-githubs-new-git-storage-architecture/)
- [Entire · A Git Forge Without a Center](https://entire.com/blog/a-git-forge-without-a-center)
- [Git logo attribution and license](https://git-scm.com/community/logos): © Jason Long, [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/).
