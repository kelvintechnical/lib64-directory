# Lab: What Is the `/lib64` Directory?

- **Series:** linux-ops-mastery — Linux Filesystem Hierarchy Standard
- **Subjects covered:** Filesystem Hierarchy Standard (FHS), `/lib64` as the home of 64-bit shared libraries on `x86_64`, the `/lib` vs `/lib64` arch split, ELF class (ELFCLASS32 vs ELFCLASS64), reading ELF arch with `file` and `readelf`, the dynamic linker `ld-linux-x86-64.so.2`, `ldconfig -p` filters by architecture, glibc version inspection
- **Career arcs covered:** RHCSA (`/lib64` is where every essential library on RHEL 9 actually lives), RHCE (Ansible runs Python from `/usr/bin/python3` linked to `/lib64/libpython3.9.so.1.0`), SRE ("wrong ELF class" production incident pattern), DevOps (multi-arch container builds — `linux/amd64` vs `linux/arm64`), AI/MLOps (CUDA `libcudart.so` ships under `/usr/lib64` on RHEL hosts; mismatched arch is a recurring training failure)
- **Prerequisite:** Basic `ls`, `cd`, `cat`, and a shell on RHEL 9 / Rocky 9 / Ubuntu
- **Time Estimate:** 20 to 35 minutes
- **Difficulty arc:** Task 1 inspect · 2–3 inventory + read · 4–5 demonstrate purpose · 6 capstone audit

---

## Objective

Stop treating "64-bit libraries" as an abstract attribute and treat `/lib64` as **the actual on-disk directory** that every 64-bit binary on a RHEL 9 x86_64 host loads its shared objects from. By the end of this lab you can list its contents, contrast it with `/lib` (32-bit on x86_64), classify libraries by ELF class with `file`, query the linker cache filtered by architecture, and inspect the version of glibc the system ships.

The lab is **inspection-only**. No installs, no library replacements, no `LD_PRELOAD`. You will look at the same `.so` files every binary loads thousands of times per day and understand why the directory matters even when it is "just" a symlink.

The capstone is the RHCSA-realistic prompt: *"A 32-bit binary fails on a 64-bit-only RHEL 9 host with `cannot execute binary file: Exec format error` (or `cannot open shared object file: libc.so.6`). Use only `file`, `ldd`, and `readelf` to confirm the architecture mismatch and propose the supported repair."*

---

## Concept: Why `/lib64` Exists

```
   ┌────────────────────────────────────────────────────────────────┐
   │  FHS on x86_64 root /                                          │
   ├────────────────────────────────────────────────────────────────┤
   │  /lib    → 32-bit shared libraries (ELFCLASS32)                │
   │  /lib64  → 64-bit shared libraries (ELFCLASS64)                │
   │  /usr/lib   → 32-bit (non-essential)                           │
   │  /usr/lib64 → 64-bit (non-essential) — where the bytes are     │
   │                                                                │
   │  Why two dirs? Both 32-bit and 64-bit libc.so.6 may need to    │
   │  coexist — same SONAME, different ELF class. Different dirs    │
   │  prevent collisions.                                           │
   │                                                                │
   │  Dynamic linker per arch:                                      │
   │     /lib/ld-linux.so.2          ← 32-bit programs              │
   │     /lib64/ld-linux-x86-64.so.2 ← 64-bit programs              │
   │                                                                │
   │  Modern RHEL (UsrMerge):                                       │
   │     /lib64 ── symlink ──► /usr/lib64                           │
   └────────────────────────────────────────────────────────────────┘
```

`/lib64` exists because the **same library SONAME** (`libc.so.6`) needs to be available in **multiple architectures** on the same disk. Both names ("library called libc, version 6") must resolve, but the bytes loaded by a 32-bit process and a 64-bit process are different ELF files. Putting one in `/lib` and the other in `/lib64` prevents the namespace collision.

> **Why this matters:** On RHEL 9 the **64-bit libraries are the real ones** — `/lib64` is where `libc.so.6`, `ld-linux-x86-64.so.2`, `libstdc++.so.6` and every other essential `.so` lives. `/lib` may exist but is typically minimal or empty on a 64-bit-only install. Every binary on the system records `/lib64/ld-linux-x86-64.so.2` as its interpreter. If `/lib64` were unreachable, **no binary would start**.

---

## 📜 Why `/lib64` Exists — The Story

The original UNIX filesystem standard had no concept of multiple architectures. Every machine was one architecture — PDP-11, VAX, SPARC — and every library on disk was built for that machine. The directory `/lib` was unambiguous.

