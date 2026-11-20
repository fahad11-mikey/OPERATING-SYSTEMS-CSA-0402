# Operating Systems – Assignment 3 (CO3)
## Memory Management: Linking, Paging, Segmentation, Virtual Memory, Page Replacement and Disk Management

> **Note:** Sample measurements and screenshots must be collected on your own Linux system. Replace the sample/analytical values where the assignment requires experimental evidence.

## Q1 – Linking, Loading and Address Binding (6 Marks)

### 1. Static and dynamic linking
- **Static linking:** library code is copied into the executable. It is larger, but can run without the matching shared library installed.
- **Dynamic linking:** shared-library code is loaded at run time. Executables are smaller and libraries can be updated independently; the library must be available.

Example build:
```bash
gcc main.c -o static_app -static
gcc main.c -o dynamic_app
ls -lh static_app dynamic_app
ldd dynamic_app
```

Run-time loading uses `dlopen()` to open a library and `dlsym()` to locate a function. Compile with `-ldl`. The program can inspect its mappings using `/proc/self/maps`.

### 2. Address binding
- **Compile time:** absolute addresses are known during compilation; rebuilding is needed if the location changes.
- **Load time:** addresses are fixed when the program is loaded.
- **Execution time:** address translation occurs while running, allowing relocation.

The **linker** combines object files and resolves symbols. The **loader** maps the executable and required libraries into memory. Position-independent code (PIC) can be loaded at different addresses, supporting shared libraries and address-space layout randomisation.

### 3. Evaluation
| Measure | Static | Dynamic |
|---|---|---|
| Executable size | Larger | Smaller |
| Memory | Library code duplicated across executables | Shared library pages can be shared |
| Startup | May avoid library resolution | Loads/resolves required libraries |
| Updating library | Usually rebuild/redeploy | Library can be updated independently |

**Decision:** use dynamic linking for the bank's common encryption library to simplify security updates and reduce disk/memory duplication. Keep deployment version control and testing to avoid incompatible library updates.

### Alternative: `dlopen()` only when needed
Load the encryption library only when encryption is requested. This reduces memory use when the feature is unused, but adds loading delay at first use and requires error handling for a missing library/function. Complexity of symbol lookup is typically constant-time on average for hash-based lookup; memory use is proportional to the loaded library and its symbols.

---

## Q2 – Paging (8 Marks)

### 1. External fragmentation
Contiguous allocation can leave scattered free holes. A process may fail to fit into any single hole even if total free memory is sufficient. Paging places pages into any available frames, eliminating external fragmentation but leaving possible **internal fragmentation** in the last page.

### 2. Address translation
For page size 1024 bytes:
- page number `p = logical_address / 1024`
- offset `d = logical_address % 1024`
- physical address `frame[p] * 1024 + d`

Page table: `{5, 2, -1, 7, 0, -1, 3, -1}`.

| Logical | Page | Offset | Result |
|---:|---:|---:|---|
| 0 | 0 | 0 | 5120 |
| 1500 | 1 | 476 | 2524 |
| 3100 | 3 | 28 | 7196 |
| 6200 | 6 | 56 | 3128 |
| 9000 | 8 | 808 | Invalid address |
| 2049 | 2 | 1 | Page fault |

### 3. Page-table size (4 KB pages)
A 4 KB page is `2^12` bytes.
- 32-bit address: `2^(32-12) = 2^20 = 1,048,576` entries.
- 48-bit address: `2^(48-12) = 2^36` entries.
With 4-byte entries, sizes are 4 MiB and 256 GiB respectively for a single-level table; multi-level tables allocate only needed portions.

### 4. TLB effective access time
Let memory access time be `m`, TLB hit ratio `h`, and TLB lookup time `t`. With a TLB, EAT is approximately `h(t+m) + (1-h)(t+2m)`. Without a TLB, a single-level page table requires approximately `2m` per memory reference (page-table read + data read).

