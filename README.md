# Flowbit v2 —— Seed-style Base Distribution & Edge Computing Architecture

> Formerly BCST-DT (Directory-Based Compressed Sliced Tunnel Delta Transmission).
> While retaining the core principle of "server-side without relying on user P2P",
> v2 refactoring is the architecture from "extreme compression + fixed slicing"
> to "seed-style base package + on-demand fetching + multi-purpose edge nodes".

---

## Design Philosophy

The core metaphor of Flowbit v2 remains a spring:

A spring does not gush out an entire lake at once; it seeps out continuously, on-demand, with minimal resistance.
The base package is not "a compressed archive of all game files" — it is the ticket to launch the game, a seed file.
Once inside, the remaining resources are fetched on-demand from edge nodes.
Developers do not need to build expensive CDNs or ask users to share bandwidth via P2P.
Tunnel nodes act as both file caches and multiplayer relays — one server, two jobs, maximum utilization.

---

## The Triangle Constraint

| Dimension | v2 Approach |
|-----------|-------------|
| Size | Reduce at the source (SVG replacement + shared_lib deduplication), not by brute-force compression |
| Speed | Small-grained category packages + no/minimal compression, decompression time near zero; fast HTTPS transfer |
| Compatibility | Pure HTTPS + standard tar + optional zstd, zero special client dependencies |

---

## Feasibility

| Condition | Satisfied |
|-----------|-----------|
| No need for users to open ports | Yes |
| No need for users to run special software | Yes |
| No need for users to share upload bandwidth | Yes |
| No reliance on commercial CDN | Yes |
| Server-side independently controllable | Yes |
| Can run on a single server | Yes (origin server + at least 1 tunnel node) |

---

## Goal 1: Seed-Style Base Package —— Just Enough to Enter the Game

**v1 approach:** Tar all files, compress with 7z LZMA2 at maximum settings, split into 50MB slices, client downloads and extracts everything.

**v2 approach:** The base package contains only the "minimum set to launch the game", targeting 50-100MB. It is essentially a seed file — containing a tunnel node address table and a resource manifest, but not all game resources.

Base package contents:

| Content | Description |
|---------|-------------|
| Game engine / runtime | Required to launch |
| Core scripts / config | Required at startup |
| Base UI (SVG) | Extremely small |
| starter_content/ | 1-2 initial scenes/characters, just enough to see the screen |
| shared_lib/ | Public file library (deduplicated common resources) |
| tunnel_nodes.json | Tunnel node address list (multiple nodes, with priority and region) |
| resource_manifest.json | Complete resource manifest (path + hash + size + category + priority) |

Once published, the base package is never modified. All subsequent changes are applied incrementally via patches and category packages.

---

## Goal 2: Public File Library (shared_lib) —— Deduplication at the Source

**v1 approach:** Each directory stores its own files; updating requires re-downloading entire slices.

**v2 approach:** Scan all directories, identify byte-for-byte identical files, extract them to shared_lib/. Leave .ref pointer files in the original locations, containing a single line path pointing to the actual file in shared_lib.

| Comparison | v1 | v2 |
|------------|----|----|
| Same button icon appears in 8 directories | Stored 8 times | Stored once + 8 pointers |
| Changing that icon | Update all 8 directories | Update only 1 file in shared_lib |
| Client download | Download 8 copies | Download 1 copy, all references share |

Pointer file example:

Filename: ui/buttons/confirm.png.ref
File content: shared_lib/btn_confirm.webp

Marked as type: "ref" in the manifest. When the client reads it, it automatically fetches from the shared_lib path.

---

## Goal 3: Category Archiving + No Double Compression

**v1 approach:** Tar all files into a single stream, compress with 7z LZMA2 at maximum settings.

**v2 approach:** Group files by type/theme into folders, then process each folder individually:

| Category | Handling | Reason |
|----------|----------|--------|
| base/ (code, config, bin/data) | tar.xz, LZMA2 max compression | Highly compressible, high dictionary hit rate |
| characters/ (character models + textures) | tar.xz, LZMA2 max compression | Cross-file redundancy within同类 files can be exploited after packing |
| costumes/ (costume textures) | tar.xz, LZMA2 max compression | Same as above |
| ui_textures/ (UI textures) | tar.xz, LZMA2 max compression | Same as above |
| audio_bgm/ (background music) | tar.zst, store mode | Already-compressed audio cannot be compressed further; archive only, no decompression needed |
| videos/ | tar.zst, store mode | Same reason |
| Event files (event_xx/) | No compression, files served directly | Short-lived resources, not worth CPU time to compress or decompress |

Key principle: **Never double-compress.** Each file is compressed at most once (or not at all). Already-compressed formats (WebP/PNG/JPG/MP3/MP4) are archived in store mode only.

---

## Goal 4: Tunnel Nodes —— From BT Mesh to HTTPS Edge Nodes

**v1 approach:** Tunnel nodes run BT clients, pre-push slices, users pull from nodes.