That changed in the late 1990s. **64-bit hardware** (DEC Alpha, then UltraSPARC, then x86_64) arrived alongside existing **32-bit installations**, and distros suddenly had to ship both library variants. Solaris pioneered the convention with `/usr/lib` (32-bit) and `/usr/lib/64` or `/usr/lib/sparcv9`. Linux distributions briefly experimented — Debian later chose **multiarch** with directories like `/usr/lib/x86_64-linux-gnu/`, while Red Hat and SUSE took the simpler **biarch** path: `/lib` for 32-bit, `/lib64` for 64-bit.

When the **Filesystem Hierarchy Standard 3.0** was published in **2015**, it formally documented `/lib<qual>` directories (where `<qual>` is the arch suffix like `64`) as the place essential 64-bit libraries belong. FHS still treats `/lib` and `/lib64` as siblings, both essential, both required to bring the system to single-user state.

The 2012-2014 **UsrMerge** transition folded both `/lib` and `/lib64` into `/usr` as symlinks. The directory names remained because thousands of binaries hard-code `/lib64/ld-linux-x86-64.so.2` as their interpreter path inside the ELF header. Renaming the directory would un-runable every binary on the system instantly. As a result, even on a brand-new RHEL 9 install — pure 64-bit, UsrMerged, modern systemd — the path `/lib64/ld-linux-x86-64.so.2` is still the very first thing the kernel touches when you exec any program.

> **The point of the story:** `/lib64` is the **arch-specific anchor** of the dynamic linker. Every 64-bit binary on Linux/x86_64 has `/lib64/ld-linux-x86-64.so.2` written into its program header. The directory is hard-coded into the world.

---

## 👪 The `/lib64` Family — Who Lives There

### Essential 64-bit libraries (always under `/lib64`)

| Library | Purpose | Architecture |
|---|---|---|
| `ld-linux-x86-64.so.2` | Dynamic linker / loader for 64-bit ELF | ELFCLASS64 / x86_64 |
| `libc.so.6` | GNU C library (glibc) — every libc function | ELFCLASS64 |
| `libm.so.6` | Math library (`sin`, `cos`, `sqrt`, etc.) | ELFCLASS64 |
| `libdl.so.2` | `dlopen`/`dlsym` for plugin loading | ELFCLASS64 |
| `libpthread.so.0` | POSIX threads (merged into glibc) | ELFCLASS64 |
| `libstdc++.so.6` | C++ standard library | ELFCLASS64 |
| `libgcc_s.so.1` | GCC runtime support (exception unwinding) | ELFCLASS64 |
| `libselinux.so.1` | SELinux userland | ELFCLASS64 |
| `libssl.so.3`, `libcrypto.so.3` | OpenSSL 3 | ELFCLASS64 |
| `libpcre2-8.so.0` | Modern regex (PCRE2 8-bit) | ELFCLASS64 |

### Related directories you will visit

| Directory | Purpose | Relation to `/lib64` |
|---|---|---|
| `/lib` | 32-bit shared libraries on x86_64 | Sibling — different ELF class |
| `/usr/lib` | 32-bit (non-essential) | Symlink target of `/lib` |
| `/usr/lib64` | 64-bit (non-essential) | Symlink target of `/lib64` |
| `/usr/local/lib64` | Locally-installed 64-bit libs | `make install --prefix=/usr/local` |
| `/etc/ld.so.conf.d/` | Per-package linker conf | Adds extra `lib64` dirs to the cache |
| `/etc/ld.so.cache` | Compiled linker cache | Indexed by SONAME + arch |

### Tools that interact with `/lib64`

| Tool | What it tells you about `/lib64` |
|---|---|
| `file /lib64/libc.so.6` | Confirms ELF 64-bit |
| `readelf -h /lib64/libc.so.6` | Header: class, machine, version |
| `ldconfig -p \| grep x86-64` | Filter linker cache by 64-bit |
| `objdump -p /bin/ls \| grep interpreter` | Confirms binary uses `/lib64/ld-linux-x86-64.so.2` |
| `rpm -qf /lib64/libc.so.6` | Owning package (`glibc-...x86_64`) |
| `ldd --version` | glibc version on the system |
| `getconf GNU_LIBC_VERSION` | Same, via libc itself |

> **The point of the family tree:** Architecture detection is a `file` + `readelf` job. The linker cache filters by arch implicitly. Mixing 32-bit and 64-bit libraries in the same path would be the worst possible bug; `/lib` vs `/lib64` prevents it.

---

## 🔬 The Anatomy of `file /lib64/libc.so.6` — In One Diagram