### 5. Page-size comparison
| Page size | Advantages | Disadvantages |
|---|---|---|
| 1 KB | Less internal waste | More page-table entries |
| 4 KB | Good general balance | More table overhead than huge pages |
| 2 MB | Fewer entries, efficient large-memory access | More internal waste, less flexible |

Multi-level tables reduce table memory; hashed page tables suit large address spaces; inverted tables use one entry per physical frame. Paging supports protection bits and shared read-only pages.

### 6. Internal fragmentation estimate
For a process of size `S` and page size `P`, wasted bytes are `(ceil(S/P)*P)-S`. For 100 random-sized processes, sum this expression for all process sizes and divide by 100 for average waste. Record your generated sizes and results.

### 7. Complete code (address translation)
```c
#include <stdio.h>
#define PAGE_SIZE 1024
#define PAGES 8
int page_table[PAGES] = {5, 2, -1, 7, 0, -1, 3, -1};
int main(void) {
    unsigned logical[] = {0,1500,3100,6200,9000,2049};
    for (int i=0;i<6;i++) {
        unsigned a=logical[i], p=a/PAGE_SIZE, d=a%PAGE_SIZE;
        printf("%u -> page %u offset %u : ",a,p,d);
        if (p>=PAGES) puts("invalid address");
        else if (page_table[p]==-1) puts("page fault");
        else printf("physical %u\n",page_table[p]*PAGE_SIZE+d);
    }
}
```
With `PAGE_SIZE=2048`, recompute page and offset using the new size before running. **Conclusion:** 4 KB is a practical general-purpose choice for a multi-user college server; larger pages can help selected workloads.

### Alternative: inverted page table
An inverted table stores one entry per physical frame, so space is proportional to physical RAM rather than the virtual address spaces of all processes. Translation requires a hash/search for process ID and virtual page; hash lookup is average O(1), while a multi-level page-table walk is O(levels). Inverted tables save table memory but add lookup complexity.

---

## Q3 – Segmentation (6 Marks)

### 1. Translation and protection
A segment table stores base and limit. For `(segment, offset)`, if `offset >= limit`, raise a bounds fault; otherwise physical address is `base + offset`. Each segment can have permissions such as read, write and execute. Code can be read-only/shared, while stack and data are private.

### 2. Segmentation with paging
Logical address → segment table (segment base/limit) → segment page table → physical frame + offset. This combines logical segments with fixed-size pages.

### 3. Comparison
| Scheme | Fragmentation | Table overhead | Sharing/protection |
|---|---|---|---|
| Pure paging | Internal only | Page tables | Page-level |
| Pure segmentation | External | Segment table | Natural segment-level |
| Segmented paging | Internal in pages | Segment + page tables | Flexible |

Segmentation matches the programmer's logical view (code, data, stack), but external fragmentation makes allocation difficult. Modern OSs mainly use paging because it allocates fixed-size frames efficiently and simplifies physical memory management.

### 4. Experiment plan
Translate 8 addresses using base/limit values, verify with a small program, deliberately access invalid memory to observe a segmentation fault, and record segment sizes/fragmentation. Linux `/proc/<pid>/maps` shows mapped regions and permissions. Pure paging alternative removes external fragmentation, but logical segment boundaries and sharing may need to be represented through mappings and permissions.

---

## Q4 – Virtual Memory and Demand Paging (8 Marks)

### 1. Page-fault steps
1. CPU references a page not resident; MMU raises a page fault.
2. OS checks whether the address is valid and permitted.
3. Locate the page on storage and obtain a free frame (or choose a victim).
4. Read page into frame and update page table/TLB.
5. Restart the faulting instruction.

**Demand paging** allows a 24 GB application to run on a 16 GB RAM machine if its active working set fits and inactive pages can be stored; performance may deteriorate if paging is frequent.

### 2. Effective access time
Memory = 100 ns; fault service = 8 ms = 8,000,000 ns. EAT ≈ `(1-p)*100 + p*8,000,000` ns.
| Fault rate | EAT (approx.) |
|---|---:|
| 1/1,000 | 8,100 ns |
| 1/100,000 | 180 ns |
| 1/1,000,000 | 108 ns |

