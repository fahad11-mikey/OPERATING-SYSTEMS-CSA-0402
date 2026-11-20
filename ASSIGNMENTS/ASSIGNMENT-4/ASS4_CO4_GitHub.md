# CSA04 – Operating Systems | Assignment 4 (CO4)
## Optimising File Storage and Disk Access

**Name:** ____________________  **Register No.:** ____________________  
**Compiler:** GCC (C11)  **OS:** Linux (record your actual version)  

---

## Task 1 – File-System Design Review

### 1.1 Weaknesses of the proposal
1. A single directory containing 12,000 students can become difficult to manage and search.
2. Student-chosen filenames can collide, causing overwrites or confusion.
3. Contiguous allocation requires adjacent free blocks; growing a file may require relocation.
4. No permissions allow unauthorized students to read, modify or delete others' files.
5. A single shared location increases the impact of accidental deletion, corruption or server failure.
6. Group-project sharing is difficult to manage safely without shared folders and permissions.

### 1.2 Improved design
Use a hierarchical directory structure, such as `/students/year/department/register-number/`. Use register numbers as unique folder names and allow descriptive filenames inside. Store file attributes (owner, size, timestamps, type and permissions). Use permission bits or ACLs to give each student private access. Create separate project directories and grant group ACLs to members. Use indexed allocation or extents for files that grow, regular backups, snapshots and recovery testing.

### 1.3 Design defence (under 100 words)
A hierarchical directory and unique register-number folders reduce name clashes and make files easier to locate. Owner permissions and ACLs prevent other students from reading or changing private files. Group-project folders are shared only with approved members, so collaboration does not require opening all student data. Backups and snapshots help recover deleted or damaged files. This design provides controlled sharing while protecting privacy.

### Alternative: Cloud storage
Cloud services provide searchable metadata, indexing and access from different devices. Finding files can be fast, but opening them depends on network latency and service availability. Storage may include version history or duplicated copies, increasing usage and cost. The college can control accounts, permissions and retention policies, but must review provider access, encryption, data location and privacy terms.

---

## Task 2 – File Allocation for Video Surveillance

### 2.1 Operations and access pattern
- Create/open recording files and continuously append video: frequent sequential writes.
- Playback/export: sequential reads.
- Search by time: direct/random access using time-to-block metadata or an index.
- Delete footage after 30 days: file deletion and free-space updates.
- Metadata lookup and directory operations: frequent but relatively small.

### 2.2 Allocation-method comparison
| Method | Sequential access | Direct access | Growth | Main issue |
|---|---|---|---|---|
| Contiguous | Very fast | Fast | Difficult | External fragmentation; relocation on growth |
| Linked | Good when traversing in order | Slow | Easy | Pointer traversal; pointer damage risk |
| FAT | Good with cached FAT | Slower chain traversal | Easy | FAT consumes memory; FAT corruption affects chains |
| Indexed | Good | Fast through index | Easy within index capacity | Index-block overhead; index corruption risk |

**Choice:** Indexed allocation (or extent-based allocation) is suitable because recordings grow and time-based seeks are needed. Maintain a timestamp-to-block index for quick playback seeking.

### 2.3 Cost calculations
File size = 4 GB = 4,096 MB; block size = 4 KB. Using binary units:
- Blocks = 4 GB / 4 KB = **1,048,576 blocks**.
- Contiguous: direct calculation of the final block location; **1 data-block read** to read the last block (not counting initial metadata).
- Linked allocation: to reach the last block from the beginning, traverse **1,048,576 blocks/links** (worst case; each block must be read in sequence).
- Indexed allocation: read the index block(s), then the target data block; for a single-level index, **2 reads** (index + data), assuming the index fits. For a larger file, additional index levels are required.

**Inode maximum file size:** 4 KB blocks and 4-byte addresses gives 1,024 addresses per indirect block.

Blocks addressable = 12 + 1,024 + 1,024² + 1,024³ = **1,074,791,436 blocks**.
Maximum size = blocks × 4 KB = **4,402,345,721,856 bytes ≈ 4.004 TiB**.