```
$ file /lib64/libc.so.6
/lib64/libc.so.6: symbolic link to libc-2.34.so

$ file /lib64/libc-2.34.so
/lib64/libc-2.34.so: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, ...
 │                    │   │       │   │             │
 │                    │   │       │   │             └─ Linking model: dynamic
 │                    │   │       │   └─ ELF machine type: AMD x86-64 (e_machine = EM_X86_64)
 │                    │   │       └─ ELF data encoding: LSB = little-endian (LSB-first byte order)
 │                    │   └─ ELF version 1 (current ELF version, all real systems use this)
 │                    └─ ELF class: 64-bit (ELFCLASS64) — vs 32-bit on /lib
 └─ File magic: ELF (0x7F E L F)

$ readelf -h /lib64/libc-2.34.so | head
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
                       ^^^^^
                       0x02 = ELFCLASS64
  Class:                             ELF64                  ← This is the canonical bit
  Data:                              2's complement, little endian
  OS/ABI:                            UNIX - System V
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
```

> **Reading rule:** A 32-bit library reports `ELF 32-bit LSB shared object, Intel 80386` from `file`. A 64-bit library on the same system reports `ELF 64-bit LSB shared object, x86-64`. Trying to link a 64-bit binary against a 32-bit library produces `wrong ELF class: ELFCLASS32` from the dynamic linker.

---

## 📚 `/lib64` Reference Table

| Task | Command | Notes |
|---|---|---|
| Confirm `/lib64` is a symlink | `ls -ld /lib64` | First char `l` on UsrMerge |
| Print symlink target | `readlink /lib64` | Returns `usr/lib64` |
| Count `.so` files | `find /usr/lib64 -name '*.so*' \| wc -l` | Thousands on RHEL 9 |
| Identify ELF class | `file /lib64/libc.so.6` | ELF 64-bit, x86-64 |
| Show ELF header | `readelf -h /lib64/libc.so.6` | Includes class, machine, ABI |
| glibc version | `ldd --version \| head -1` | `glibc 2.34` on RHEL 9 |
| `libc` build version | `/lib64/libc.so.6` (run directly) | Prints version + build details |
| Filter cache by arch | `ldconfig -p \| grep 'x86-64'` | Only 64-bit entries |
| Binary's interpreter | `readelf -l /bin/ls \| grep interpreter` | Always `/lib64/ld-linux-x86-64.so.2` |
| Owning RPM | `rpm -qf /lib64/libc.so.6` | `glibc-2.34-...x86_64` |

> **Rule one of `/lib64`:** Treat it as the **most critical directory on the system**. Every other directory can have transient issues. If `/lib64` is unreachable, the kernel cannot start the next process.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Every binary the grader runs depends on `/lib64`. Knowing it is a symlink to `/usr/lib64` resolves half the "where is this library?" questions instantly. |
| **RHCE candidate** | Ansible's Python interpreter on RHEL 9 links to `/lib64/libpython3.9.so.1.0`. Container target hosts must keep this binding. |
| **SRE / Platform** | "wrong ELF class: ELFCLASS32" is the classic "wrong image arch" production failure. The fix is one `file` command away. |
| **DevOps** | Multi-arch buildx — `linux/amd64` vs `linux/arm64` — produces different `/lib64` contents. Image manifests select the right one. |
| **AI / MLOps** | CUDA `libcudart.so` and `libcublas.so` ship under `/usr/lib64`. Mismatched driver/library arch breaks training silently. |

---

## 🔧 The 6 Tasks

> Six inspection-only phases that build the **identify → contrast → classify → version → audit** habit for 64-bit libraries.

---

### Task 1 — Inspect `/lib64` itself

**Purpose:** Confirm whether `/lib64` is a directory or a symlink on your RHEL 9 system, and capture the metadata you will reference throughout the lab.

```bash
ls -ld /lib64
stat /lib64
readlink /lib64
file /lib64
readlink -f /lib64
ls -ld /lib  
```

**Human-Readable Breakdown:** Same five inspections as the `/bin` and `/lib` labs (long listing, inode metadata, raw target, classification, canonical path), plus a side glance at `/lib` to compare. Both should be symlinks on a modern RHEL 9 install.

**Reading it left to right:** `ls -ld /lib64` returns one line with the `l` prefix. `stat` shows the symlink properties. `readlink` returns the relative target. `readlink -f` resolves to absolute. Comparing `/lib` to `/lib64` reveals whether both follow the UsrMerge convention.

**The story:** On an x86_64 RHEL 9 server, both `/lib` and `/lib64` are symlinks, but `/usr/lib64` is **full** of real `.so` files while `/usr/lib` is sparse or near-empty unless 32-bit compatibility packages are installed. The size of those two target directories tells you whether the host supports legacy 32-bit binaries.

