# Jellyfin media access: host-path bind mount vs SMB/CIFS

> Resolves wayfinder ticket #16. Compiled 2026-08-07 via `/research` probe.
> Scope: how should Jellyfin (a Dockhand/Compose container on TrueNAS SCALE
> 25.10) consume its media library, given that the media lives on the **same
> ZFS pool** the container runs on, and SMB exports of that pool already exist?
> Research only — no host access, no config edits. Feeds topology decision
> **#13**.

## TL;DR

- **Jellyfin's own stance (point 1):** Network storage is **supported but must
  be mounted at the OS level** — "A network storage device that is using samba
  or NFS must be directly mounted to the OS." The **database must be local**
  ("not on a network storage device"). The only network-mount caveat Jellyfin
  calls out is an **NFSv3 locking** interaction with .NET; SMB has no equivalent
  warning. So SMB/NFS is *conditional*, not *discouraged* — but "conditional"
  is doing no work in our topology because the data is already local.
- **Loopback-SMB overhead (point 2):** Mounting the host's own SMB export back
  into the container adds ZFS → Samba daemon → TCP/IP loopback → CIFS client →
  VFS → Jellyfin, versus ZFS → VFS → Jellyfin for a direct bind mount. Loopback
  never hits the NIC, but it pays for protocol marshalling, extra
  user↔kernel context switches, and per-packet TCP processing. **This is pure
  overhead for zero benefit on a same-box topology.** The only scenario where
  loopback SMB is justified is when the SMB *server* is a different host than
  the SMB *client* — i.e. not ours.
- **Streaming performance (point 3):** For large sequential reads, even
  loopback SMB saturates well beyond what 4K direct play needs (4K HEVC
  remux ≈ 60–80 Mbps; loopback SMB does hundreds of Mbps on modern hardware).
  **Direct play is not the bottleneck either way.** Where local access pulls
  ahead is **transcode source reads** (parallel ffmpeg reads at high bitrate)
  and **seek/scrub latency** (random access, range requests): both hit SMB's
  per-op overhead and small-read semantics harder than a native read. ZFS
  dataset tuning (1M recordsize for media) applies identically to both paths
  — it's the access path that differs.
- **Portability vs locality (point 4):** SMB's only real argument is
  host-independence — Jellyfin could relocate and keep seeing the same library.
  Against that: an extra failure surface (smbd, mount-on-boot ordering,
  credentials, stale handles) and the overhead above. For a homelab where
  Jellyfin and the ZFS pool are **co-located and not expected to move**,
  **locality wins.** Portability is a speculative future need; the overhead
  and fragility are present-tense, certain costs.
- **Mount mechanics if SMB were chosen (point 5):** Two options. (a) Mount on
  the **host**, then bind-mount the host path into the container — keeps the
  container unprivileged but re-introduces all the loopback overhead above
  *plus* a host-side fstab/systemd-mount dependency. (b) Mount **inside** the
  container via `mount.cifs` — requires `CAP_SYS_ADMIN` (and in practice
  `CAP_DAC_READ_SEARCH` too, per Docker forum reports) or full `--privileged`
  mode, which **directly violates the repo's unprivileged-container posture**
  (caddy/dashy carry no capabilities). Option (b) is a non-starter for us;
  option (a) is the only SMB form that would be acceptable, and it's still the
  worse choice on the merits.
- **Recommendation for #13 (point 6):** **Host-path bind mount**
  (`/mnt/puddle/<dataset>/media:/media:rw` or equivalent). One-line rationale:
  *the data is already on the same box as the container, so the only thing a
  network mount adds is overhead and a failure surface — and it would force a
  privileged container to boot.* This matches the established repo precedent
  (caddy and dashy both bind-mount co-located files). **Confidence: high.**

---

## Point 1 — Jellyfin's official guidance on network-mounted libraries

The canonical source is the **Storage** page of the Jellyfin docs. The load-bearing
sentences, quoted verbatim:

