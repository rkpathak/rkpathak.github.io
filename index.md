---\ntitle: Linux File I/O in Practice\n---
# Linux File I/O: A Practical Reference

How to read and write files on Linux, which API to pick, and what can go wrong.

**How to use this guide**
- Need a quick answer? Read [Pick a method](#1-pick-a-method-in-30-seconds), then jump to that method.
- Each method follows the same layout: *What it is → Example → Use it when → Pros → Cons*.
- Writing anything that must survive a crash? Read [Durability](#3-durability-when-is-data-really-safe) first. It is the most commonly misunderstood topic.

**Contents**
1. [Pick a method](#1-pick-a-method-in-30-seconds)
2. [The big picture](#2-the-big-picture)
3. [Durability](#3-durability-when-is-data-really-safe)
4. [The methods](#4-the-methods) (stdio, read/write, mmap, O_DIRECT, POSIX AIO, Linux AIO, io_uring, zero-copy)
5. [Recipes](#5-recipes) (atomic update, appending, cache hints, preallocation, locking)
6. [Tuning and observability](#6-tuning-and-observability)
7. [Comparison matrix and kernel versions](#7-comparison-matrix-and-kernel-versions)
8. [Common mistakes checklist](#8-common-mistakes-checklist)

---

## 1. Pick a method in 30 seconds

| You are building... | Start with | Why |
|---|---|---|
| A simple CLI tool, config or text processing | stdio (`fopen`/`fwrite`) | Simple, buffered, portable |
| A log writer (many processes) | `open` + `O_APPEND` + `write` | Appends don't overwrite each other |
| Multi-threaded access to one file | `pread`/`pwrite` | No shared file offset |
| Read-mostly lookup table or index | `mmap` | Random access with no syscall per lookup |
| Copying files | `copy_file_range` (or `cp --reflink`) | Stays in the kernel; may be instant |
| Serving static files over a socket | `sendfile` | No user-space copy |
| Database or storage engine with its own cache | `O_DIRECT` + `pwrite`/`io_uring` | Skip the page cache you don't need |
| Thousands of concurrent I/Os, NVMe | `io_uring` | Batched, truly async |
| Portable async with no Linux-only APIs | POSIX AIO | Works, but it is thread-based |

**Rule of thumb:** start with stdio, move to `read`/`write` when you need control, and reach for `mmap`, `O_DIRECT` or `io_uring` only when measurements show you need them.

---

## 2. The big picture

```
  Your application        fwrite / write / mmap / io_uring
  ─────────────────────────────────────────────────────────  user space
  C library (glibc)       stdio buffer (usually 4 KB for files)
  ─────────────────────────────────────────────────────────  syscall boundary
  VFS                     inodes, dentries, fd table
  Page cache              dirty pages wait here for write-back
  Filesystem              ext4 / xfs / btrfs (journal, allocation)
  Block layer             I/O scheduler, request merging
  Device driver           NVMe / SCSI / virtio
  Hardware                device write cache (volatile!) -> media
```

Three facts to remember:

1. **`write()` returns when data reaches the page cache, not the disk.** A crash or power loss can lose it. See [Durability](#3-durability-when-is-data-really-safe).
2. **The page cache is shared and automatic.** It gives you free caching and read-ahead. It also uses RAM and can add latency (write-back bursts).
3. **The fastest I/O is the I/O you don't do.** Batch small writes, cache in your application, and tell the kernel your access pattern (see [cache hints](#cache-hints-fadvise-and-readahead)).

Also note: small I/O always costs a full block. A 1-byte write to an uncached page means the kernel reads a whole 4 KB page, changes 1 byte, and later writes 4 KB back. SSDs have a similar effect at the flash-page level (write amplification).

---

## 3. Durability: when is data really safe?

### The sync functions

| Call | What it does | Cost | Notes |
|---|---|---|---|
| `fsync(fd)` | Flushes this file's data **and** metadata, waits for the device | Medium | The default choice |
| `fdatasync(fd)` | Flushes data and only the metadata needed to read it (e.g. file size) | Lower | Skips timestamp updates. Good for overwrites |
| `syncfs(fd)` | Flushes everything on that filesystem | High | |
| `sync()` | Flushes everything, system-wide | Very high | |
| `sync_file_range()` | Starts or waits for write-back of a byte range | Low | **No durability guarantee**: no metadata flush, no device-cache flush. Use only as a throttling or ordering aid |
| `O_SYNC` / `O_DSYNC` | Makes *every* write behave like `fsync` / `fdatasync` | High | Simple but slow |

### Rules that prevent data loss

1. **`fsync` returning 0 is the only proof of persistence.** `write()`, `fflush()`, `fclose()`, `munmap()` and `O_DIRECT` alone prove nothing.
2. **After creating or renaming a file, `fsync` the directory too.** Otherwise the file's data may be safe but its name may not be.
3. **If `fsync` fails, do not retry it and assume success.** On Linux, after a failed `fsync` the kernel may mark the dirty pages clean, so a second `fsync` can return 0 even though the data was lost (the "fsyncgate" problem, famous from PostgreSQL). Treat the failure as fatal for that file: report it, and rebuild from a known-good copy or log.
4. **Errors can show up late.** Write errors may only appear at `fsync()` or `close()` (especially on NFS), so always check both.
5. **The device matters.** Drives with a volatile write cache honor `fsync` by issuing a flush, which is slow. SSDs with power-loss protection make flushes cheap. Never mount with barriers disabled unless you accept the risk.

### Write-back timing (so you can predict behavior)

- Dirty pages are flushed by kernel flusher threads. They wake every 5 s (`vm.dirty_writeback_centisecs`).
- A page is normally written once it is about 30 s old (`vm.dirty_expire_centisecs`), or sooner if dirty memory crosses `vm.dirty_background_ratio` (default 10%). Past `vm.dirty_ratio` (default 20%), writers get **throttled**, which is the cause of many "random" latency spikes.

---

## 4. The methods

### 4.1 Standard I/O (stdio): `fopen` / `fread` / `fwrite` / `fclose`

**What it is.** The C library's buffered I/O. It collects small writes in a user-space buffer and issues one `write()` syscall when the buffer fills or you flush. For regular files the buffer size is the filesystem's block size (normally 4 KB), not the 8 KB `BUFSIZ`.

```
  fwrite() -> stdio buffer -> (full / fflush) -> write() -> page cache -> disk
```

**Example**

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    FILE *fp = fopen("stdio_demo.txt", "w");
    if (!fp) { perror("fopen"); return 1; }

    /* Optional: bigger buffer. It must stay valid until fclose(). */
    static char buf[64 * 1024];
    setvbuf(fp, buf, _IOFBF, sizeof buf);    /* _IOFBF full, _IOLBF line, _IONBF none */

    for (int i = 0; i < 1000; i++)
        fprintf(fp, "Line %04d\n", i);       /* no syscall for most of these */

    /* fflush moves data to the kernel only. For durability: fflush + fsync. */
    if (fflush(fp) != 0 || fsync(fileno(fp)) != 0)
        perror("flush/fsync");

    /* fclose can fail (final flush). Always check it. */
    if (fclose(fp) != 0) { perror("fclose: data may be lost"); return 1; }
    return 0;
}
```

**Use it when**
- Logs, config files, CSV/JSON, report generation, text processing.
- Moderate-size sequential reads and writes where simplicity matters.
- You want code that is portable beyond Linux.

**Pros**
- Simplest API; buffering is automatic, so far fewer syscalls for small writes.
- Formatted I/O built in (`fprintf`, `fscanf`, `getline`).
- Thread-safe: glibc locks each `FILE`. (Use `flockfile` to keep several calls together, or the `_unlocked` variants for speed.)
- Portable ISO C.

**Cons**
- No direct control over durability: you need `fflush` then `fsync(fileno(fp))`.
- Error reporting is coarse (a sticky `ferror()` flag); partial failures are hard to locate.
- Unflushed data in the buffer is **lost if the process crashes** or is killed.
- No async I/O, and `O_DIRECT` is awkward (you would have to `open()` first and use `fdopen()`).
- Not suited to concurrent appends from multiple processes. One logical record may be split across several `write()` calls. Use `O_APPEND` with a single `write()` per record instead.

---

### 4.2 POSIX I/O: `open` / `read` / `write` / `close`

**What it is.** The direct syscall interface. No user-space buffering: each call is one syscall.

**Key `open` flags**

| Flag | Meaning |
|---|---|
| `O_CREAT`, `O_TRUNC`, `O_EXCL` | Create, truncate, fail if exists |
| `O_APPEND` | Every write goes to the end of file (atomically) |
| `O_SYNC` / `O_DSYNC` | Each write waits for storage (like `fsync` / `fdatasync`) |
| `O_DIRECT` | Bypass the page cache ([see 4.4](#44-direct-io-o_direct)) |
| `O_CLOEXEC` | Don't leak the fd to child processes. **Use it by default** |
| `O_NOATIME` | Don't update access time on reads (saves metadata writes) |
| `O_TMPFILE` | Create an unnamed temp file (see [atomic update](#atomic-file-update)) |

**Example: a robust write loop.** `write()` may write fewer bytes than asked (signals, disk full, pipes and sockets), and may fail with `EINTR`.

```c
#include <errno.h>
#include <unistd.h>

/* Returns 0 on success, -1 on error (errno set). */
int write_all(int fd, const void *buf, size_t len)
{
    const char *p = buf;
    while (len > 0) {
        ssize_t n = write(fd, p, len);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        p   += n;
        len -= (size_t)n;
    }
    return 0;
}
```

```c
int fd = open("data.bin", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
if (fd < 0) { perror("open"); return 1; }
if (write_all(fd, data, len) < 0 || fsync(fd) < 0) perror("write/fsync");
if (close(fd) < 0) perror("close");          /* close can report deferred errors */
```

#### Positional and vectored variants (same method, more control)

| Call | What it adds |
|---|---|
| `pread` / `pwrite` | Read/write at an explicit offset. Does not touch the shared file offset, so threads can use one fd safely without `lseek` races |
| `readv` / `writev` | Gather/scatter: one syscall for a header plus a body, no extra copy |
| `preadv` / `pwritev` | Both of the above |
| `preadv2` / `pwritev2` (4.6+) | Per-call flags: `RWF_NOWAIT` (fail with `EAGAIN` instead of blocking), `RWF_DSYNC`, `RWF_SYNC`, `RWF_APPEND`, `RWF_HIPRI` (polled I/O) |

```c
ssize_t n = pread(fd, buf, len, offset);      /* offset is NOT changed */
ssize_t m = pwrite(fd, buf, len, offset);
```

Caveat: on Linux, `pwrite` on an fd opened with `O_APPEND` **ignores the offset and appends**.

**Use it when**
- You need exact control over caching, flags and when syscalls happen.
- Binary formats, databases, storage code, multi-threaded file access (`pread`/`pwrite`).
- Multi-process logging (`O_APPEND`).

**Pros**
- Full flag control; nothing hidden.
- `pread`/`pwrite` are thread-safe and lock-free for different regions.
- It is the base layer that mmap, `O_DIRECT`, AIO and `io_uring` build on.

**Cons**
- Small I/O means one syscall each; you must batch yourself.
- You must handle short reads/writes and `EINTR`.
- No formatted I/O. You do your own serialization.
- Manual fd lifetime (leaks and double-closes are common bugs).
- **`epoll`/`poll`/`O_NONBLOCK` don't help with regular files.** They are always "ready" and reads can still block on disk. This is why AIO and `io_uring` exist.

---

### 4.3 Memory-mapped I/O: `mmap`

**What it is.** The file appears as ordinary memory. Touching a page triggers a page fault, and the kernel loads it from the page cache (reading disk if needed). Later accesses to that page need no syscall.

```
  ptr[i] -> page fault (first touch) -> kernel loads page -> mapped into your process
```

**Example**

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

int main(void)
{
    size_t size = 100 * 4096;
    int fd = open("mmap_demo.dat", O_RDWR | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    /* fallocate (not ftruncate) reserves real blocks. With a sparse file, a write
       through the mapping can hit "disk full" and kill you with SIGBUS. */
    if (fallocate(fd, 0, 0, size) < 0) { perror("fallocate"); return 1; }

    char *map = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (map == MAP_FAILED) { perror("mmap"); return 1; }

    strcpy(map, "Hello from mmap\n");              /* a plain memory write */

    madvise(map, size, MADV_SEQUENTIAL);           /* access-pattern hint */

    if (msync(map, size, MS_SYNC) < 0) perror("msync");   /* persist the mapping */
    munmap(map, size);                                    /* does NOT sync by itself */
    close(fd);
    return 0;
}
```

**Useful flags and hints**

| Flag | Meaning |
|---|---|
| `MAP_SHARED` | Writes go to the file and are visible to other processes |
| `MAP_PRIVATE` | Copy-on-write; changes are never written back |
| `MAP_POPULATE` | Pre-load all pages at map time (reads the whole file!) |
| `MADV_RANDOM` / `MADV_SEQUENTIAL` | Turn read-ahead off / up |
| `MADV_WILLNEED` | Start reading these pages into the page cache |
| `MADV_DONTNEED` | Drop the mapping's pages (for private anonymous memory this discards data) |
| `MADV_POPULATE_READ` (5.14+) | Pre-fault pages without the cost of touching them |
| `MAP_SYNC` (4.15+, DAX/persistent memory only) | Makes CPU cache flushes enough for durability |

**Handling SIGBUS.** Accessing a mapped page that has no backing store raises `SIGBUS`, not an error code. Causes: another process truncated the file; an I/O error; or disk full on a sparse file. If you map files you don't fully control, install a handler:

```c
#include <setjmp.h>
#include <signal.h>

static sigjmp_buf jb;
static void on_sigbus(int sig) { (void)sig; siglongjmp(jb, 1); }

struct sigaction sa = { .sa_handler = on_sigbus };
sigemptyset(&sa.sa_mask);
sigaction(SIGBUS, &sa, NULL);

if (sigsetjmp(jb, 1) == 0) {
    volatile char c = map[offset];     /* volatile: the read must really happen */
    (void)c;
} else {
    fprintf(stderr, "SIGBUS: file truncated or I/O error\n");
}
```

**Use it when**
- Random access to large read-mostly files: lookup tables, dictionaries, indexes.
- Several processes share data through a file (`MAP_SHARED`).
- Reading a file many times (repeated access is just memory access).

**Pros**
- No user-space copy and no syscall per access once pages are mapped.
- Kernel does caching, read-ahead and eviction for you.
- Natural fit for pointer-like data structures stored in a file.

**Cons**
- **Errors arrive as signals**, not return codes (see SIGBUS above).
- **No control over write-back order.** You can't say "write this page before that one", which makes crash-consistent updates hard. `msync`/`fsync` is your only tool.
- Page faults and TLB invalidations have a cost. On fast NVMe and many cores, mmap can be *slower* than `pread` (see the paper "Are You Sure You Want to Use MMAP in Your Database Management System?").
- Latency spikes: any memory access might block on disk.
- Not ideal for files that grow often (use `mremap` or remap).
- Address-space limits: about 2-3 GB on 32-bit; `vm.max_map_count` (default 65530) limits the number of mappings.
- For a pure sequential scan, `read()` or `pread()` with a large buffer is usually as fast or faster.
- Don't mix with `O_DIRECT` on the same file.
- `MAP_HUGETLB` only applies to anonymous or hugetlbfs mappings, not ordinary files.

---

### 4.4 Direct I/O: `O_DIRECT`

**What it is.** The kernel transfers data straight between your buffer and the device, skipping the page cache. You become responsible for caching.

```
  Normal:  app buffer -> page cache -> disk
  Direct:  app buffer ---------------> disk
```

**The three alignment rules.** Buffer address, transfer length and file offset must all be aligned to the device's logical block size (often 512 B) or, safely, the filesystem block size (4 KB). Violations return `EINVAL`. On Linux 6.1+, ask the kernel with `statx(..., STATX_DIOALIGN, ...)`.

**Example**

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    /* Use a real disk filesystem. tmpfs rejects O_DIRECT before Linux 6.6. */
    int fd = open("direct_demo.dat",
                  O_WRONLY | O_CREAT | O_TRUNC | O_DIRECT | O_DSYNC | O_CLOEXEC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    void *buf;
    if (posix_memalign(&buf, 4096, 4096) != 0) { fputs("alloc failed\n", stderr); return 1; }
    memset(buf, 'A', 4096);

    ssize_t n = pwrite(fd, buf, 4096, 0);        /* aligned buffer, length, offset */
    if (n < 0) perror("pwrite"); else printf("wrote %zd bytes\n", n);

    free(buf);
    close(fd);
    return 0;
}
```

`O_DIRECT` alone does **not** guarantee durability. The data may sit in the device's write cache, and file-size or allocation metadata may not be flushed. Add `O_DSYNC` (as above) or call `fdatasync`.

Direct I/O is sometimes silently handled as buffered I/O (for example extending writes, or data that is compressed or inline on btrfs). Don't assume every call really bypasses the cache.

**Use it when**
- You have your own buffer cache (InnoDB, Oracle, many storage engines). **Note:** PostgreSQL by default does *not* use direct I/O; it relies on the page cache.
- Large streaming transfers you will not re-read (backup ingest).
- Benchmarking real device speed.
- You want predictable memory use and no write-back storms.

**Pros**
- No double caching; predictable memory and latency.
- The right model for software that manages its own cache.

**Cons**
- Alignment rules are strict and vary by device and filesystem.
- Small or unaligned I/O is slow: no merging and no read-ahead.
- Still needs `fsync`/`O_DSYNC` for durability.
- Behavior differs by filesystem. Some reject it (`EINVAL` at `open`) or fall back to buffered.
- Don't mix with `mmap` or buffered access to the same file.

---

### 4.5 POSIX AIO: `aio_read` / `aio_write` (glibc)

**What it is.** A portable async API. In glibc it is **implemented with a user-space thread pool** that runs ordinary blocking calls, not kernel-level async I/O.

**Example**

```c
#include <aio.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int fd = open("aio_demo.dat", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    static char data[] = "Async hello\n";       /* must stay valid until completion */
    struct aiocb cb;
    memset(&cb, 0, sizeof cb);
    cb.aio_fildes = fd;
    cb.aio_buf    = data;
    cb.aio_nbytes = strlen(data);
    cb.aio_offset = 0;

    if (aio_write(&cb) < 0) { perror("aio_write"); return 1; }

    /* Wait without spinning. */
    const struct aiocb *list[1] = { &cb };
    while (aio_error(&cb) == EINPROGRESS)
        aio_suspend(list, 1, NULL);

    int err = aio_error(&cb);
    if (err) fprintf(stderr, "aio: %s\n", strerror(err));
    else     printf("wrote %zd bytes\n", aio_return(&cb));   /* call exactly once */

    close(fd);
    return 0;
}
/* Build: gcc aio_demo.c -o aio_demo   (add -lrt on glibc older than 2.34) */
```

Other calls: `lio_listio` (batch), `aio_fsync`, `aio_cancel`.

**Use it when**
- You need async-looking file I/O in code that must stay portable across POSIX systems, and performance is not critical.

**Pros**
- Standard API, no extra libraries.
- Batch submission with `lio_listio`; completion by signal, thread callback or polling.

**Cons**
- Not truly async: thread-pool overhead, and many requests can mean many threads.
- Awkward error handling (`aio_error` then `aio_return`).
- The buffer and `aiocb` must stay valid until completion.
- Largely superseded by `io_uring` on Linux. Treat it as legacy.

---

### 4.6 Linux native AIO: `io_submit` / `io_getevents` (libaio)

**What it is.** Kernel-level async I/O with no thread pool, but practical only for **`O_DIRECT`** files. Needs `libaio` (`libaio-dev`).

**Example.** Important: libaio functions return **`-errno`** and do not set `errno`.

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <libaio.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int fd = open("kaio_demo.dat", O_WRONLY | O_CREAT | O_TRUNC | O_DIRECT, 0644);
    if (fd < 0) { perror("open"); return 1; }

    void *buf;
    if (posix_memalign(&buf, 4096, 4096) != 0) return 1;
    memset(buf, 'X', 4096);

    io_context_t ctx = 0;
    int rc = io_setup(128, &ctx);
    if (rc < 0) { fprintf(stderr, "io_setup: %s\n", strerror(-rc)); return 1; }

    struct iocb cb, *cbs[1] = { &cb };
    io_prep_pwrite(&cb, fd, buf, 4096, 0);

    rc = io_submit(ctx, 1, cbs);
    if (rc < 0) { fprintf(stderr, "io_submit: %s\n", strerror(-rc)); return 1; }

    struct io_event ev;
    rc = io_getevents(ctx, 1, 1, &ev, NULL);         /* blocks until done */
    if (rc < 0) { fprintf(stderr, "io_getevents: %s\n", strerror(-rc)); return 1; }
    printf("result: %ld bytes\n", (long)ev.res);     /* negative = -errno */

    io_destroy(ctx);
    free(buf);
    close(fd);
    return 0;
}
/* Build: gcc kaio_demo.c -o kaio_demo -laio */
```

**Use it when**
- You already run an `O_DIRECT` storage engine on libaio (for example InnoDB, or Seastar/ScyllaDB in its AIO backend) and want to stay on it.
- For new code, prefer `io_uring`.

**Pros**
- True kernel async; no thread overhead.
- Good throughput for direct I/O workloads.

**Cons**
- Without `O_DIRECT`, `io_submit` quietly **blocks** like a normal syscall.
- Even with `O_DIRECT` it can block on metadata work: extending a file, allocating blocks. Preallocate files (`fallocate`) to avoid this.
- Mostly limited to files (poll support exists since 4.18, but it is awkward).
- Extra library; being replaced by `io_uring`.

---

### 4.7 `io_uring`

**What it is.** The modern Linux async interface (kernel 5.1+). The application and kernel share two ring buffers: a **submission queue (SQ)** for requests and a **completion queue (CQ)** for results. You can queue many operations and submit them with a single syscall.

```
  app -> [SQE][SQE][SQE] -> io_uring_enter() -> kernel works
  app <- [CQE][CQE]      <- results posted by kernel
```

It handles buffered and direct file I/O, sockets, pipes, timers, `fsync`, `accept`, and more.

**Example** (with `liburing`). Like libaio, liburing returns **`-errno`**.

```c
#include <fcntl.h>
#include <inttypes.h>
#include <liburing.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    struct io_uring ring;
    int rc = io_uring_queue_init(32, &ring, 0);
    if (rc < 0) { fprintf(stderr, "queue_init: %s\n", strerror(-rc)); return 1; }

    int fd = open("uring_demo.dat", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    static char data[4096];
    memset(data, 'U', sizeof data);

    /* Queue 10 writes, submit them with ONE syscall. */
    for (int i = 0; i < 10; i++) {
        struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
        if (!sqe) { fputs("SQ full\n", stderr); return 1; }
        io_uring_prep_write(sqe, fd, data, sizeof data, (off_t)i * sizeof data);
        io_uring_sqe_set_data64(sqe, i);               /* tag to match completions */
    }
    io_uring_submit(&ring);

    /* Collect 10 completions (they may arrive in any order). */
    for (int i = 0; i < 10; i++) {
        struct io_uring_cqe *cqe;
        rc = io_uring_wait_cqe(&ring, &cqe);
        if (rc < 0) { fprintf(stderr, "wait_cqe: %s\n", strerror(-rc)); return 1; }
        if (cqe->res < 0)
            fprintf(stderr, "op %" PRIu64 " failed: %s\n",
                    io_uring_cqe_get_data64(cqe), strerror(-cqe->res));
        io_uring_cqe_seen(&ring, cqe);
    }

    io_uring_queue_exit(&ring);
    close(fd);
    return 0;
}
/* Build: gcc uring_demo.c -o uring_demo -luring.  io_uring_prep_write needs Linux 5.6+. */
```

**Notable features**

| Feature | What it gives you |
|---|---|
| Batching | N operations, 1 syscall |
| Linked SQEs (`IOSQE_IO_LINK`) | "Do B only after A succeeds" (e.g. write then fsync) |
| Fixed files / fixed buffers | Lower per-operation overhead (not the same as zero-copy) |
| `SQPOLL` | A kernel thread polls the SQ, so submissions need no syscall. Costs a CPU core; needs privileges before 5.11 |
| `IOPOLL` | Polled completions for `O_DIRECT` on NVMe (lowest latency) |
| Multishot ops | One request produces many completions (e.g. `accept`, `recv`) |
| `URING_CMD` | NVMe passthrough |
| Zero-copy send (`SEND_ZC`) | Send without copying into kernel buffers |

**Use it when**
- High-throughput storage engines, servers, anything issuing very many I/Os, or mixing file and network I/O in one loop.
- Fast NVMe where queue depth matters.

**Pros**
- Truly async; one interface for files, sockets and more.
- Low syscall overhead with batching.
- Actively developed, and the direction Linux I/O is going.

**Cons**
- Needs a recent kernel. Basics from 5.1, but many ops need 5.6+. Features arrive every release, so check your minimum kernel.
- Steeper learning curve. Use `liburing`, not raw syscalls.
- For plain buffered sequential I/O it often gives little over `pread`. Some buffered operations are still run by kernel worker threads internally.
- Larger attack surface. Some environments disable it: Docker's default seccomp profile blocks it, and kernel 6.6+ has the `kernel.io_uring_disabled` sysctl.
- The application must handle completions carefully (buffers must stay valid until the CQE arrives).

---

### 4.8 Moving data without user space: `sendfile`, `splice`, `copy_file_range`, reflink

**What they are.** Calls that move bytes between file descriptors entirely inside the kernel.

```
  read+write copy:   disk -> page cache -> USER BUFFER -> page cache -> disk
  kernel-side copy:  disk -> page cache -----------------> page cache -> disk   (no user buffer)
  sendfile to NIC:   disk -> page cache -> NIC (DMA, with scatter-gather hardware)
```

File to file still involves a CPU copy inside the kernel, unless the filesystem can reflink or the server can copy (see below).

| Call | Moves data between | Notes |
|---|---|---|
| `sendfile` | file to socket (or file to file, 2.6.33+) | Classic static-file serving. Plain sendfile skips TLS unless kTLS is used |
| `splice` | fd to fd, **one end must be a pipe** | Proxies, relays. `vmsplice` and `tee` are companions |
| `copy_file_range` (4.5+) | file to file | Can do a server-side copy on NFS 4.2 / SMB, or reflink on XFS and btrfs |
| `ioctl(FICLONE)` / `FICLONERANGE` | file to file (same filesystem) | Instant copy-on-write clone (XFS, btrfs, bcachefs). `cp --reflink=auto` uses it |
| `ioctl(FIDEDUPERANGE)` | file to file | Share identical blocks |

**Example.** All of these can return **short counts**, so loop. (`sendfile` moves at most about 2 GB per call.)

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/stat.h>
#include <unistd.h>

/* Copy a whole file in the kernel. Returns 0 on success, -1 on error. */
int copy_file(const char *src, const char *dst)
{
    int in = open(src, O_RDONLY | O_CLOEXEC);
    if (in < 0) return -1;
    int out = open(dst, O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, 0644);
    if (out < 0) { close(in); return -1; }

    struct stat st;
    if (fstat(in, &st) < 0) goto fail;

    off_t left = st.st_size;
    while (left > 0) {
        ssize_t n = copy_file_range(in, NULL, out, NULL, (size_t)left, 0);
        if (n < 0) {
            if (errno == EINTR) continue;
            goto fail;       /* EXDEV/ENOSYS/EINVAL: fall back to a read/write loop */
        }
        if (n == 0) break;   /* source shrank */
        left -= n;
    }
    return close(in), close(out);

fail:;
    int e = errno;
    close(in); close(out);
    errno = e;
    return -1;
}
```

**Use it when**
- File copy tools, backup and replication code (`copy_file_range`).
- Web servers sending static files (`sendfile`).
- Relays and proxies (`splice`).

**Pros**
- No user-space buffer; fewer syscalls and context switches.
- `copy_file_range` / reflink can be near-instant and use no extra space.

**Cons**
- Support depends on filesystem and kernel version; `copy_file_range` behavior across different filesystems changed between kernel versions. Handle `EXDEV`/`ENOSYS`/`EINVAL` and fall back to a read/write loop.
- `splice` needs a pipe (default pipe size is 64 KB) and is fiddly.
- The data never passes through your code, so you can't compress, checksum or encrypt on the fly.

---

## 5. Recipes

### Atomic file update

Writing a config file in place with `O_TRUNC` means a crash leaves an empty or partial file. Instead, write a new file and `rename()` it over the old one.

```
 1. Create a temp file in the SAME directory (same filesystem): mkstemp(), or O_TMPFILE + linkat
 2. Copy the original's permissions/owner if needed (fchmod / fchown)
 3. write() the full contents (use write_all)
 4. fsync(temp fd)                       <- data is on disk
 5. close(temp fd)
 6. rename(temp, target)                 <- atomic replace
 7. fsync(open(parent directory))        <- the rename itself is on disk
```

```c
/* Step 7: sync the directory that contains the target file. */
int dfd = open("/etc/myapp", O_RDONLY | O_DIRECTORY | O_CLOEXEC);
if (dfd >= 0) { fsync(dfd); close(dfd); }
```

Variants: `renameat2(RENAME_NOREPLACE)` fails if the target exists. `renameat2(RENAME_EXCHANGE)` swaps two paths atomically.

### Appending (logs)

- Open with `O_APPEND` and write each record with **one** `write()` (or one `writev()`). On local Linux filesystems, concurrent appends from many processes don't overwrite or interleave within a record.
- Don't rely on this over NFS.
- With stdio, records can be split at buffer boundaries. Use `setvbuf` with a buffer larger than your longest record, or flush after every record, or use `write()` directly.

### Cache hints: `fadvise` and `readahead`

```c
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL); /* bigger read-ahead window (pages are NOT dropped) */
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);     /* disable read-ahead */
posix_fadvise(fd, off, len, POSIX_FADV_WILLNEED);/* start reading now; does not block */

/* Free cache after a big one-time write. Dirty pages are NOT dropped, so sync first. */
fdatasync(fd);
posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
```

- `readahead(fd, off, len)` is a Linux-specific call that can **block** until the data is read. `POSIX_FADV_WILLNEED` is the non-blocking equivalent.
- Default read-ahead is 128 KB per device (`/sys/block/<dev>/queue/read_ahead_kb`, or `blockdev --setra`).
- Streaming backups and similar jobs can flush useful data out of the page cache. Hints (or `O_DIRECT`) prevent that.

### Preallocation, holes and sparse files

```c
fallocate(fd, 0, 0, 100 << 20);                 /* reserve 100 MB of real blocks */
fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, off, len);  /* free space, keep size */
fallocate(fd, FALLOC_FL_COLLAPSE_RANGE, off, len);   /* remove a range; file shrinks */
fallocate(fd, FALLOC_FL_INSERT_RANGE,   off, len);   /* insert a hole; file grows */
```

- `ftruncate(fd, size)` only sets the size and creates a **sparse** file (no blocks reserved). Later writes can fail with "disk full".
- `fallocate` needs `_GNU_SOURCE`. `COLLAPSE_RANGE`/`INSERT_RANGE` need block-aligned arguments and extent-based filesystems (ext4, XFS).
- `posix_fallocate` falls back to writing zeros if the filesystem lacks support.
- Find data and holes with `lseek(fd, off, SEEK_DATA)` and `SEEK_HOLE`.
- On 32-bit builds, define `_FILE_OFFSET_BITS=64` to use files over 2 GB.

### File locking

Locks are **advisory**: they only work if every process uses them.

| Call | Notes |
|---|---|
| `flock(fd, LOCK_EX/LOCK_SH)` | Whole-file lock, simple |
| `fcntl(F_SETLK)` (POSIX locks) | Byte ranges, but released when *any* fd for the file is closed by the process, and not per-thread |
| `fcntl(F_OFD_SETLK)` (3.15+) | Byte ranges, tied to the open file description, safe for threads. **Preferred** |

### Security basics

- Open with `O_CLOEXEC` always; add `O_NOFOLLOW` when you don't want to follow symlinks.
- Use `openat`/`openat2` relative to a directory fd to avoid race conditions in path checks (TOCTOU).
- Use `mkstemp` or `O_TMPFILE` for temp files, with `O_EXCL` for creation races.

---

## 6. Tuning and observability

### Tuning knobs

| Setting | Effect |
|---|---|
| `vm.dirty_background_ratio` / `vm.dirty_ratio` | When background and forced write-back start. Lower values mean smaller, smoother bursts |
| `vm.dirty_expire_centisecs`, `vm.dirty_writeback_centisecs` | Age before write-back; flusher wake interval |
| `/sys/block/<dev>/queue/scheduler` | `none` for NVMe/virtio; `mq-deadline` for HDDs and mixed loads; `bfq` for desktops; `kyber` for low-latency targets |
| `/sys/block/<dev>/queue/read_ahead_kb` | Read-ahead size |
| Mount options | `noatime` / `relatime` reduce metadata writes |
| cgroup v2 `io.max`, `memory.max` | In containers, page cache counts against the cgroup's memory, and I/O can be throttled |

### Tools: finding out what your program really does

| Tool | Use |
|---|---|
| `strace -f -T -e trace=file,desc ./prog` | See every syscall, with timings |
| `fio` | Benchmark methods fairly (`ioengine=psync,mmap,libaio,io_uring`, `direct=0/1`) |
| `iostat -x 1` | Device utilization, queue depth, latency |
| `vmstat 1`, `/proc/meminfo` (`Dirty`, `Writeback`) | Dirty page level |
| `perf`, `bpftrace` / `biolatency`, `blktrace` | Per-I/O latency and block-layer detail |
| `cat /proc/<pid>/io` | Per-process bytes read/written |

Example benchmark to compare buffered and direct random reads:

```
fio --name=randread --filename=testfile --size=2G --rw=randread --bs=4k \
    --ioengine=io_uring --iodepth=32 --direct=1 --runtime=30 --time_based
```

Always measure on your real filesystem and hardware. Results change a lot between NVMe, SATA and network storage.

---

## 7. Comparison matrix and kernel versions

| Method | Simplicity | Async | Bypass page cache | Avoids user copy | Min requirement |
|---|---|---|---|---|---|
| stdio | Easiest | No | No | No | Any |
| `read`/`write`, `pread`/`pwrite` | Easy | No | Optional (`O_DIRECT`) | No | Any |
| `mmap` | Easy to start, tricky to do well | No | No | Yes | Any |
| `O_DIRECT` | Hard | No | Yes | Yes (DMA to your buffer) | Any; alignment rules |
| POSIX AIO | Moderate | Thread-based | Optional | No | glibc |
| libaio | Hard | Yes (only with `O_DIRECT`) | Yes | Yes | `libaio` |
| `io_uring` | Moderate with liburing | Yes | Optional | Optional (fixed buffers, `SEND_ZC`) | 5.1 (5.6+ for most ops) |
| `sendfile` / `splice` / `copy_file_range` | Easy | No | No | Yes (no user buffer) | 2.6.17+ / 4.5+ |

**When features arrived**

| Feature | Kernel |
|---|---|
| `splice` | 2.6.17 |
| `fallocate` | 2.6.23 |
| `sendfile` file-to-file | 2.6.33 |
| `O_TMPFILE` | 3.11 |
| `COLLAPSE_RANGE` / `renameat2` / OFD locks | 3.15 |
| `INSERT_RANGE` | 4.1 |
| `copy_file_range` | 4.5 |
| `preadv2` / `pwritev2` | 4.6 |
| `RWF_NOWAIT` | 4.14 |
| `MAP_SYNC` | 4.15 |
| libaio poll | 4.18 |
| `io_uring` | 5.1 |
| `io_uring` read/write ops, `openat2` | 5.6 |
| Unprivileged `SQPOLL` | 5.11 |
| `MADV_POPULATE_READ` | 5.14 |
| `statx(STATX_DIOALIGN)` | 6.1 |
| `kernel.io_uring_disabled` sysctl, tmpfs `O_DIRECT` | 6.6 |

---

## 8. Common mistakes checklist

- [ ] Treating a successful `write()`, `fflush()` or `fclose()` as "on disk". Use `fsync`/`fdatasync`.
- [ ] Not checking the return values of `fclose()`, `close()` and `fsync()`.
- [ ] Retrying `fsync` after a failure and trusting the second result.
- [ ] Assuming `write()` writes everything. Handle short writes and `EINTR`.
- [ ] Forgetting to `fsync` the **directory** after create or rename.
- [ ] Replacing a file in place instead of write-temp-then-`rename`.
- [ ] `O_DIRECT` with a `malloc` buffer or unaligned offset or length (`EINVAL`).
- [ ] Testing `O_DIRECT` on tmpfs.
- [ ] `mmap` on a sparse file (disk full gives `SIGBUS`) or on a file someone else may truncate.
- [ ] Using `lseek` + `read` on a shared fd from several threads. Use `pread`.
- [ ] Using `pwrite` on an `O_APPEND` fd and expecting the offset to be honored.
- [ ] Expecting `epoll` or `O_NONBLOCK` to make regular-file I/O non-blocking.
- [ ] Freeing or reusing a buffer before its async I/O has completed.
- [ ] Reading `errno` after libaio or liburing calls. They return `-errno`.
- [ ] Calling `posix_fadvise(DONTNEED)` on dirty data and expecting it to be freed.
- [ ] Leaking file descriptors, or leaking them into child processes (`O_CLOEXEC`).
- [ ] Benchmarking with a warm page cache and believing the numbers.

---

### The three laws, revisited

1. **Success from `write()` means nothing about persistence.** Data is safe only after `fsync` (and a directory `fsync` for new names) returns 0, on hardware that honors flushes.
2. **There is no small I/O.** Anything below a block still moves a whole block.
3. **The fastest I/O is the one you don't do.** Batch, cache, hint, and measure before you optimize.