**Expected output:**

```text
lrwxrwxrwx. 1 root root 9 Apr 12  2024 /lib64 -> usr/lib64
  File: /lib64 -> usr/lib64
  Size: 9         	Blocks: 0          IO Block: 4096   symbolic link
Device: fd00h/64768d	Inode: 1572868     Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
usr/lib64
/lib64: symbolic link to usr/lib64
/usr/lib64
lrwxrwxrwx. 1 root root 7 Apr 12  2024 /lib -> usr/lib
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -ld` | Long format, do not descend |
| `stat` | Inode metadata |
| `readlink -f` | Canonicalized path |
| `file` | Classify by content |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/lib64` is a directory | Non-UsrMerge distro — fine, lab continues |
| `/lib64` missing entirely | Pure 32-bit container — substitute `/lib` everywhere |
| Both `/lib` and `/lib64` empty | Container stripped — install `glibc` package |

---

### Task 2 — Contrast `/lib` vs `/lib64` contents

**Purpose:** Demonstrate that `/lib64` holds the **64-bit `libc.so.6`** while `/lib` (if populated) holds the **32-bit `libc.so.6`** — same SONAME, different ELF class, hence different directories.

```bash
ls -la /usr/lib | head -n 10
ls -la /usr/lib64 | head -n 10
ls -la /usr/lib64 | wc -l
ls -la /usr/lib | wc -l
file /usr/lib/libc.so.6 2>/dev/null || echo "/usr/lib/libc.so.6 not present (pure 64-bit host)"
file /usr/lib64/libc.so.6
```

**Human-Readable Breakdown:** Show the top of each directory, count their entries, and `file` the libc in each — proving that if both exist, they are different ELF classes.

**Reading it left to right:** `ls -la` reveals hidden entries and total counts. `wc -l` gives quick cardinality. The `||` short-circuit on the `file /usr/lib/libc.so.6` command handles the very common case where 32-bit libc is not installed.

**The story:** RHEL 9 ships **64-bit-only** by default. To run a 32-bit application you must install `glibc.i686` (note the architecture suffix), which populates `/usr/lib/libc.so.6`. That coexistence is the entire reason the two directories exist as siblings.

**Expected output:**

```text
total 320
drwxr-xr-x.   8 root root   4096 Mar 22  2024 .
drwxr-xr-x.  13 root root   4096 Mar 22  2024 ..
drwxr-xr-x.   3 root root   4096 Mar 22  2024 NetworkManager
drwxr-xr-x.   2 root root  20480 Mar 22  2024 binfmt.d
drwxr-xr-x.   2 root root   4096 Mar 22  2024 cracklib
drwxr-xr-x.   2 root root   4096 Mar 22  2024 debug
drwxr-xr-x.   2 root root   4096 Mar 22  2024 dracut
total 168388
drwxr-xr-x.  74 root root  20480 Mar 22  2024 .
drwxr-xr-x.  13 root root   4096 Mar 22  2024 ..
lrwxrwxrwx.   1 root root     16 Mar 22  2024 ld-linux-x86-64.so.2 -> ld-2.34.so
-rwxr-xr-x.   1 root root 220584 Mar 22  2024 ld-2.34.so
-rwxr-xr-x.   1 root root      0 Mar 22  2024 ld-linux.so.2
4127
312
/usr/lib/libc.so.6 not present (pure 64-bit host)
/usr/lib64/libc.so.6: symbolic link to libc-2.34.so
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -la` | All files including hidden, long format |
| `wc -l` | Count lines |
| `\|\| echo` | Fallback when previous command fails |
| `file PATH` | Classify file |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/usr/lib/libc.so.6` exists and is 32-bit | Host has multilib — that's expected on some workstations |
| Both libc files show same class | Misconfigured host or container — investigate |
| Directory count is reversed | Confirm with `find` — RHEL 9 always has more under `/usr/lib64` |

---

### Task 3 — Inspect glibc version on the system

**Purpose:** Three different ways to ask "what version of glibc is this?" — because every senior engineer's first instinct on a "GLIBC_2.34 not found" error is to confirm the host's libc version.

```bash
ldd --version | head -n 3
echo "---"
/lib64/libc.so.6 | head -n 5
echo "---"
getconf GNU_LIBC_VERSION
echo "---"
rpm -q glibc
echo "---"
readelf -V /lib64/libc.so.6 2>/dev/null | head -n 20
```