For a 10% slowdown, `EAT <= 110 ns`; solve `100 + p(8,000,000-100) <= 110`, so `p <= 1.25 × 10^-6` (approximately one fault per 800,000 accesses).

### 3. Copy-on-write and thrashing
`fork()` initially shares pages; a write causes a private copy, making fork faster and saving memory. Thrashing occurs when processes spend more time paging than executing. The working-set model estimates recently used pages; if total working sets exceed RAM, reduce multiprogramming or add memory.

### 4. Allocation and conclusion
Equal allocation gives each process the same frames; proportional allocation gives frames according to process size. OS uses fault rates/working sets to balance. For the radiologist, improve locality and use sufficient RAM for the active image working set; adding RAM is usually more reliable than tolerating continuous faults.

**Experiment:** write a sequential/random memory-touch test, count faults using `getrusage()` or `/usr/bin/time -v`, limit memory in a VM/cgroup, and plot faults against frames. A 32 GB machine can largely avoid paging if the workload fits, but extra RAM costs money; compare its cost with measured slowdown.

---

## Q5 – Page-Replacement Techniques (8 Marks)

### 1. Algorithms
- **FIFO:** remove the oldest loaded page; simple but may suffer Belady's anomaly.
- **LRU:** remove the page least recently used; uses recent access history.
- **Optimal:** remove the page whose next use is farthest in the future; theoretical benchmark, not implementable online.
- **Clock/second chance:** circular pointer and reference bit; clears set bits and replaces the first page with bit 0.

### 2. Completed LRU function
```c
int lru(const int ref[], int len, int frames) {
    int f[MAXF], last[MAXF], used=0, faults=0;
    for (int i=0;i<len;i++) {
        int k=find(f,used,ref[i]);
        if (k>=0) { last[k]=i; continue; }
        faults++;
        if (used<frames) {
            f[used]=ref[i]; last[used]=i; used++;
        } else {
            int victim=0;
            for (int j=1;j<frames;j++) if (last[j]<last[victim]) victim=j;
            f[victim]=ref[i]; last[victim]=i;
        }
    }
    return faults;
}
```

### 3. Expected results
Reference string: `7,0,1,2,0,3,0,4,2,3,0,3,2,1,2,0,1,7,0,1`.
| Frames | FIFO | LRU | Optimal |
|---:|---:|---:|---:|
| 3 | 15 | 12 | 9 |
| 4 | 10 | 8 | 8 |

Belady string `1,2,3,4,1,2,5,1,2,3,4,5`: FIFO faults are 9 with 3 frames and 10 with 4 frames. FIFO's queue order can worsen with an extra frame. LRU has the stack property: its resident set with fewer frames is a subset of that with more frames, so it does not exhibit Belady's anomaly.

### 4. Complexity and decision
Naive FIFO/LRU simulations with arrays can take O(nf) for n references and f frames; LRU timestamp replacement scans frames, while an optimized hash + linked list can achieve O(1) average per reference. Optimal's look-ahead is O(nf) in this implementation. Clock is O(n) amortized over references and uses one reference bit per frame. Recommend LRU-like/Clock for a server because it adapts to locality with practical overhead; validate with the workload's fault measurements.

---

## Q6 – Disk Management (4 Marks)

### 1. Formatting and boot
Low-level formatting prepares physical sectors; partitioning divides the disk; logical formatting creates a filesystem. Boot block contains bootstrapping code. Firmware starts bootloader, bootloader loads the kernel, and kernel initializes hardware and mounts the root filesystem. Bad blocks are remapped/avoided by device firmware or filesystem tools.

### 2. RAID comparison
| RAID | Usable capacity (6 × 4 TB) | Fault tolerance | Performance |
|---|---:|---|---|
| 0 | 24 TB | None | High throughput |
| 1 | 12 TB (3 mirrored pairs) | One disk per mirror pair | Good reads, writes duplicated |
| 5 | 20 TB | 1 disk | Good reads, parity-write overhead |
| 10 | 12 TB | Multiple failures if in different mirror pairs | Strong performance |