> "Jellyfin is designed to directly read media from the filesystem. A network
> storage device that is using samba or NFS must be **directly mounted to the
> OS**."
> — [jellyfin.org/docs/general/administration/storage](https://jellyfin.org/docs/general/administration/storage/)

> "The Jellyfin database should also be **stored locally and not on a network
> storage device**."

Implications:

- SMB/NFS is **supported** (Jellyfin can read from it) but **conditional** on
  the share being an OS-level mount — Jellyfin does **not** traverse UNC /
  `smb://` / `nfs://` paths itself. "Shared network folder" in the add-library
  dialog exists but is for the rare OS that surfaces network paths that way;
  the documented path is an OS mount presented as a local directory.
- The **database** (SQLite, the metadata index) is **non-negotiably local**.
  This matters for our topology: even if someone chose SMB for media, the
  Jellyfin data dir (config/cache/db) must be a bind mount or named volume on
  local storage.
- The only network-specific **performance/stability warning** Jellyfin raises
  on this page concerns **NFSv3 + .NET file locking**:

  > If playback on NFSv3 has long startup times it may be due to .NET locking
  > without NFSv3 having locking enabled.

  Workarounds offered: `DOTNET_SYSTEM_IO_DISABLEFILELOCKING`, the `nolock`
  mount option, enabling the lock daemon, or using NFSv4. **There is no
  equivalent SMB-specific warning** in the official docs — SMB is treated as a
  generic "mount it at the OS level" case.

- A separate caution worth carrying into any access-method choice: scheduled
  library-maintenance tasks "remove items from your library if triggered while
  your media storage is unavailable." A network mount that flakes (smbd
  restart, stale handle) is a *content-loss* risk; a local bind mount isn't.

<details>
<summary><b>What this means for our topology</b></summary>

"Conditional, not discouraged" sounds like a green light for SMB. But read
carefully: the condition is **"must be OS-mounted."** Our media is *already*
on the OS's filesystem (the same ZFS pool the container runs on). There is
nothing for an SMB mount to give us that the kernel doesn't already expose via
VFS. The Jellyfin guidance is silent on the **loopback** SMB case — it
addresses remote network storage, where mounting is *necessary* to get the
bytes local. Our case is the opposite: the bytes are already local. The
guidance therefore does not advocate SMB for us; it merely permits SMB for
people whose media is elsewhere.

</details>

---

## Point 2 — The loopback-SMB overhead

**Data path comparison** (both reach the same ZFS blocks):

```
Bind mount:       ZFS ARC/pool → VFS → container namespace → Jellyfin
Loopback SMB:     ZFS ARC/pool → VFS → smbd (userland)
                                    → SMB protocol encode
                                    → TCP loopback (kernel, no NIC)
                                    → CIFS client (kernel)
                                    → VFS → container namespace → Jellyfin
```

The loopback hop never touches a physical NIC — the kernel short-circuits
127.0.0.1 traffic in software (see the ServerFault/"how fast is 127.0.0.1"
discussion). So the cost is **not** network bandwidth. The cost is:

- **Protocol marshalling.** Every read becomes an SMB request/response pair:
  oplock negotiation, lease semantics, credit-based flow control, signature
  checks (if enabled). None of that exists for a VFS read.
- **Context switches.** smbd is a userland process. Each SMB op crosses
  user→kernel→user at the server side, and the client crosses the same
  boundary. A direct read is a single syscall.
- **Per-op overhead, not throughput overhead.** Loopback SMB has plenty of
  *bandwidth* for sequential streaming (hundreds of Mbps on modern hardware).
  Its penalty is **per-operation latency and CPU**, which shows up in random
  access, metadata stat() storms (library scans), and many small reads —
  exactly the workload of seeking/scrubbing and of ffmpeg reading transcode
  sources in chunks.
- **Extra moving parts.** smbd must be up before the mount is usable; a smbd
  restart can invalidate handles; credentials must be supplied at mount time.