### 2.4 Free-space management for 4 TB
- Bit vector: one bit per 4 KB block. 4 TB / 4 KB = 1,073,741,824 blocks; bit vector = 1,073,741,824 bits = **128 MiB**.
- Linked free list: low bitmap overhead, but finding many free blocks requires traversal.
- Grouping/counting: stores groups or runs of free blocks, allowing faster bulk allocation when free space is clustered.

### Complete the Code – Contiguous First-Fit
```c
int alloc_contiguous(char file, int size)
{
    int run = 0, start = -1;
    for (int b = 0; b < BLOCKS; b++) {
        if (disk[b] == '.') {
            if (run == 0) start = b;
            run++;
            if (run == size) {
                for (int j = start; j < start + size; j++)
                    disk[j] = file;
                return start;
            }
        } else run = 0;
    }
    return -1;
}
```
**Expected explanation:** In the sample layout, free space is split into separate holes. Although the total free blocks may be sufficient, no single run of 10 blocks is available, so contiguous allocation fails. Indexed allocation can place data blocks in different free locations and store their addresses in an index block, so it can allocate the file if enough individual blocks are available.

### Alternative: FAT
With an in-memory FAT, reaching block *k* still requires following approximately *k* chain entries (**O(k)**). Indexed allocation can locate a block with a direct index lookup (**O(1)** for a single-level index; multi-level indexing takes a small number of reads). A 4 TB disk with 4 KB clusters has 1,073,741,824 clusters; a 4-byte FAT entry would require about **4 GiB** of RAM/storage for the FAT. FAT damage can disrupt many file chains; indexed allocation damage is more localized to the affected index.

---

## Task 3 – Modern File Systems

### 3.1 Comparison
| File system | Typical use | Journaling | Snapshots | Checksums / CoW |
|---|---|---|---|
| FAT32 | Removable media, broad compatibility | No | No built-in snapshots; no CoW |
| NTFS | Windows disks | Metadata journal | No native snapshot feature like CoW FS; integrity features available |
| ext4 | Linux general-purpose/server | Yes | No native CoW snapshots; metadata/data checksums vary by feature |
| XFS | Large Linux files, servers | Yes | No native filesystem snapshots/CoW |
| Btrfs | Linux, flexible storage | CoW design | Snapshots, checksums |
| ZFS | Storage/backup servers | Transactional CoW | Snapshots, checksums, pools |
| APFS | Apple devices | CoW | Snapshots, encryption, space sharing |

*Limits vary by implementation and configuration; verify current platform limits before deployment.*

### 3.2 Journaling and consistency
A crash during a write may leave metadata updates incomplete. Journaling records planned metadata changes so the filesystem can replay or roll back the journal after restart. Copy-on-write writes changed data/metadata to new locations and updates pointers only after the new version is ready, preserving the previous consistent state.
- **ext4 journal:** journals data and metadata; stronger protection, more write overhead.
- **ordered:** journals metadata and writes file data before committing related metadata; common default.
- **writeback:** journals metadata but does not guarantee data is written before metadata commit; can be faster but weaker after a crash.

### 3.3 Suitable choices
- Pen drive shared between Windows and Linux: **FAT32** for maximum compatibility (large files over 4 GB are not supported); **exFAT** is often better for large files if supported on both systems.
- Linux database server: **ext4** for mature reliability and straightforward administration; XFS is also suitable for high-throughput workloads after benchmarking.
- Laptop: **NTFS** for Windows, **APFS** for macOS, or **ext4** for Linux, based on the operating system.
- Backup server needing snapshots: **Btrfs or ZFS**, because snapshots and checksums help recovery and integrity.

### 3.4 Filesystem inspection commands
Run on Linux and include your own terminal screenshots/output in the repository:
```bash
stat .
df -i .
ls -i .
sudo tune2fs -l /dev/<your-ext4-device>
# Alternative:
sudo dumpe2fs -h /dev/<your-ext4-device>
```
Do not run the last commands against a device until you identify the correct filesystem device.