**v2 approach:** Tunnel nodes are reduced to standard HTTPS file servers + multiplayer relays. Two sync modes are supported:

| Mode | Description |
|------|-------------|
| Lazy cache (recommended for starting out) | Client requests → node misses → fetches from origin → caches → returns |
| Active sync (for scale) | Origin server pushes new versions to nodes; nodes can sync among themselves |

Node storage strategy:

| Data Type | Storage Location | Eviction Policy |
|-----------|-----------------|-----------------|
| Base package + shared_lib | NVMe hot cache, permanent | Never evicted |
| Current event files | NVMe hot cache | Deleted via del-patch when event ends |
| Old category packages / old events | HDD warm cache | LRU, auto-delete after 30 days of no access |
| Files marked tombstone | Any | Physically deleted after 7 days |

Nodes serve HTTPS via nginx with Let's Encrypt or self-signed certificates. No BT client, no special protocol needed.

---

## Goal 5: Patches & Activity Lifecycle —— Incremental Layering + Safe Deletion

**v1 approach:** bsdiff generates deltas, merges when threshold reached, del-patch deletes expired files.

**v2 approach:**

Patch strategy:

| Patch Type | Handling | Location |
|------------|----------|----------|
| Config / text modification | Standalone patch file | patches/patch_xxx.json |
| Single small file replacement | Standalone patch file | patches/patch_xxx.bin |
| Code / logic hotfix | Standalone patch file | patches/patch_xxx.script |
| Batch modification of multiple files | Rebuild category package | characters_v1.2.tar.xz |
| Event launch | Entire event directory pushed to nodes | events/event_xx/ |
| Event takedown | del-patch instruction | del_patches/del_xxx.json |

Deletion flow (core safety mechanism):

| Step | Executor | Description |
|------|----------|-------------|
| 1. File marked as deprecated | System auto | Triggered when event ends or patch is superseded |
| 2. Sandbox verification | Automated | Built-in client (internal test server) launches game, verifies normal operation after deletion |
| 3. AI analysis suggestion | AI-assisted | Scans dependency graph, outputs recommendation report (does NOT execute deletion) |
| 4. Human confirmation | QA / Developer | Reviews AI report + sandbox result, manually approves or rejects |
| 5. del-patch dispatched | System | Client + tunnel nodes delete synchronously |
| 6. Rollback window | System | Retained for 7 days, can be revoked during this period |
| 7. Origin server archival | System | Moved to cold storage |
| 8. Final deletion | Human / scheduled | Confirmed permanently unnecessary, then purged |

AI only provides suggestions. It has no super-user privileges and cannot proactively clean up files.

---

## Security & Transmission Spec

All communication between nodes and between nodes and clients MUST go through HTTPS (TLS 1.3). Plaintext HTTP is not allowed.

| Item | Recommendation |
|------|----------------|
| Certificate type | Start with Let's Encrypt (free) |
| Advanced recommendation | If possible, purchase commercial CA certificates and complete ICP filing (for users in mainland China) |
| Benefits of filing | More stable TLS handshake, avoids ISP hijacking, better compatibility with older devices |
| Application-layer encryption | Optional AES-256-GCM: origin encrypts → client decrypts, nodes store ciphertext only |
| Manifest signing | Ed25519 private key signature; client verifies signature before trusting content |
| File integrity | Each file includes a SHA-256 hash; client verifies after download |

---

## Transmission Performance Optimization

| Optimization | Method | Effect |
|--------------|--------|--------|
| HTTP/2 | Enable http2 in nginx | Multiplexing, reduces latency by 30-50% |
| Brotli pre-compression | Generate .br files at build time, nginx serves them directly | manifest/JSON size reduced by 70-80% |
| Cache-Control | Category packages set to max-age=31536000, immutable | Client does not re-request on second launch |
| TCP BBR | Enable BBR congestion control in kernel | Cross-region sync 2-5x faster |
| Range requests | Natively supported by nginx | Resume downloads, no restart on weak networks |
| Inter-node sync | rsync + SSH | Transfers only differences, supports resume |

---

## Recommended Specifications

### Origin Server (Main Server / Multiplayer Core)

| Component | Recommended | Notes |
|-----------|-------------|-------|
| CPU | 12 cores (i7-13700K / Ryzen 7 7700X) | Multiplayer core + matchmaking + Flowbit origin service |
| Memory | 32GB DDR4 dual-channel (2×16GB) | 16GB is too tight; 32GB has headroom; 64GB for scale |
| System disk | 512GB NVMe SSD | OS + logs |
| Data disk | 2TB NVMe SSD | Hot data + build artifacts + current full version |
| Network | 2.5Gbps dedicated | Origin fallback + coordination |
| Power | Idle ~20W, full load ~130W | |

### Tunnel Nodes (Edge Nodes)