The Stack Overflow analysis of the analogous Windows localhost-SMB case makes
the same point: the localhost hop adds "significant overhead in terms of disk
access and access times" relative to direct local access, and the recommended
fix is "don't — access the local filesystem directly" (Apple Stack Exchange
similarly discourages `smb://localhost/share`).

**Is loopback SMB ever justified?** Only when the data is genuinely remote from
the *client* and the loopback is incidental — e.g. a Windows box mounting a
share served from itself for app-compatibility reasons. In our case the client
(container) and the server (ZFS pool) are the **same physical host**, and the
container has a direct VFS path available. There is no justification; the
loopback is pure waste.

<details>
<summary><b>Why loopback ≠ free just because there's no NIC</b></summary>

A common intuition is "loopback is just memory copies, so it's ~free." That's
true for *raw TCP throughput* (a `iperf3` over loopback will peg CPU before it
saturates). But SMB-over-loopback pays on three axes that raw TCP doesn't:

1. **SMB protocol layer** sits on top of the TCP layer. Request framing, oplock
   state, signing, lease management — all happen regardless of whether the
   transport is loopback or wire.
2. **smbd is userland.** The server side context-switches on every op. A direct
   read is one kernel entry. An SMB read is: client syscall → kernel → TCP →
   kernel → smbd userland → kernel (to read the file) → back out the same path.
3. **Cache effectiveness.** ZFS ARC caches blocks. A direct read hits ARC
   directly. An SMB read also hits ARC (Samba doesn't re-cache the file
   contents in a way that helps), but the *page cache* on the client side and
   the *oplock* state on the server side add layers that can mask or duplicate
   cache hits. For streaming this is neutral; for seek-heavy access it adds
   latency.

So: throughput-bound workload (direct play) — loopback SMB is fine but still
slower than direct. Latency-bound workload (seeking, scans, transcode random
reads) — loopback SMB is measurably worse. Either way it is never *better* than
the direct path that's already available.

</details>

---

## Point 3 — Performance for streaming workloads

Three sub-cases, each with a different sensitivity profile:

**4K direct play (direct stream, no transcode).** This is a single sequential
read at the file's bitrate. 4K HEVC remux peaks around **60–80 Mbps** (~8–10
MB/s); 4K web-dl is lower. Loopback SMB on modern hardware delivers multiples
of that. **Direct play is not the bottleneck under either access method.** The
ticket's concern here is largely hypothetical at homelab scale: if your storage
can't sustain 10 MB/s sequential, you have a pool problem, not an
access-method problem.