### Alternative: Btrfs/ZFS for database storage
CoW filesystems provide snapshots and checksums, but small random updates can cause extra allocation and metadata work. Snapshot space depends on how much data changes while snapshots are retained; snapshots initially share blocks and grow as blocks change. Checksums detect corruption. Recovery may be faster because the filesystem can roll back or mount a consistent snapshot, though actual database recovery and performance depend on configuration and workload testing.

---

## Task 4 – Disk Scheduling

Requests: **98, 183, 37, 122, 14, 124, 65, 67**; initial head = **53**; cylinders = 0–199.

### 4.1 Service order and head movement
Assume SCAN/LOOK initially move toward cylinder 0; C-SCAN/C-LOOK move toward higher cylinders first. For SCAN, the head reaches 0 before reversing. For C-SCAN, it reaches 199 and wraps to 0.

| Algorithm | Service order (including boundary where applicable) | Total movement |
|---|---|---:|
| FCFS | 98,183,37,122,14,124,65,67 | 640 |
| SSTF | 65,67,37,14,98,122,124,183 | 236 |
| SCAN | 37,14,0,65,67,98,122,124,183 | 236 |
| C-SCAN | 65,67,98,122,124,183,199,0,14,37 | 382 (includes wrap) |
| LOOK | 37,14,65,67,98,122,124,183 | 208 |
| C-LOOK | 37,14,183,124,122,98,67,65 | 328 |


### 4.2 Simulation measurements
For 10,000 requests under light, medium and heavy load, run your simulation and record **actual measured** average head movement and maximum waiting time. Do not invent measurement results. Define the load levels (request arrival rate and queue size) and use the same generated workload to compare algorithms fairly.


### 4.3 Fairness
SSTF can starve requests far from the current head because closer requests may continually arrive. SCAN provides more predictable service by sweeping across the disk. C-SCAN services requests in one direction and wraps to the beginning, giving more uniform waiting time than SCAN, which services requests in both directions and can favour cylinders near the middle.

### 4.4 SSD scheduler
SSDs have no mechanical seek time, so traditional seek-optimizing algorithms provide less benefit. Linux may use `none` to avoid unnecessary scheduling or `mq-deadline` to manage latency and request ordering. Check your own device:
```bash
cat /sys/block/<disk>/queue/scheduler
```

### Complete the Code – SSTF TODO
```c
for (int k = 0; k < N; k++) {
    int best = -1, best_dist = MAXCYL + 1;
    for (int i = 0; i < N; i++) {
        if (!done[i]) {
            int dist = abs(req[i] - pos);
            if (dist < best_dist) {
                best_dist = dist;
                best = i;
            }
        }
    }
    done[best] = 1;
    order[k] = req[best];
    pos = req[best];
}
```
**Expected SSTF output:** `53 65 67 37 14 98 122 124 183 -> 236 cylinders`.

**Continuous requests near cylinder 60:** SSTF repeatedly serves nearby requests and may delay cylinder 183 for a long time. With an endless stream of close requests, the distant request can starve.

### Alternative: LOOK/C-LOOK
LOOK turns at the last request in the current direction rather than moving to the disk boundary. C-LOOK similarly wraps from the last request in one direction to the first request in the other direction. They avoid unnecessary boundary travel, reducing movement for this queue. Finding the next request by scanning unsorted requests is **O(n)** per choice; maintaining a sorted queue can reduce selection overhead, at the cost of insertion/update work.

---

## Task 5 – Distributed and Cloud Storage

### 5.1 Comparison
| Option | Consistency | Availability | Performance | Cost |
|---|---|---|---|---|
| NFS | Depends on protocol/server configuration | Server-dependent; single server can be a failure point | LAN access, simple shared files | Low setup; server maintenance |
| Distributed FS (HDFS/Ceph) | Distributed consistency mechanisms | Replication and failover | More nodes and administration |
| Cloud object storage (S3/GCS) | Service-specific consistency model | High durability/availability by provider design | Pay for storage, requests and data transfer |

### 5.2 Replication and consistency
Three replicas can keep data available when a node fails, provided replicas are placed on independent failure domains. Replication improves durability and availability but consumes roughly three times raw storage for three full copies (before compression/overhead). Strong consistency makes updates visible predictably but may require coordination and affect latency. Eventual consistency can improve availability/latency but clients may temporarily see stale data, depending on the service.