**Human-Readable Breakdown:** Four different version sources: `ldd --version` (says glibc version), executing `/lib64/libc.so.6` directly (libc is itself an executable that prints its version banner), `getconf GNU_LIBC_VERSION` (libc API call), `rpm -q glibc` (package name), and `readelf -V` to dump the version definitions inside the library.

**Reading it left to right:** Each tool sources the version from a different layer — the linker, the runtime, the libc API, the package database, and the ELF version section. When they all agree (RHEL 9 = glibc 2.34), the system is consistent. When they disagree, something has been hand-patched.

**The story:** RHEL 9 ships glibc 2.34. RHEL 8 ships 2.28. RHEL 7 ships 2.17. Knowing which release maps to which glibc lets you predict whether a binary built on a newer RHEL will run on an older one (it won't — version symbols).

**Expected output:**

```text
ldd (GNU libc) 2.34
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
---
GNU C Library (GNU libc) stable release version 2.34.
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
---
2.34
---
glibc-2.34-100.el9_4.2.x86_64
---
Version definition section '.gnu.version_d' contains 33 entries:
  Addr: 0x0000000000189a40  Offset: 0x189a40  Link: 4 (.dynstr)
  000000: Rev: 1  Flags: BASE  Index: 1  Cnt: 1  Name: libc.so.6
  0x001c: Rev: 1  Flags: none  Index: 2  Cnt: 1  Name: GLIBC_2.2.5
  0x0038: Rev: 1  Flags: none  Index: 3  Cnt: 1  Name: GLIBC_2.3
  0x0054: Rev: 1  Flags: none  Index: 4  Cnt: 1  Name: GLIBC_2.3.2
  0x0070: Rev: 1  Flags: none  Index: 5  Cnt: 1  Name: GLIBC_2.4
  0x008c: Rev: 1  Flags: none  Index: 6  Cnt: 1  Name: GLIBC_2.6
  0x00a8: Rev: 1  Flags: none  Index: 7  Cnt: 1  Name: GLIBC_2.7
```

**Switches**

| Token | Meaning |
|---|---|
| `ldd --version` | Linker version (= glibc) |
| `/lib64/libc.so.6` | Run libc directly to print banner |
| `getconf GNU_LIBC_VERSION` | libc-provided constant |
| `readelf -V` | Dump version definitions |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `getconf` not found | Install `glibc-common` |
| `/lib64/libc.so.6` fails to execute | Damaged libc — boot rescue media |
| Different versions reported | Multi-libc system (rare) — `find / -name 'libc-*.so'` |

---

### Task 4 — Filter the linker cache by architecture

**Purpose:** Demonstrate that `ldconfig -p` annotates every entry with an architecture string (`libc6,x86-64` or `libc6` for 32-bit), and that you can filter for one or the other.

```bash
ldconfig -p | wc -l
echo "---"
ldconfig -p | grep 'x86-64' | wc -l
echo "---"
ldconfig -p | grep -v 'x86-64' | head -n 10
echo "---"
ldconfig -p | awk '/libc\.so\.6/ {print}' | head
echo "---"
ldconfig -p | awk -F'=>' '/x86-64/ {print $2}' | xargs dirname | sort -u | head
```

**Human-Readable Breakdown:** Total cache size, count of 64-bit entries, the first 10 non-64-bit entries (typically header lines or 32-bit if multilib), the libc family across all archs, and finally the unique directories under which 64-bit libraries live (should be dominated by `/lib64` and `/usr/lib64`).

**Reading it left to right:** `ldconfig -p` outputs `SONAME (arch-flags) => /path`. `awk -F'=>'` splits on `=>` so `$2` is the path. `dirname` extracts directory. `sort -u` deduplicates.

**The story:** The linker cache is the **truth source** for what the dynamic linker considers loadable. If `ldconfig -p | grep libfoo` returns nothing, the linker cannot find `libfoo` regardless of whether it is on disk. Filtering by `x86-64` confirms architecture compatibility for the host.

**Expected output:**

```text
4128
---
4115
---
4128 libs found in cache `/etc/ld.so.cache'
---
	libc.so.6 (libc6,x86-64, OS ABI: Linux 3.2.0) => /lib64/libc.so.6
---
/lib64
/usr/lib64
/usr/lib64/dyninst
/usr/lib64/iscsi
/usr/lib64/mariadb
/usr/lib64/mysql
```

**Switches**

| Token | Meaning |
|---|---|
| `ldconfig -p` | Print cache contents |
| `grep -v` | Invert (lines NOT matching) |
| `awk -F'=>'` | Field separator is `=>` |
| `xargs dirname` | Extract directory from each path |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `grep -v 'x86-64'` only returns the header line | Pure 64-bit host (expected on default RHEL 9) |
| `grep 'x86-64'` returns 0 | Host is non-x86_64 — substitute `aarch64`, `ppc64le`, etc. |
| Awk pipe returns nothing | Re-pipe through `cat -A` to inspect whitespace |

---

### Task 5 — Confirm a binary uses `/lib64/ld-linux-x86-64.so.2`

**Purpose:** Read the **program header** of a real binary to see the `INTERP` field — the hard-coded path to the dynamic linker — and prove that it always points at `/lib64`.

```bash
readelf -l /bin/ls 2>/dev/null | grep -A1 -E 'INTERP|interpreter'
echo "---"
file /bin/ls
echo "---"
file /lib64/ld-linux-x86-64.so.2
echo "---"
strings /bin/ls | grep -E '^/lib(64)?/ld-' | head -n 3
echo "---"
for b in /bin/ls /bin/cat /bin/bash /usr/sbin/sshd; do
  echo -n "$b: "
  readelf -l "$b" 2>/dev/null | awk '/program interpreter/ {print $NF}' | tr -d '[]'
done
```

**Human-Readable Breakdown:** Five views: `readelf -l` shows the program-header INTERP field, `file /bin/ls` confirms it as ELF 64-bit, `file /lib64/ld-linux-x86-64.so.2` confirms the interpreter is also a 64-bit ELF, `strings` confirms the path is literally embedded in the binary, and a loop reports the interpreter for four common binaries.

**Reading it left to right:** Every ELF binary has a `PT_INTERP` program header that names the dynamic linker the kernel must invoke. The string is baked into the ELF at link time. On RHEL 9 x86_64, every dynamic binary has `/lib64/ld-linux-x86-64.so.2`. If you renamed `/lib64`, the kernel could not find the interpreter, and **no program would start** — including any program you would use to rename it back.

**The story:** This is the **physical anchor** that makes `/lib64` un-renamable. Even if you `mv /lib64 /lib64.bak`, every subsequent `exec()` syscall fails with `ENOENT` because the kernel is hard-coded to look up the interpreter at the path stored in the ELF. That property is unique to `/lib64` and `/lib`; no other directory has this kind of structural lock.

**Expected output:**

```text
  INTERP         0x0000000000000318 0x0000000000400318 0x0000000000400318 0x000000000000001c 0x000000000000001c R 0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
---
/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, ...
---
/lib64/ld-linux-x86-64.so.2: symbolic link to ld-2.34.so
---
/lib64/ld-linux-x86-64.so.2
---
/bin/ls: /lib64/ld-linux-x86-64.so.2
/bin/cat: /lib64/ld-linux-x86-64.so.2
/bin/bash: /lib64/ld-linux-x86-64.so.2
/usr/sbin/sshd: /lib64/ld-linux-x86-64.so.2
```

**Switches**

| Token | Meaning |
|---|---|
| `readelf -l` | Show program headers (segments) |
| `INTERP` | Program-header type naming the dynamic linker |
| `strings` | Extract printable strings from a binary |
| `tr -d '[]'` | Strip square brackets from awk output |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Different interpreter for one binary | That binary is statically linked or built for a different arch |
| `readelf -l` empty | Binary is not ELF (script?) — try `file` first |
| Interpreter is `/lib/ld-linux.so.2` | That binary is 32-bit — confirm with `file` |

---

### Task 6 — Capstone: diagnose 32-bit binary on 64-bit host

**Task statement:** *"A binary fails on a 64-bit-only RHEL 9 host with `cannot execute binary file: Exec format error` (or runs, but errors with `cannot open shared object file: libc.so.6`). Use only `file`, `ldd`, and `readelf` to confirm the architecture mismatch and propose the supported repair."*

**Purpose:** Combine `file`, `ldd`, and `readelf` into the canonical 32-vs-64 diagnosis loop. Use an existing 64-bit binary as the "host arch" reference; demonstrate what a 32-bit binary would show if one were present.

```bash
TARGET=/bin/ls
echo "=== Architecture diagnosis for $TARGET ==="
echo
echo "--- 1. Kernel arch ---"
uname -m
echo
echo "--- 2. Binary classification (file) ---"
file "$TARGET"
echo
echo "--- 3. ELF header (readelf -h) ---"
readelf -h "$TARGET" 2>/dev/null | grep -E 'Class:|Data:|Machine:|Type:'
echo
echo "--- 4. Required interpreter (readelf -l) ---"
readelf -l "$TARGET" 2>/dev/null | awk '/program interpreter/ {print $NF}' | tr -d '[]'
echo
echo "--- 5. Library resolution (ldd) ---"
ldd "$TARGET" 2>&1 | head -n 5
echo
echo "--- 6. Available archs of glibc on disk ---"
find /usr/lib /usr/lib64 -maxdepth 1 -name 'libc.so.6' -exec file {} \; 2>/dev/null
echo
echo "--- 7. Proposed repair (NOT executed) ---"
echo "If file/readelf shows ELFCLASS32 (Intel 80386) and uname -m is x86_64:"
echo "  Either:"
echo "    a) Install 32-bit runtime:    sudo dnf install glibc.i686 libstdc++.i686"
echo "    b) Build/obtain a 64-bit copy of the binary (preferred)"
echo "  After (a), confirm with ldd that 'libc.so.6 => /lib/libc.so.6' resolves."
```

**Human-Readable Breakdown:** Seven-section audit. Section 1 records kernel architecture. Section 2 classifies the target binary with `file`. Section 3 reads ELF header fields. Section 4 reads the interpreter path. Section 5 shows library resolution. Section 6 enumerates which arch of glibc is present on disk. Section 7 documents the two supported repair paths.

**Reading it left to right:** The diagnostic walks the same chain the kernel walks at exec time — arch, ELF class, interpreter path, library resolution. Any mismatch (32-bit binary on 64-bit-only host, or 64-bit binary on 32-bit-only host) shows up in section 2 or section 5.

**Layer stack you traversed:**

```text
Kernel (uname -m)              ← target architecture
   │
   ▼
Binary ELF header (file)        ← class, machine
   │
   ▼
PT_INTERP (readelf -l)         ← which dynamic linker
   │
   ▼
Library resolution (ldd)        ← does each .so resolve?
   │
   ▼
On-disk arch (find + file)      ← what archs are available?
```

**The story:** The **wrong ELF class** error is unmistakable once you've seen it. RHEL 9 servers default to 64-bit-only — no `glibc.i686` package installed. Running an old 32-bit vendor binary fails immediately. The supported path is `dnf install glibc.i686 libstdc++.i686 zlib.i686` (and the rest of the binary's dependencies). The preferred path is rebuilding the binary for 64-bit.

**Expected output:**

```text
=== Architecture diagnosis for /bin/ls ===

--- 1. Kernel arch ---
x86_64

--- 2. Binary classification (file) ---
/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, ..., for GNU/Linux 3.2.0, stripped

--- 3. ELF header (readelf -h) ---
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64

--- 4. Required interpreter (readelf -l) ---
/lib64/ld-linux-x86-64.so.2

--- 5. Library resolution (ldd) ---
	linux-vdso.so.1 (0x00007ffc...)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007fbc...)
	libcap.so.2 => /lib64/libcap.so.2 (0x00007fbc...)
	libc.so.6 => /lib64/libc.so.6 (0x00007fbc...)
	libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007fbc...)