**Transcode source reads.** ffmpeg reads the source (often a high-bitrate
remux, 50–60 GB) while writing the transcode to the transcode dir. Two
differences from direct play: (a) reads can be **non-sequential** when ffmpeg
seeks for stream analysis, GOP boundaries, or subtitle extraction; (b) on a
busy box, multiple transcodes run in parallel, multiplying I/O. Loopback SMB's
per-op overhead and its serialization through a single smbd hurt here. **Local
access is the better choice when transcoding is in play** — and our topology
(#10) explicitly plans for NVENC transcoding.

**Seeking / scrubbing.** When a user scrubs, the client issues a range request;
Jellyfin seeks the file and re-buffers. Each seek is a small random read. SMB's
per-op cost (and, historically, weaker behavior on sparse/seek-heavy reads than
local VFS) adds latency to every scrub. ZFS ARC absorbs repeated seeks either
way, but the *first* seek after a cold spot pays the SMB round-trip penalty
that a bind mount wouldn't. **Local access gives snappier seeking.**

**Benchmark note.** I did not find a rigorous published benchmark of
loopback-SMB-vs-bind-mount for a same-host Jellyfin container. The
characterization above rests on (a) the general loopback-overhead analysis
(Stack Overflow, Microsoft's slow-SMB troubleshooting guide), (b) Jellyfin
forum/Reddit community reports of SMB stutter and the common advice to "mount
it local," and (c) first-principles reasoning about the data path. **Confidence
on the *direction* of the effect is high; confidence on exact magnitudes is
low — but for our topology the magnitudes don't matter, because the cheaper
option (bind mount) is also the simpler and more secure one.**

<details>
<summary><b>ZFS dataset tuning (applies to both access methods — but matters here)</b></summary>

Jellyfin's storage doc gives explicit ZFS guidance that should ride along with
whatever access method #13 picks:

- **Media dataset:** recordsize **1M** for large media files. "For ZFS datasets
  containing large media files… a record size of 1 M is likely appropriate for
  optimal performance." Do **not** use 4K/8K for media ("you will likely
  encounter performance issues").
- **Jellyfin database / config dataset:** recordsize **4K or 8K** (SQLite). The
  default 128K is wrong for the DB. "Ideally, you should use a record size of
  4 K or 8 K on the dataset that contains your Jellyfin Server SQLite
  database."
- **Don't put the DB / transcode on ZFS-on-mechanical.** "Do not use
  ZFS-formatted mechanical drives to store your Jellyfin Server data (everything
  except your media files)."
- **Transcode dir sizing:** "a single 50 GB Blu-ray remux might consume as
  much as ~60 GB or as little as ~15 GB after transcoding." Size the transcode
  dataset (or tmpfs) accordingly.
- **DB growth:** "A database for a moderate-sized library can grow anywhere
  from 10 to 100 GB."

These are dataset-level decisions independent of bind-mount-vs-SMB, but they
reinforce that the **access method is the smaller half of the storage story** —
getting recordsize right matters more than which local path the bytes arrive
on. Both get the same ARC either way.

</details>

---

## Point 4 — Portability vs locality

| Argument for SMB (portability) | Counter for our case |
|---|---|
| Jellyfin could move to a different host later and still see the same library by re-mounting the same share. | The ZFS pool *is* the TrueNAS box. Moving Jellyfin off the box means the media has to move too (or stay and be accessed over real SMB, which is a different topology than this ticket). "Jellyfin moves, media stays" is not a realistic homelab move for us — it would mean re-hosting multi-TB media. |
| SMB decouples the container from host path layout. | The host path layout (`/mnt/puddle/<dataset>/...`) is already a first-class thing in our repo's mental model; caddy and dashy bind-mount co-located host files. Decoupling from it would be inconsistent, not cleaner. |
| SMB is "standard" — many tutorials show it. | Tutorials show SMB because their authors' media is on a *different* box (a NAS separate from the Docker host). Copying that pattern onto a same-box topology is cargo-culting. |

| Argument against SMB (the cost) | Severity |
|---|---|
| Extra failure surface: smbd, mount-on-boot ordering, credentials, stale handles, library-scan-on-unavailable-storage deleting items. | **Real and recurring.** The Jellyfin doc's "remove items from your library if triggered while your media storage is unavailable" warning is the sharp edge. |
| CPU/latency overhead (point 2/3). | Modest for direct play, material for transcode/seek. |
| Forces a privileged container (point 5, option b) OR a host-side mount+bind (point 5, option a) — the latter still has overhead and adds an fstab dependency. | **High** — either outcome is worse than a plain bind mount. |

**Verdict for our topology: locality wins.** Portability is a hypothetical
future cost-saver; the overhead and fragility are present-tense and certain.
The repo is not trying to be multi-host-portable — it is explicitly a
single-TrueNAS-box homelab stack tree. Optimize for the topology you have.

---

## Point 5 — Mount mechanics if SMB were chosen

Two places the SMB mount could live, each with a sharp tradeoff:

**Option A — host-side mount, then bind-mount into the container.**

```yaml
# (host) /etc/fstab or a systemd .mount unit:
# //127.0.0.1/media  /mnt/smb/media  cifs  credentials=...,uid=568,gid=568,iocharset=utf8,vers=3.0  0  0

# (compose) the container just sees a local path:
volumes:
  - /mnt/smb/media:/media:rw
```

- Container stays **unprivileged** — consistent with caddy/dashy posture.
- But: re-introduces **all** the loopback overhead from point 2, **plus** a
  host-side mount dependency (boot ordering, credential file, remount-on-smbd-restart).
- On TrueNAS SCALE, managing a host-level CIFS mount of the box's *own* export
  is unusual and unsupported territory — the appliance model expects you to
  just read the local dataset. You'd be fighting the platform.

**Option B — mount inside the container via `mount.cifs`.**

This requires the container to perform the `mount(2)` syscall, which needs
**`CAP_SYS_ADMIN`** — and per Docker forum reports, in practice often also
**`CAP_DAC_READ_SEARCH`**, or just full **`--privileged`** mode:

- Docker forum (rimelek / Ákos Takács): `--privileged` is **not** a security
  feature — "Using a privileged flag will make your container more dangerous
  giving you all possible capabilities and access to the host devices." The
  right question is which capability is actually missing; `DAC_READ_SEARCH` is
  the usual suspect for CIFS.
- Community consensus (Docker forum, Stack Overflow, r/sysadmin): **mount on
  the host, bind-mount into the container** is the documented best practice
  specifically because it avoids privileged containers.

For our repo this is **disqualifying**: caddy and dashy carry **no** added
capabilities, and the whole stack tree is designed around unprivileged
Dockhand-managed containers that only get the docker socket (caddy) or nothing
(dashy). Introducing a privileged Jellyfin — or even a `CAP_SYS_ADMIN` one —
would be the most privileged container in the fleet, on the same box as the
TLS-terminating edge proxy. **Option B is a non-starter.**

So if SMB were chosen, only Option A is acceptable, and Option A is still
worse on the merits (point 2/4) than a direct bind mount. The mechanics
reinforce the recommendation rather than opening a path to it.

<details>
<summary><b>Why CAP_SYS_ADMIN is the capability people warn about</b></summary>

`CAP_SYS_ADMIN` is the catch-all "do privileged filesystem and namespace
operations" capability in Linux. It covers `mount(2)`, `umount(2)`,
`pivot_root(2)`, setting hostname/domainname, various ioctl operations, and a
long tail of admin operations. It is routinely described as "the new root" —
granting it effectively undoes most of the container boundary. A container
with `CAP_SYS_ADMIN` can, in many configs, escalate to full host compromise
(e.g. by mounting the host root filesystem). This is why Trend Micro and
others flag privileged containers as a backdoor risk, and why the conservative
posture is: if a workload needs `CAP_SYS_ADMIN`, find another way to give it
what it needs. For SMB, "another way" = host-side mount + bind (Option A) =
the same pattern the community recommends for exactly this reason.

</details>

---

## Point 6 — Recommendation for #13

**Use a host-path bind mount.** In compose terms:

```yaml
volumes:
  - /mnt/puddle/<media-dataset>:/media:rw
  # plus a local path or named volume for /config (DB must be local — point 1)
```

**One-line rationale:** *the media is already on the same box and ZFS pool as
the container, so a network mount can only add overhead and a failure surface —
and the in-container CIFS alternative would force a privileged container that
breaks the repo's unprivileged posture; a direct bind mount is strictly
better on performance, simplicity, and security, and it matches the precedent
already set by caddy and dashy.*

**Confidence: high.** This is not a close call given the topology. The only
world in which SMB wins is one where Jellyfin and the media pool are on
*different* hosts — and that is not our world.

**Caveats to carry into #13's implementation (not blockers, just follow-ups):**

- The media dataset should be at ZFS **recordsize 1M** (Jellyfin doc, point 3's
  ZFS sidebar). Confirm/set this when laying out the dataset.
- The Jellyfin **config/database** path must be **local** and on a
  **small-recordsize (4K–8K)** dataset, ideally not on ZFS-over-mechanical.
  This is independent of the media access method but easy to get wrong.
- The **transcode** directory should be sized for ~60 GB per concurrent remux
  transcode; tmpfs or a fast local dataset is appropriate. Same point as GPU
  research (#10) — transcode I/O is its own concern.
- Ownership: the container (UID/GID from the linuxserver image, typically
  1000:1000 unless overridden) must be able to read the media dataset and
  read/write config+transcode. ACLs or `chown -R` on the dataset at setup time.

---

## Confidence notes

- **Doc-backed, high confidence:** Jellyfin's storage guidance (OS-level mount
  required; DB must be local; NFSv3 locking caveat; ZFS recordsize 1M for
  media / 4K–8K for DB; don't use ZFS-on-mechanical for DB) — all from the
  official Jellyfin docs page. The Docker-internal-CIFS-requires-capabilities
  point is from the Docker forum (rimelek) and corroborated by Stack Overflow /
  r/sysadmin; the consensus "host-mount + bind" pattern is widely stated.
- **First-principles + community, high-direction / low-magnitude confidence:**
  the loopback-SMB-overhead characterization (point 2) and the
  streaming/transcode/seek performance comparison (point 3). Direction of the
  effect is well-grounded (data-path reasoning + general loopback-overhead
  analyses on Stack Overflow / Microsoft slow-SMB guide + community "mount it
  local" advice). I did **not** find a rigorous published benchmark of
  loopback-SMB-vs-bind-mount for same-host Jellyfin specifically, so exact
  numbers are not cited. For our topology this doesn't matter — the cheaper
  option is also the simpler and more secure one.
- **Repo-internal, high confidence:** the unprivileged-container posture
  (caddy has no `cap_add`; dashy has no `cap_add`), the bind-mount precedent
  (`caddy` bind-mounts `./Caddyfile` and the docker socket; `dashy`
  bind-mounts `./conf.yml`), and the single-box homelab framing (README).

## Sources

- [Jellyfin — Storage (official docs)](https://jellyfin.org/docs/general/administration/storage/) — canonical: OS-mount requirement, DB-local requirement, NFSv3 locking caveat, ZFS recordsize guidance.
- [Jellyfin — Hardware acceleration overview](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/) — context for transcode I/O expectations.
- [Docker forum — SMB/CIFS mount within container](https://forums.docker.com/t/smb-cifs-mount-within-container/6661) — `CAP_SYS_ADMIN` insufficient; `--privileged` is not a security feature; host-mount + bind is the expected pattern (rimelek / Ákos Takács).
- [Stack Overflow — Mount SMB/CIFS share within a Docker container](https://stackoverflow.com/questions/27989751/mount-smb-cifs-share-within-a-docker-container) — capability requirements; host-side mount recommended.
- [Trend Micro — Why running a privileged container in Docker is a bad idea](https://www.trendmicro.com/en_us/research/19/l/why-running-a-privileged-container-in-docker-is-a-bad-idea.html) — `CAP_SYS_ADMIN` / privileged-container risk.
- [Microsoft Learn — Slow SMB file transfer troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/slow-smb-file-transfer) — SMB performance factors (general reference).
- [Stack Overflow — Does accessing data through "localhost" incur network stack overhead?](https://stackoverflow.com/questions/33790358/windows-does-accessing-data-through-localhost-incur-network-stack-overhead) — loopback access adds overhead vs direct local access.
- [Server Fault — How fast is 127.0.0.1?](https://serverfault.com/questions/234223/how-fast-is-127-0-0-1) — loopback is handled in software, no NIC.
- [Apple Stack Exchange — Can I connect to a local SMB share?](https://apple.stackexchange.com/questions/98331/can-i-connect-to-a-local-smb-share) — OS actively discourages `smb://localhost`.
- [Repo briefing — `truenas/apps/README.md`](../truenas/apps/README.md) — per-stack conventions, unprivileged posture, bind-mount precedent.
- [Repo briefing — `truenas/apps/caddy/compose.yaml`](../truenas/apps/caddy/compose.yaml) · [`dashy/compose.yaml`](../truenas/apps/dashy/compose.yaml) — bind-mount precedent, no `cap_add`.
- [Repo briefing — `research/jellyfin-gpu-passthrough.md`](./jellyfin-gpu-passthrough.md) — sibling research; transcode planning context.