### 5.3 Device management
- **Device drivers:** translate OS requests into device-specific commands.
- **Buffering:** temporarily holds data during transfers to handle speed differences.
- **Caching:** keeps frequently used data in faster memory to reduce disk access.
- **Spooling:** queues jobs for a device that processes work sequentially (commonly printing).
- **Block-device interface:** lets the OS read/write fixed-size blocks and manage requests independent of filesystem details.

### Alternative: one large NFS server
A single NFS server is simpler and may be cheaper, but reads/writes can bottleneck as users increase; throughput is limited by server CPU, network and disks. It stores one copy, using less capacity than three replicas, but server failure can make all files unavailable unless a separate backup/failover exists. A replicated distributed system costs more and uses extra storage but offers better fault tolerance and scaling.

---

## Task 6 – Design Challenge: Video-Streaming Storage

### Stage 1 – Workload and targets
- Upload: one sequential write per video.
- Playback: many sequential reads; users can seek to arbitrary minutes (random/direct access).
- Peak demand: first week; popular videos should be cached.
- Example measurable targets: playback begins within 2 seconds for cached content; seek response under 1 second; no dropped playback at the target concurrent-user count. Validate these targets using a test workload.

### Stage 2 – Architecture
Use a hybrid design: large HDD storage for the full video library, SSDs for metadata and popular-video cache, and optional cloud storage for off-site durability or overflow. Use a Linux filesystem such as XFS/ext4 on servers. Store videos as immutable files and use indexed/extent-based allocation. Build a time-to-byte-offset index for seeking; serve videos in chunks.

### Stage 3 – Decisions
- Block size: 4 KB filesystem blocks; streaming reads use larger chunks (for example, 1–4 MB) to reduce I/O overhead.
- Scheduler: benchmark `mq-deadline` or `none` for SSDs; use a suitable Linux scheduler for HDD workloads and compare results.
- Cache: keep metadata and the most popular first-week videos/chunks on SSD.
- Replication: maintain at least 2 copies for availability, with a separate backup strategy; choose the final replication factor based on budget and recovery objectives.

### Stage 4 – Example calculations (assumptions)
Assume a library of 10,000 videos averaging 1 GB each: raw library = **10 TB**. With 2 copies, capacity required = **20 TB**, excluding filesystem overhead and cache. If 1,000 concurrent users each need 5 Mbps, aggregate delivery bandwidth = **5,000 Mbps = 5 Gbps**. A time index with 1 entry per 10 seconds enables seeking to a nearby chunk rather than scanning from the beginning. With 4 KB blocks, a 1 MB chunk contains 256 blocks; playback reads chunks sequentially after the seek. Actual throughput and read count depend on encoding, chunk size, index format and storage.

### Stage 5 – Demonstration plan
Compare two configurations using the same video-like files and concurrent read workload:
1. HDD-only, no cache.
2. HDD library + SSD cache for popular videos/chunks.
Record average and 95th-percentile start latency, seek latency, throughput, cache hit ratio and CPU use. Use `fio` or a small program to generate repeatable reads; report measured results and command lines in the repository. Do not claim measurements until you have run them.

### Alternative: Cloud object storage + CDN
Store videos in cloud object storage and distribute them through a CDN. Students in different cities retrieve content from nearby edge locations, which can reduce playback-start and seek latency compared with distant origin storage. Monthly cost includes stored GB/TB, requests and outbound data transfer/CDN usage. Operations become simpler because hardware maintenance and scaling are handled by providers, but billing, access policies, network dependence and provider configuration must be monitored. Estimate costs using the actual library size and monthly viewing traffic before choosing.

---

## GitHub Repository Checklist
- Repository name: `CSA04-<RegisterNumber>-Assignment4`
- Keep each task in its own folder (`Task01` … `Task06`).
- Add C source files, commands, outputs/screenshots and README.
- Record your actual GCC version and Linux distribution/version.
- Compile C programs using `gcc -std=c11 -Wall -Wextra` (add `-pthread` only when required).
- Commit each task separately as you complete it; include your real measurements and terminal evidence.