--- 6. Available archs of glibc on disk ---
/usr/lib64/libc.so.6: symbolic link to libc-2.34.so

--- 7. Proposed repair (NOT executed) ---
If file/readelf shows ELFCLASS32 (Intel 80386) and uname -m is x86_64:
  Either:
    a) Install 32-bit runtime:    sudo dnf install glibc.i686 libstdc++.i686
    b) Build/obtain a 64-bit copy of the binary (preferred)
  After (a), confirm with ldd that 'libc.so.6 => /lib/libc.so.6' resolves.
```

**Cleanup**

```bash
# nothing destructive — this lab was inspection-only
# no files written, no packages installed, no libraries replaced
echo "diagnosis complete — /lib64 untouched"
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Exec format error` on a binary you built | Cross-compilation arch mismatch — rebuild for host arch |
| `wrong ELF class: ELFCLASS32` from `ldd` | 32-bit binary needs 32-bit libs — install `glibc.i686` |
| Binary runs but loads wrong library | `LD_LIBRARY_PATH` pollution — `unset LD_LIBRARY_PATH` and retry |
| `uname -m` reports `aarch64` | ARM host — substitute `aarch64` for `x86_64` throughout |

---

## 🔍 `/lib64` Decision Guide

```
Binary fails at runtime — is it a library issue?
  │
  ├── "Exec format error"
  │       └── ✅ ELF class mismatch (32-bit binary on 64-bit-only host)
  │              ↳ Install glibc.i686 OR rebuild for 64-bit
  │
  ├── "wrong ELF class: ELFCLASSxx"
  │       └── ✅ Right family but wrong arch
  │              ↳ Check uname -m vs file <binary>
  │
  ├── "libfoo.so.5: cannot open shared object file"
  │       └── ✅ See /lib lab — missing library
  │
  ├── "GLIBC_2.34 not found"
  │       └── ✅ Binary built against newer glibc than host has
  │              ↳ Use newer base image or container
  │
  └── "Segfault on libc call"
          └── ✅ Library corruption — rpm -V glibc
                 ↳ dnf reinstall glibc
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Inspect `/lib64` and `/lib` symlinks with `ls -ld`, `stat`, `readlink`, `file`
- [ ] 02 Contrast contents of `/usr/lib` and `/usr/lib64`, confirm libc arch
- [ ] 03 Query glibc version four ways (ldd, libc.so.6, getconf, rpm)
- [ ] 04 Filter the linker cache by `x86-64` architecture
- [ ] 05 Read program-header `INTERP` to confirm `/lib64/ld-linux-x86-64.so.2`
- [ ] 06 Run the seven-section 32-vs-64 architecture diagnosis capstone

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Confusing `/lib` and `/lib64` | "wrong ELF class" errors | Match arch to binary with `file` |
| Trying to `mv` or `rm` `/lib64` | Every command fails instantly | Never modify `/lib64` on a running system |
| Installing 32-bit libs into `/lib64` | Linker confusion | Use `dnf install pkg.i686` — RPM places correctly |
| Running 32-bit binary on 64-bit-only host | "Exec format error" | Install multilib runtime or rebuild |
| Setting `LD_LIBRARY_PATH=/lib` for 64-bit | Wrong libs loaded | Use `/lib64` paths or skip `LD_LIBRARY_PATH` |
| Trusting `file` on a stripped binary's arch | "stripped" omitted but arch is fine | Arch is in the ELF header; `readelf -h` confirms |
| Assuming RHEL 9 has 32-bit libs | `dnf install` of i686 packages fails by default | Enable multilib explicitly |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Know that `/lib64` holds the real libraries on x86_64 and `/lib64/ld-linux-x86-64.so.2` is hard-coded into every binary's ELF interpreter field.