For hospital records, prefer RAID 10 for predictable performance and resilience, plus separate backups. RAID is not a backup. Swap may be a dedicated partition or swap file; a swap file is easier to resize, while a separate disk can avoid competing I/O but adds hardware.

### 3. Linux checks and experiment
```bash
lsblk
sudo fdisk -l
swapon --show
```
Capacity for 6 × 4 TB: RAID 0 = 24 TB; RAID 1 with three pairs = 12 TB; RAID 5 = `(6-1)*4` = 20 TB; RAID 10 = `(6/2)*4` = 12 TB. Actual usable capacity is lower due to filesystem/format overhead and TB/TiB differences. Measure SSD/HDD with a controlled test such as `fio` and record settings/results.

### 4. RAID 6 alternative
RAID 6 uses two parity blocks per stripe and tolerates any two disk failures. Six 4 TB drives provide `(6-2)*4 = 16 TB` usable capacity. Writes incur extra parity computation and I/O, but protection is stronger than RAID 5. Choose RAID 6 where rebuild risk and uptime outweigh write-performance cost.

---

## Q7 – Memory-Management Plan for a Smart-Factory Edge Device (10 Marks)

### Stage 1: Requirements
- Device RAM: 2 GB; no swap.
- Real-time control: strict deadline, must not page or be delayed by memory faults.
- ML task: 1.5 GB, throughput-oriented and may tolerate delays.
- Logging: low priority, variable memory use.
- Measure control-loop worst-case latency and memory use; reserve memory for OS and future tasks.

### Stage 2: Compare mechanisms
| Mechanism | Control task | ML task | Logging |
|---|---|---|---|
| Contiguous | Predictable but hard to allocate | Large contiguous request difficult | Simple, can fragment |
| Paging | Flexible; page faults unsuitable | Suitable | Suitable |
| Segmentation | Logical isolation | Logical regions | Separate buffers |
| Demand paging | Avoid for deadline task | Useful if locality fits | Useful |

Lock critical memory with `mlock()`/`mlockall()` after reserving enough RAM and prefaulting pages. Keep control task memory bounded. Avoid swapping and overcommit assumptions.

### Stage 3: Verify (example C program)
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <time.h>
int main(void) {
    const size_t n=4*1024*1024;
    char *p=calloc(n,1);
    if(!p) return 1;
    if(mlock(p,n)!=0) perror("mlock");
    struct timespec a,b;
    clock_gettime(CLOCK_MONOTONIC,&a);
    for(size_t i=0;i<n;i+=4096) p[i]=1;
    clock_gettime(CLOCK_MONOTONIC,&b);
    printf("Touched %zu bytes; elapsed %.6f seconds\n",n,
      (b.tv_sec-a.tv_sec)+(b.tv_nsec-a.tv_nsec)/1e9);
    munlock(p,n); free(p);
}
```
Compile: `gcc -O2 memory_lock.c -o memory_lock`. Run and compare with a version without `mlock`; record actual timings. This simple test is not a certified real-time benchmark. Simulate frame use for control, ML and logging and record faults as frame counts vary.

### Stage 4: Decision
Reserve RAM for Linux and control task; lock/prefault the control task and use fixed-size buffers. Give ML a controlled memory budget and logging bounded buffers with low priority. A fourth 500 MB task exceeds the comfortable free-memory budget and requires reducing ML allocation, redesigning tasks, or upgrading RAM. Limitation: Linux with locked memory still cannot guarantee hard real-time deadlines under all hardware/interrupt conditions.

### Alternative: RTOS without virtual memory
An RTOS can provide more predictable control-task latency and avoid paging, but memory is fixed and the 1.5 GB ML workload may be constrained. Adding tasks requires careful static allocation and scheduling analysis. Linux with locked pages offers flexible applications but less deterministic worst-case latency.

---

## Submission checklist
- [ ] Run programs on your Linux system and capture real output/screenshots.
- [ ] Record your own measurements, system details and plots.
- [ ] Upload source files, commands, results and this README to GitHub.
- [ ] Explain each algorithm and design choice during verification.