| Component | Recommended | Notes |
|-----------|-------------|-------|
| CPU | 4 cores (i3-12100 / N100 / RK3588 ARM) | File service + relay + lightweight room hosting |
| Memory | 16GB DDR4 dual-channel (2×8GB) | File cache page cache + connection buffer |
| System disk | 128GB SATA SSD | OS |
| Data disk | 512GB NVMe (hot cache) + 2TB HDD (warm cache) | NVMe for active content + base; HDD for older |
| Network | 1Gbps | Relay throughput bottleneck is at the NIC |
| Power | Idle ~6-8W (ARM) / ~12W (x86), full load ~35W | |

### Build Server

| Component | Recommended | Notes |
|-----------|-------------|-------|
| CPU | 8+ cores | tar.xz LZMA2 is single-threaded; more cores allow parallel compression of multiple category packages |
| Memory | 32GB DDR4 dual-channel (2×16GB) | Each LZMA2 thread consumes 1-2GB RAM |
| Data disk | 1TB NVMe SSD | Build intermediate files are I/O intensive |
| Power | On-demand, runs only during builds | Monthly cost can be as low as ¥10-20 (pay-as-you-go instances) |

---

## Power & Energy Reduction

### Hardware Level

| Method | Effect |
|--------|--------|
| Use ARM-based nodes | Power drops from ~25W to 4-6W |
| Dual-channel memory (2 sticks) | Doubles bandwidth, higher TLS throughput, serves more concurrent connections |
| HDD spin-down | 0 RPM after 5 minutes of inactivity |
| NVMe ASPM power management | Enters L1.2 low-power state when idle |
| CPU frequency scaling | powersave mode at idle, cuts power in half |
| Headless operation | No monitor attached, SSH management, saves GPU power |

### Software Level

| Method | Effect |
|--------|--------|
| nginx worker count = CPU cores | No idle spinning |
| TLS session resumption | Reduces handshake CPU cost |
| Rate limiting + queuing | Prevents traffic spikes from maxing out CPU |
| Off-peak builds | Run compression at 3-5 AM |
| Client request coalescing | Merge pending downloads every 5 seconds, fewer connections |

### Actual Power Cost Estimate (self-hosted, 3 nodes + 1 origin)

| Device | Daily avg. power | Monthly cost (¥0.6/kWh) |
|--------|-----------------|------------------------|
| Origin server | ~1.2 kWh | ~¥22 |
| 3× ARM tunnel nodes | ~0.15 kWh each | ~¥8 each |
| Total | | **~¥46/month** |

When no one is playing and tunnel nodes auto-shut down → monthly cost drops further to ~¥15.

---

## Cost Control

| Strategy | Method |
|----------|--------|
| Auto start/stop tunnel nodes | Shut down below threshold, spin up above threshold (cloud pay-as-you-go instances) |
| Cold storage archival | Files not accessed for 30 days moved to object storage infrequent-access tier (a few RMB/TB/month) |
| LAN client-to-client transfer | Client mDNS discovers neighbors; already-downloaded resources fetched directly from LAN peers |
| Rollback window | del-patch retained for 7 days; confirmed no issues before final deletion, avoids re-distribution after accidental deletion |
| Origin server as fallback | Normally serves zero traffic; all requests go through tunnel nodes; origin only used if all nodes fail |

---

## Multiplayer Extension (Multi-Purpose Edge Nodes)

Tunnel nodes do more than file distribution — they also handle multiplayer relay and compute overflow:

| Capability | Description | When Enabled |
|------------|-------------|--------------|
| File service | HTTPS file cache and distribution | Always on, low priority |
| Multiplayer relay | Client ↔ tunnel ↔ main server, forwards game state sync packets | When users are playing multiplayer |
| Room hosting overflow | When main server is overloaded, match rooms are assigned to run on tunnel nodes (Docker containers) | Main server CPU > 70% or memory > 80% |
| Inter-node direct connect | UDP hole punching assist; clients connect directly when possible, node relays only on failure | At large scale |

The main server only handles matchmaking and global state. Match data travels directly between tunnel nodes. Docker cgroups ensure file service and multiplayer processes are resource-isolated and do not compete.

---

## Known Limitations

| Limitation | Description |
|------------|-------------|
| Base package does not contain all resources | Background download needed after first launch; weak-network users are affected |
| Tunnel nodes need public IPs | Each node still needs a reachable address (can use NAT traversal as alternative, but adds latency) |
| Multiplayer overflow adds complexity | Room logic must be synced between main server and nodes; increases development effort |
| Not compatible with v1 clients | v2 protocol is incompatible with v1; client must be re-implemented |
| Already-compressed media cannot be re-compressed | Video/audio/WebP can only use store mode, resulting in larger size than extreme compression |

---

## License

GNU Affero General Public License v3.0 (AGPL-3.0). See LICENSE file for details.

---

## Summary

Flowbit v2 can be used not only for **multiplayer/game servers**, but also for everyday scenarios — as a more advanced **directory distribution** system, just like its predecessor.

Created by: Dmr(Dts967)
**No one may privately patent this concept without permission.**