**RHCE candidate**
- In Ansible roles deploying vendor software, always check ELF class of provided binaries. Use `community.general.alternatives` to manage multiple library versions.

**SRE / Platform interview**
- "Wrong ELF class: ELFCLASS32" → walk the seven-section diagnosis. Identify root cause (32-bit binary on 64-bit-only host), propose `dnf install glibc.i686` or rebuilding.

**DevOps**
- Multi-arch buildx (`docker buildx build --platform linux/amd64,linux/arm64`) generates per-arch `/lib64` (or `/lib/aarch64-linux-gnu/`). Manifest selection happens at pull.

**AI / MLOps**
- `nvidia-smi`, `libcudart.so`, `libcublas.so` all ship under `/usr/lib64`. Mismatched CUDA driver vs library produces `CUDA driver version is insufficient` — same diagnostic flow.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [/bin](https://github.com/kelvintechnical/bin-directory) | Binaries that load from `/lib64` |
| [/sbin](https://github.com/kelvintechnical/sbin-directory) | Admin binaries — same arch story |
| [/lib](https://github.com/kelvintechnical/lib-directory) | 32-bit sibling — direct contrast |
| [/usr](https://github.com/kelvintechnical/usr-directory) | Canonical home (`/usr/lib64`) |
| [/etc](https://github.com/kelvintechnical/etc-directory) | `ld.so.conf.d` lives here |
| [/boot](https://github.com/kelvintechnical/boot-directory) | Kernel arch must match `/lib64` arch |
| [/home](https://github.com/kelvintechnical/home-directory) | User binaries respect same `/lib64` |
| [/root](https://github.com/kelvintechnical/root-directory) | First place to land in rescue |
| [/var](https://github.com/kelvintechnical/var-directory) | `ld.so` errors logged via journald |
| [/tmp](https://github.com/kelvintechnical/tmp-directory) | Scratch space |
| [/opt](https://github.com/kelvintechnical/opt-directory) | Vendor software with own `lib64` |
| [/srv](https://github.com/kelvintechnical/srv-directory) | Service data |
| [/dev](https://github.com/kelvintechnical/dev-directory) | Device nodes |
| [/proc](https://github.com/kelvintechnical/proc-directory) | `/proc/<pid>/maps` shows loaded `/lib64` addresses |
| [/sys](https://github.com/kelvintechnical/sys-directory) | Kernel object exposure |
| [/run](https://github.com/kelvintechnical/run-directory) | Runtime state |
| [/media](https://github.com/kelvintechnical/media-directory) | Removable media |
| [/mnt](https://github.com/kelvintechnical/mnt-directory) | Manual mount points |
| [/afs](https://github.com/kelvintechnical/afs-directory) | AFS mount point |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
