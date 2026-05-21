# What is the /lib64 Directory?

**Linux Filesystem Hierarchy Standard (FHS)** | Lab 04  
**Part of:** [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)  
**Previous Lab:** [What is the /lib directory?](https://github.com/kelvintechnical/lib-directory)  
**Next Lab:** [What is the /usr directory?](https://github.com/kelvintechnical/what-is-the-usr-directory)  
**Time Estimate:** 45–60 minutes (5 labs)

---

## 🎯 Objective

Understand what the `/lib64` directory is, why a 64-bit system needs it as a separate location from `/lib`, what the **dynamic linker** does, and how to troubleshoot architecture-mismatch errors — through five hands-on labs that take you from "I never noticed it" to "I can diagnose a 32-vs-64-bit failure on sight."

---

## 🧠 Big Idea — What is /lib64?

`/lib64` stands for **64-bit libraries**. It contains the shared `.so` files compiled for the **64-bit** CPU architecture (x86-64, aarch64, ppc64le, s390x).

> Think of `/lib64` as the **modern engine room**. While `/lib` is the historic library directory (often holding 32-bit code for backward compatibility), `/lib64` is where the **real** machinery for your 64-bit operating system lives — including the most important program on the entire system: the **dynamic linker** `ld-linux-x86-64.so.2`.

**Key rule:** If your CPU is 64-bit (almost all modern systems), the libraries your binaries actually load come from `/lib64`.

**One-sentence definition:**
> `/lib64` is the canonical home of **64-bit shared libraries** on a multilib-capable Linux system, including the dynamic linker that bootstraps every dynamically-linked 64-bit binary on the machine.

---

## 🏗️ Why /lib64 Exists — The Story

When Linux moved from 32-bit (x86) to 64-bit (x86-64) in the early 2000s, the kernel could run **both** kinds of binaries side by side — but a 64-bit program cannot load a 32-bit library, and vice versa. They are completely different machine code.

The Linux Standard Base solved this with a rule: keep 32-bit `.so` files in `/lib`, keep 64-bit `.so` files in `/lib64`. Same library name (`libc.so.6`), different folder, different machine code. A 64-bit binary asks the kernel for `libc.so.6` and the dynamic linker (which knows the binary's architecture from its ELF header) automatically searches `/lib64` first. A 32-bit binary running on the same machine searches `/lib` first. **Two parallel universes, one filesystem.**

On modern RHEL/Fedora (the UsrMerge era), `/lib64` is a symlink to `/usr/lib64`, where the real files live. The split between `/lib` (32-bit) and `/lib64` (64-bit) is preserved, just one level deeper.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| List contents of /lib64 | `ls /lib64` |
| Count how many libraries are in /lib64 | `ls /lib64 \| wc -l` |
| Confirm /lib64 is a symlink | `ls -la / \| grep lib64` |
| Verify a binary is 64-bit ELF | `file /lib64/libc.so.6` |
| Find the dynamic linker on this system | `ls /lib64/ld-linux*` |
| Run the dynamic linker directly | `/lib64/ld-linux-x86-64.so.2 --help` |
| Show every 64-bit lib the linker can find | `ldconfig -p \| grep x86-64` |
| Compare 32-bit vs 64-bit lib counts | `ls /lib \| wc -l && ls /lib64 \| wc -l` |
| See a process's loaded 64-bit libs | `cat /proc/<pid>/maps \| grep lib64` |

---

## 🗺️ The /lib64 Family — Architecture Layout

```
/                              ← root
├── lib              -> usr/lib       ← 32-bit + arch-independent (symlink)
├── lib64            -> usr/lib64     ← 64-bit shared libs (symlink)
└── usr/
    ├── lib/                          ← arch-independent files, 32-bit libs
    │   └── modules/<kernel>/         ← kernel modules (architecture-tagged)
    ├── lib64/                        ← the real 64-bit engine room
    │   ├── ld-linux-x86-64.so.2      ← the dynamic linker
    │   ├── libc.so.6 -> libc-2.34.so ← versioned symlinks
    │   ├── libssl.so.3               ← OpenSSL
    │   ├── libcrypto.so.3            ← OpenSSL crypto
    │   └── ... (thousands more)
    └── libexec/                      ← helper binaries, not in $PATH
```

| Path | What's there |
|---|---|
| `/lib64` | 64-bit shared libs (symlink to `/usr/lib64`) |
| `/usr/lib64/ld-linux-x86-64.so.2` | **The dynamic linker** — the only program the kernel runs to start every other 64-bit program |
| `/usr/lib64/libc.so.6` | GNU C Library — every C program depends on this |
| `/usr/lib64/libssl.so.3`, `libcrypto.so.3` | OpenSSL — TLS for sshd, httpd, curl, dnf, ... |
| `/usr/lib64/libpam.so.0` | PAM — every login authenticates through here |
| `/usr/lib64/security/` | PAM modules (pam_unix.so, pam_sss.so, ...) |
| `/usr/lib64/python3.X/` | Architecture-tagged Python C extensions |

---

# 🔧 Lab 1 — Navigate to /lib64

**Goal:** Land in `/lib64`, prove it's a symlink, and confirm you understand the 64-bit context you're in.

---

### Task 1.1 — `cd /lib64` and resolve the symlink

**Purpose:** First move. Like `/lib`, this is a symlink on modern systems — verify it.

```bash
cd /lib64
pwd
pwd -P
```

**Expected output (RHEL 9 / Fedora):**

```
/lib64
/usr/lib64
```

**Switches**

| Token | Meaning |
|---|---|
| `cd /lib64` | Absolute path — the leading `/` means "start from root" |
| `pwd` | Default = `-L` = logical path (the names you walked) |
| `pwd -P` | **P**hysical path — resolves every symlink to its real target |

**Output decoded**

| Line | What it tells you |
|---|---|
| `/lib64` | The name your shell walked — `/lib64` appears as a directory |
| `/usr/lib64` | The real on-disk location after the symlink is resolved |

> **The Story:** When the UsrMerge happened, distributions wanted to consolidate `/bin`, `/sbin`, `/lib`, and `/lib64` under `/usr` — but breaking thousands of scripts and binaries that hard-coded paths like `/lib64/ld-linux-x86-64.so.2` was unacceptable. The compromise: **leave the names alone, make them symlinks**. So `cd /lib64` still works, every binary that says `interp /lib64/ld-linux-x86-64.so.2` still works, but the actual file lives at `/usr/lib64/ld-linux-x86-64.so.2`.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| `pwd -P` | You never learn that `/lib64` is a symlink — and later you'll be confused why `/usr/lib64` looks identical |
| The leading `/` | `cd lib64` looks for `lib64` in your current directory — fails unless you happened to be at `/` |
| The `/lib64` symlink itself | **Catastrophic.** Every dynamically-linked binary on the system fails to start — the kernel can't find `ld-linux-x86-64.so.2` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: cd: /lib64: No such file or directory` | You're on a 32-bit-only system or an exotic arch — `ls /` to see what library directory exists |
| `pwd` and `pwd -P` show the same thing | Older or non-UsrMerge distro — `/lib64` is a real directory there |
| `pwd -P` returns `/usr/lib` (no `64`) | Pure 64-bit system that merged everything — rare but legal (some Arch configurations) |

---

### Task 1.2 — Prove /lib64 is a symlink the same way as /lib

**Purpose:** Read the symlink directly with `ls -la /` so you see it character-by-character.

```bash
ls -la / | grep lib64
```

**Expected output:**

```
lrwxrwxrwx.   1 root root     9 May 11 14:02 lib64 -> usr/lib64
```

**Command explained — the bouncer breakdown**

| Part | Meaning |
|---|---|
| `ls -la /` | List root with long format, including dotfiles |
| `\|` | Pipe — send `ls`'s output to grep |
| `grep lib64` | Keep only lines containing `lib64` |

**Output decoded — every column**

| Column | Value | Meaning |
|---|---|---|
| 1 (char 1) | `l` | **L**ink — symbolic link (not `-` regular, not `d` directory) |
| 1 (chars 2–10) | `rwxrwxrwx` | Symlink permissions are always wide open; the **target's** perms are what enforce access |
| 1 (char 11) | `.` | SELinux context attached |
| 2 | `1` | Hard link count |
| 3–4 | `root root` | Owner and group |
| 5 | `9` | Length of the target string — `usr/lib64` is exactly 9 characters |
| 6 | `May 11 14:02` | Last modification of the link itself (not the target) |
| 7 | `lib64 -> usr/lib64` | Name → target |

> **Why the size column shows `9` instead of a real file size:** Symlinks are tiny — the entire "content" is just the target path as text. `usr/lib64` is 9 characters, so the link is 9 bytes. Compare to `lib -> usr/lib` which is 7 bytes.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| `-l` from `ls -la` | You see just names — can't distinguish symlinks from directories |
| `\| grep lib64` | You scroll through 25+ entries trying to find the two `lib*` lines |
| The trailing `/` | `ls -la` lists your current dir, not root — you'd miss `/lib64` entirely |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Shows `drwxr-xr-x` instead of `l...` | Pre-UsrMerge system (CentOS 7, Debian 9) — `/lib64` is a real directory |
| Target is `/usr/lib64` (absolute) instead of `usr/lib64` (relative) | Both legal; relative is preferred so the symlink works inside chroots and containers |

---

### Task 1.3 — Confirm you're on a 64-bit system

**Purpose:** `/lib64` only matters if your CPU is 64-bit. Prove it.

```bash
uname -m
arch
getconf LONG_BIT
```

**Expected output:**

```
x86_64
x86_64
64
```

**Switches**

| Token | Meaning |
|---|---|
| `uname -m` | **M**achine architecture — what kind of CPU the kernel is running on |
| `arch` | Same answer as `uname -m`, shorter command |
| `getconf LONG_BIT` | Asks the C library how many bits are in a `long int` — 64 on a 64-bit ABI |

**Output decoded**

| Token | Meaning |
|---|---|
| `x86_64` | Intel/AMD 64-bit (sometimes written `amd64`) |
| `64` | The C ABI is 64-bit |

**Other architectures you'll see in production:**

| `uname -m` | What it is | Lib path |
|---|---|---|
| `x86_64` | Intel/AMD 64-bit | `/lib64` |
| `aarch64` | ARM 64-bit (Raspberry Pi 4+, AWS Graviton, Apple M1 VMs) | `/lib64` (still!) |
| `ppc64le` | IBM POWER little-endian | `/lib64` |
| `s390x` | IBM Z mainframe | `/lib64` |
| `i686` / `i386` | 32-bit Intel | `/lib` only — no `/lib64` |
| `armv7l` | 32-bit ARM | `/lib` only |

> **Important:** `/lib64` is the **same directory name** across all 64-bit architectures. But the libraries inside are **architecture-specific** — an x86-64 `libc.so.6` will not run on an aarch64 system. The directory name says nothing about which 64-bit CPU.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `uname -m` says `i686` but `/lib64` exists | You're on a 32-bit userland running on a 64-bit kernel — rare, usually old embedded systems |
| `getconf` not found | On a stripped container — `uname -m` is enough to confirm arch |

---

### Task 1.4 — Find the dynamic linker — the one binary that starts them all

**Purpose:** Meet `ld-linux-x86-64.so.2`. This is the most important file on a 64-bit Linux system after the kernel itself.

```bash
ls -la /lib64/ld-linux-x86-64.so.2
file /lib64/ld-linux-x86-64.so.2
file -L /lib64/ld-linux-x86-64.so.2
```

**Expected output:**

```
lrwxrwxrwx. 1 root root 10 May 11 14:02 /lib64/ld-linux-x86-64.so.2 -> ld-2.34.so
/lib64/ld-linux-x86-64.so.2: symbolic link to ld-2.34.so
/lib64/ld-linux-x86-64.so.2: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked (uses shared libs), ...
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -la` | Long listing with all entries |
| `file` | Identify a file by magic bytes |
| `file -L` | Follow symlinks — report on the **target** |

**Output decoded**

| Token | Meaning |
|---|---|
| `lrwxrwxrwx` | Symlink |
| `-> ld-2.34.so` | Points to the actual linker binary (glibc 2.34 build) |
| `ELF 64-bit LSB shared object` | A 64-bit shared library on a little-endian system |
| `x86-64` | x86-64 machine code |

> **The Story:** Every dynamically-linked 64-bit binary on your system has a "shebang-equivalent" for the kernel — a special field in its ELF header called the **interpreter** (`PT_INTERP`). On x86-64 Linux, that field always says `/lib64/ld-linux-x86-64.so.2`. When you run `ls`, the kernel reads `/bin/ls`, sees the interpreter field, and **runs the linker first** — passing `ls` as an argument. The linker loads `ls`, finds and maps every `.so` it needs from `/lib64`, patches in function addresses, and only then jumps into `ls`'s entry point. You never see this happen. Every command, every shell, every service — they all start with `ld-linux-x86-64.so.2` running first.

**Prove it — read the interpreter field of any binary:**

```bash
readelf -l /bin/ls | grep -i interp -A 1
```

**Expected output:**

```
INTERP    0x0000000000000318  0x0000000000400318  0x0000000000400318
          0x000000000000001c  0x000000000000001c  R      0x1
          [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

That `[Requesting program interpreter: ...]` line is the **literal address** the kernel uses to bootstrap `/bin/ls`. Change it (or delete the file it points to), and `ls` simply will not start.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ld-linux-x86-64.so.2` missing | **Bricked system.** No dynamically-linked binary will start. Boot rescue media |
| `file` says `i386` instead of `x86-64` | Wrong architecture installed — system was probably restored from a 32-bit backup |

---

# 🔧 Lab 2 — Inspect 64-bit Libraries

**Goal:** Look closely at the most important `.so` files in `/lib64` and learn to read their architecture metadata.

---

### Task 2.1 — Plain `ls /lib64` and read the patterns

**Purpose:** Unlike `/lib` (which is mostly directories), `/lib64` is mostly **files** — thousands of versioned `.so` libraries.

```bash
ls /lib64 | head -20
ls /lib64 | wc -l
```

**Expected output (truncated):**

```
ld-2.34.so
ld-linux-x86-64.so.2
libabsl_bad_any_cast_impl.so.2103.0.1
libabsl_bad_any_cast_impl.so.2103.0.2
libabsl_base.so.2103.0.1
libacl.so.1
libacl.so.1.1.2301
libapr-1.so.0
libapr-1.so.0.7.0
libargon2.so.1
libargon2.so.1.0.0
libasound.so.2
libasound.so.2.0.0
libattr.so.1
libattr.so.1.1.2501
libaudit.so.1
libaudit.so.1.0.0
libavahi-client.so.3
libavahi-client.so.3.5.9
libavahi-common.so.3

1842
```

**Switches**

| Token | Meaning |
|---|---|
| `ls /lib64` | List contents |
| `\| head -20` | First 20 lines |
| `\| wc -l` | Total count |

**Output decoded — the naming pattern**

| Pattern | Meaning |
|---|---|
| `libfoo.so` | Compile-time symlink — what developers link against |
| `libfoo.so.N` | Runtime soname — what binaries record in their ELF |
| `libfoo.so.N.M.P` | Real file — actual versioned library on disk |

**Example chain — three layers:**

```
libacl.so          (not always present in /lib64 — sometimes only in -devel package)
libacl.so.1        → libacl.so.1.1.2301    ← runtime symlink (loader uses this)
libacl.so.1.1.2301                          ← the actual binary file
```

> **The Story:** The triple-name scheme lets the system upgrade libraries without breaking anything. When glibc gets patched from 2.34 to 2.35:
> 1. The new file `libc-2.35.so` is dropped into `/lib64`
> 2. The symlink `libc.so.6` is re-pointed from `libc-2.34.so` to `libc-2.35.so`
> 3. The old file can be left or removed
>
> Every binary on disk that says "I need libc.so.6" automatically picks up the new code on its next launch — without recompilation.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Counts vary wildly between systems | Normal — minimal containers may have ~50 files, full RHEL desktop ~3000 |
| You see files with `.so` and no version | Development headers package (`*-devel`) was installed — those are linker symlinks |

---

### Task 2.2 — Verify a library's architecture with `file`

**Purpose:** Always confirm a `.so` is the right architecture before troubleshooting. A wrong-architecture library is the #1 cause of "library not found" lies.

```bash
file /lib64/libc.so.6
file -L /lib64/libc.so.6
file /lib64/libssl.so.3
```

**Expected output:**

```
/lib64/libc.so.6: symbolic link to libc.so.6
/lib64/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, BuildID[sha1]=..., not stripped
/lib64/libssl.so.3: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=..., stripped
```

**Output decoded — every token in the ELF line**

| Token | Meaning |
|---|---|
| `ELF` | **E**xecutable and **L**inkable **F**ormat — Linux binary standard |
| `64-bit` | 64-bit address space (8-byte pointers) |
| `LSB` | **L**east **S**ignificant **B**yte first — little-endian (x86 family) |
| `shared object` | A `.so` — not standalone, loaded by other binaries |
| `x86-64` | Machine type — Intel/AMD 64-bit |
| `version 1 (GNU/Linux)` | OS ABI tag — built for Linux's syscall conventions |
| `dynamically linked` | This library itself depends on other libraries |
| `BuildID[sha1]=...` | Unique hash for matching against `debuginfo` packages |
| `not stripped` / `stripped` | Whether symbol info was removed (stripped = smaller, no easy debugging) |

> **The bouncer for `file`:** `file` reads only the first ~256 bytes of the file. Those bytes contain the **ELF magic number** (`\x7fELF`), the architecture byte, the endianness byte, the OS ABI byte, and so on. `file` decodes those bytes and prints a human-readable description — it does not run the file, parse the rest, or trust the filename. That's why `file` is safe to run on untrusted binaries.

**Verify with the lower-level tool `readelf`:**

```bash
readelf -h /lib64/libc.so.6 | grep -E 'Class|Data|Machine'
```

**Expected output:**

```
  Class:                             ELF64
  Data:                              2's complement, little endian
  Machine:                           Advanced Micro Devices X86-64
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `file` says `ELF 32-bit` for a file in `/lib64` | Wrong file location — that's a 32-bit lib, should be in `/lib`. Reinstall the package |
| `file` says `Machine: ARM aarch64` on an x86 system | Disk image from a different architecture mounted in — investigate |
| `not stripped` (debug symbols present) | Larger file but allows easier debugging — normal for some libs |

---

### Task 2.3 — Use `nm` to list a library's exported symbols

**Purpose:** Every `.so` exports functions for other programs to call. Look at what `libc.so.6` actually offers.

```bash
nm -D /lib64/libc.so.6 | grep ' T ' | head -10
nm -D /lib64/libc.so.6 | wc -l
```

**Expected output:**

```
00000000000a8230 T a64l
00000000000c2e90 T abort
00000000000a01b0 T abs
00000000000c3590 T accept
00000000000c35e0 T accept4
00000000000b78a0 T access
00000000000c1c40 T acct
00000000000c0290 T addmntent
000000000003ed20 T adjtime
000000000003ee70 T adjtimex

2511
```

**Switches**

| Token | Meaning |
|---|---|
| `nm` | List **n**a**m**es (symbols) from object files |
| `-D` | **D**ynamic symbols — what's exported for runtime linking |
| `' T '` | grep pattern — `T` marks **text section** symbols (i.e., functions) |
| `\| wc -l` | Count |

**Output decoded — every column**

| Column | Meaning |
|---|---|
| `00000000000a8230` | Symbol's address inside the library (offset from base load address) |
| `T` | **T**ext segment — this is an exported function (lowercase `t` = local function) |
| `abort` | The function name — what callers reference |

**Symbol type codes (full table):**

| Code | Meaning |
|---|---|
| `T` / `t` | Function (uppercase = global, lowercase = local) |
| `D` / `d` | Initialized data |
| `B` / `b` | Uninitialized data (BSS) |
| `R` / `r` | Read-only data |
| `U` | Undefined — this symbol is **needed** but not provided here (must come from another library) |
| `W` | Weak — overridable by a stronger definition elsewhere |

> **Why a sysadmin cares:** When a binary fails with `undefined symbol: foo`, the binary expects `foo` to be in some library but isn't finding it. `nm -D /path/to/lib*.so | grep ' foo$'` tells you which library to install or upgrade.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `nm: ... no symbols` | Library was stripped of symbols — install the corresponding `-debuginfo` package |
| Output is overwhelmingly long | Pipe to `grep <function>` to find specific symbols |

---

# 🔧 Lab 3 — Run the Dynamic Linker Directly

**Goal:** Treat `ld-linux-x86-64.so.2` as a first-class program. Understand the magic that happens before `main()` ever runs.

---

### Task 3.1 — Run the linker as a standalone command

**Purpose:** The dynamic linker isn't just a library — it can be **executed directly**. Doing so reveals its built-in help and version.

```bash
/lib64/ld-linux-x86-64.so.2 --version
/lib64/ld-linux-x86-64.so.2 --help | head -20
```

**Expected output:**

```
ld.so (GNU libc) stable release version 2.34.
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.

Usage: ld.so [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
You have invoked `ld.so', the program interpreter for dynamically-linked
ELF programs.  Usually, the program interpreter is invoked automatically
when a dynamic executable is started.

You may invoke the program interpreter program directly from the command
line to launch the EXECUTABLE-FILE as if it had been invoked normally...
```

**Output decoded**

| Token | Meaning |
|---|---|
| `ld.so (GNU libc) stable release version 2.34` | The linker is part of glibc; this is glibc 2.34's linker |
| `program interpreter for dynamically-linked ELF programs` | The linker's official self-description |
| `EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]` | Yes — you can invoke any binary **through** the linker manually |

> **The Story:** Most users have no idea this works. The linker advertises itself as a callable program because that's how you debug binaries whose interpreter field is broken — you call the linker manually and pass the binary as an argument. The kernel does this automatically every time you run any dynamically-linked program, but the same machinery is available to you on the command line.

---

### Task 3.2 — Use the linker to inspect a binary's needs

**Purpose:** `--list` does what `ldd` does — but it's the actual linker telling you the truth, not a shell-script wrapper.

```bash
/lib64/ld-linux-x86-64.so.2 --list /bin/ls
ldd /bin/ls
```

**Expected output (both produce the same dependency listing):**

```
linux-vdso.so.1 (0x00007ffc8f9d6000)
libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f3a4c200000)
libcap.so.2 => /lib64/libcap.so.2 (0x00007f3a4c100000)
libc.so.6 => /lib64/libc.so.6 (0x00007f3a4c000000)
libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007f3a4be00000)
/lib64/ld-linux-x86-64.so.2 (0x00007f3a4c4d0000)
```

**Switches**

| Token | Meaning |
|---|---|
| `--list` | Tell the linker to print the dependency list and exit (don't actually run the binary) |
| `/bin/ls` | The binary to inspect |

**Why this matters — `ldd` vs the linker directly:**

> `ldd` is a shell script (look at `cat $(which ldd)` — it's plain bash) that sets some environment variables and invokes the linker. On a binary from an **untrusted source**, `ldd` actually **runs** the binary briefly (with `LD_TRACE_LOADED_OBJECTS=1`). This is a known security hole — malicious binaries can override that flag and execute arbitrary code when you run `ldd` on them. Calling the linker directly with `--list` is the **safe** way.

**Security demo (don't actually run on untrusted files):**

```bash
ldd ./suspicious-binary             # ⚠️ can execute hostile code
/lib64/ld-linux-x86-64.so.2 --list ./suspicious-binary  # ✅ safe
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `--list` and `ldd` give different output | Different glibc versions installed — verify with `/lib64/ld-linux-x86-64.so.2 --version` |

---

### Task 3.3 — Override search path with `--library-path`

**Purpose:** Force the linker to look only in a specific directory. Useful for testing, for chroots, and for proving things break in controlled ways.

```bash
/lib64/ld-linux-x86-64.so.2 --library-path /lib64 /bin/ls /etc | head -3
/lib64/ld-linux-x86-64.so.2 --library-path /nonexistent /bin/ls /etc 2>&1 | head -3
```

**Expected output (first command — works):**

```
adjtime
alternatives
audit
```

**Expected output (second command — controlled failure):**

```
/bin/ls: error while loading shared libraries: libselinux.so.1: cannot open shared object file: No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `--library-path PATH` | **Replace** the standard search path with `PATH` |
| `/bin/ls /etc` | The binary and its arguments — passed through to the binary after the linker is done |

**Output decoded**

| Result | Why |
|---|---|
| First call succeeds | `/lib64` is the real location of all needed libs — linker finds them |
| Second call fails | `/nonexistent` has no libraries; linker reports the first missing one and stops |

> **The bouncer breakdown of the second invocation:**
> 1. You told the linker: "search only `/nonexistent`"
> 2. Linker reads `/bin/ls`, sees the dependency list
> 3. First dep: `libselinux.so.1` — linker checks `/nonexistent/libselinux.so.1` — not there
> 4. Linker does NOT fall back to `/lib64` because you told it to use ONLY `/nonexistent`
> 5. Linker prints the standard error and exits with status 127

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| `--library-path` | Linker uses defaults — both invocations succeed; the demo loses its point |
| The argument after `--library-path` | Linker has no path to search — same effect as `/nonexistent` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| First invocation also fails | Your `/lib64` is broken — `ls /lib64/libselinux.so.1` should exist |
| Want to **append** instead of replace | Use the `LD_LIBRARY_PATH` env var: `LD_LIBRARY_PATH=/extra:/dirs /bin/ls` |

---

# 🔧 Lab 4 — Multilib: /lib vs /lib64

**Goal:** Understand exactly when you'd see 32-bit libraries alongside 64-bit ones, and how the system keeps them separate.

---

### Task 4.1 — Check if multilib is enabled

**Purpose:** Default RHEL/Fedora installations are 64-bit-only. To get 32-bit support, you have to install `glibc.i686`.

```bash
ls /lib/libc.so.6 2>/dev/null
ls /lib64/libc.so.6
rpm -qa | grep -E 'glibc\..*' | sort
```

**Expected output (pure 64-bit system):**

```
(empty — no /lib/libc.so.6)
/lib64/libc.so.6
glibc-2.34-100.el9_5.x86_64
glibc-common-2.34-100.el9_5.x86_64
glibc-langpack-en-2.34-100.el9_5.x86_64
```

**Expected output (multilib-enabled system):**

```
/lib/libc.so.6
/lib64/libc.so.6
glibc-2.34-100.el9_5.i686
glibc-2.34-100.el9_5.x86_64
glibc-common-2.34-100.el9_5.x86_64
```

**Switches**

| Token | Meaning |
|---|---|
| `2>/dev/null` | Silently discard the "no such file" error if the path doesn't exist |
| `rpm -qa` | **Q**uery **a**ll installed packages |
| `grep -E 'glibc\..*'` | Extended regex — match `glibc.` followed by anything (catches the arch suffix) |

**Output decoded**

| Field | Meaning |
|---|---|
| `glibc-...x86_64` | The 64-bit glibc — present on every 64-bit system |
| `glibc-...i686` | The 32-bit glibc — present **only** if multilib is enabled |
| `glibc-common` | Architecture-independent files (locales, timezone data) — only one needed |

> **The Story:** "Multilib" means a 64-bit Linux system that can also run 32-bit binaries. The kernel supports this natively — you just need the 32-bit libraries installed. RPM differentiates by the **arch suffix** (`.x86_64` vs `.i686`), so both versions of `glibc` can be installed side-by-side without filename conflicts: 64-bit files land in `/lib64`, 32-bit files in `/lib`.

**Enable multilib if you need it:**

```bash
sudo dnf install glibc.i686
ls /lib/libc.so.6  # now exists
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Trying to run a 32-bit binary and getting `No such file or directory` | The binary is 32-bit; install multilib: `sudo dnf install glibc.i686` |
| The error mentions `/lib/ld-linux.so.2` (not `/lib64/ld-linux-x86-64.so.2`) | Same root cause — the 32-bit linker is missing |

---

### Task 4.2 — Compare 32-bit and 64-bit version of the same library

**Purpose:** Prove they are different files with different machine code.

```bash
ls -la /lib/libc.so.6 2>/dev/null
ls -la /lib64/libc.so.6
file /lib/libc.so.6 2>/dev/null
file /lib64/libc.so.6
```

**Expected output (multilib system):**

```
lrwxrwxrwx. 1 root root 12 May 11 14:02 /lib/libc.so.6 -> libc-2.34.so
lrwxrwxrwx. 1 root root 12 May 11 14:02 /lib64/libc.so.6 -> libc.so.6
/lib/libc.so.6:   ELF 32-bit LSB shared object, Intel 80386, ...
/lib64/libc.so.6: ELF 64-bit LSB shared object, x86-64, ...
```

**Output decoded**

| Difference | What it means |
|---|---|
| `32-bit` vs `64-bit` | Different pointer size — completely different machine code |
| `Intel 80386` vs `x86-64` | Different CPU instruction set (the 32-bit one targets a CPU from 1985) |

> **Why the names match but the contents differ:** Both libraries export the same C functions (`printf`, `malloc`, `fopen`, ...) — but the **machine code** that implements them differs because 32-bit and 64-bit programs use different calling conventions, register layouts, and pointer sizes. Same API, different ABI.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| 32-bit `libc.so.6` doesn't exist on your system | Pure 64-bit install — that's normal and recommended unless you need to run legacy software |

---

### Task 4.3 — Check which architectures `ldconfig` knows about

**Purpose:** `ldconfig` tags each cached library with its ABI. See how it distinguishes 32-bit from 64-bit.

```bash
ldconfig -p | head -3
ldconfig -p | grep 'libc\.so\.6'
ldconfig -p | grep -c 'x86-64'
ldconfig -p | grep -c 'libc6,'
```

**Expected output:**

```
1842 libs found in cache `/etc/ld.so.cache'
        libc.so.6 (libc6,x86-64, OS ABI: Linux 3.2.0) => /lib64/libc.so.6
        libc.so.6 (libc6, OS ABI: Linux 3.2.0) => /lib/libc.so.6
1842
1843
```

**Output decoded — the parenthetical tags**

| Tag | Meaning |
|---|---|
| `(libc6, x86-64, ...)` | 64-bit library, glibc 2.x ABI |
| `(libc6, ...)` with **no** arch | 32-bit library (the absence of `x86-64` is the marker) |
| `OS ABI: Linux 3.2.0` | Minimum kernel version this library is built for |

> **The bouncer rule:** When a 64-bit binary asks for `libc.so.6`, the linker consults the cache, finds **two** entries with that soname, but only picks the one tagged `x86-64` because the binary itself is x86-64. A 32-bit binary asking for the same name picks the other one. **The cache is architecture-aware.**

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Only one `libc.so.6` line | Not a multilib system — that's fine |
| Two lines but both have the same path | Cache is corrupt — `sudo ldconfig` to rebuild |

---

# 🔧 Lab 5 — Real-World 64-bit Troubleshooting

**Goal:** Diagnose and fix the architecture-mismatch errors that look like ordinary "library not found" errors but aren't.

---

### Task 5.1 — Identify a wrong-architecture binary

**Purpose:** Sometimes the error message is misleading. The library exists — but it's the wrong architecture.

**Scenario:** Someone copied an x86-64 binary onto an aarch64 system. They try to run it:

```bash
./mystery-binary
```

**Expected output:**

```
bash: ./mystery-binary: cannot execute: required file not found
```

**The misleading part:** This **sounds** like a missing-library error. But the binary itself exists. The "required file not found" is the kernel's confusing way of saying **"the interpreter listed in the ELF header doesn't exist on this system."**

**Diagnose:**

```bash
file ./mystery-binary
uname -m
readelf -l ./mystery-binary | grep -i interp
```

**Expected output:**

```
./mystery-binary: ELF 64-bit LSB executable, x86-64, ...
aarch64
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

**Output decoded**

| Token | Meaning |
|---|---|
| Binary is `x86-64` | Built for Intel/AMD 64-bit |
| System is `aarch64` | ARM 64-bit |
| Interpreter is `/lib64/ld-linux-x86-64.so.2` | A binary that **does not exist** on aarch64 systems (theirs is `/lib/ld-linux-aarch64.so.1`) |

> **The Story:** The kernel saw an ELF binary, read its interpreter field, tried to load `/lib64/ld-linux-x86-64.so.2`, didn't find it (because aarch64 systems have `/lib/ld-linux-aarch64.so.1` instead), and gave you the unhelpful "required file not found" error. Every byte of the binary was readable. Every dependency was installable. But the **architecture** was wrong and there's no fix short of replacing the binary with a native build.

**Fix:**

```bash
sudo dnf install <package>  # for the correct arch
```

Or, if you absolutely must run it, install an emulator:

```bash
sudo dnf install qemu-user-static
./mystery-binary  # qemu transparently translates x86-64 to aarch64
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cannot execute: required file not found` despite the binary clearly existing | Architecture mismatch — `file` the binary and `uname -m` the system |
| Same error on a fresh install | Multilib not installed for a 32-bit binary on 64-bit system — install `glibc.i686` |

---

### Task 5.2 — Fix a missing 64-bit library end-to-end

**Purpose:** The real, ordinary case — a service won't start because one `.so` is gone from `/lib64`.

**Scenario:**

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

**Expected output (failure):**

```
× httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled)
     Active: failed (Result: exit-code)
   Process: 12345 ExecStart=/usr/sbin/httpd ... (code=exited, status=127)
httpd: error while loading shared libraries: libssl.so.3: cannot open shared object file: No such file or directory
```

**Five-command fix:**

```bash
ldd /usr/sbin/httpd | grep 'not found'
sudo dnf provides '*/libssl.so.3'
sudo dnf reinstall openssl-libs
sudo ldconfig
sudo systemctl start httpd && sudo systemctl status httpd
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `ldd \| grep 'not found'` | Confirm which `.so` is missing — could be more than one |
| `dnf provides '*/libssl.so.3'` | Find the package that ships this exact file |
| `dnf reinstall openssl-libs` | Re-place the missing file from the package |
| `sudo ldconfig` | Refresh `/etc/ld.so.cache` so the new file is indexed |
| Restart and verify | Confirm the original failure is gone |

**Expected final status:**

```
● httpd.service - The Apache HTTP Server
     Active: active (running)
```

> **The Story:** Real troubleshooting is short. Read the error, identify the missing file, identify the package, reinstall, refresh cache, retry. Five commands. The discipline is doing them in **this order** — `ldconfig` after `dnf` matters because the cache only updates on package install if `%post` runs it (and sometimes it doesn't). Running it manually is a safe extra step.

---

### Task 5.3 — Audit a process's actual loaded 64-bit libraries

**Purpose:** `ldd` shows compile-time. `/proc/<pid>/maps` shows what's loaded **right now**. Useful for confirming a fix took effect without restarting everything.

```bash
pgrep -x httpd | head -1 | xargs -I {} cat /proc/{}/maps | grep '/lib64' | awk '{print $NF}' | sort -u | head -10
```

**Expected output:**

```
/usr/lib64/libapr-1.so.0.7.0
/usr/lib64/libaprutil-1.so.0.6.1
/usr/lib64/libc.so.6
/usr/lib64/libcrypt.so.2.0.0
/usr/lib64/libcrypto.so.3.0.7
/usr/lib64/libexpat.so.1.8.7
/usr/lib64/libpcre2-8.so.0.11.0
/usr/lib64/libssl.so.3.0.7
/usr/lib64/libsystemd.so.0.33.0
/usr/lib64/libz.so.1.2.11
```

**The full pipeline — bouncer breakdown**

| Stage | Job |
|---|---|
| `pgrep -x httpd` | Find PIDs matching the exact name `httpd` |
| `\| head -1` | Pick the first one (typically the parent process) |
| `\| xargs -I {} cat /proc/{}/maps` | Build `cat /proc/<pid>/maps` and run it |
| `\| grep '/lib64'` | Keep only lines mentioning `/lib64` |
| `\| awk '{print $NF}'` | Print only the **last** field of each line (the path) |
| `\| sort -u` | De-duplicate (each lib appears multiple times in `maps` — one per memory segment) |
| `\| head -10` | First 10 |

**Why this is better than `ldd /usr/sbin/httpd`:**

- `ldd` shows **possible** dependencies at compile time
- `/proc/<pid>/maps` shows what **actually got loaded** at runtime
- httpd loads **modules** via `LoadModule` directives — those `.so` files only appear in `/proc/.../maps`, not in `ldd`

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `pgrep` returns nothing | Service isn't running — start it first |
| `Permission denied` on `/proc/<pid>/maps` | Run with `sudo` — another process's memory map is protected |
| Pipeline shows libs in `/lib` (not `/lib64`) | Service is 32-bit — verify with `file /usr/sbin/httpd` |

---

### Task 5.4 — Exam-style scenario: mixed troubleshooting

**Task statement (RHCSA-style):** *"A custom application installed to `/opt/myapp/bin/myapp` fails with 'error while loading shared libraries: libcustom.so.1'. The library file is at `/opt/myapp/lib/libcustom.so.1.0.0`. Make the application start successfully without modifying its binary or the library file."*

**Solution:**

```bash
ls /opt/myapp/lib/
file /opt/myapp/lib/libcustom.so.1.0.0
file /opt/myapp/bin/myapp
sudo ln -sf /opt/myapp/lib/libcustom.so.1.0.0 /opt/myapp/lib/libcustom.so.1
echo "/opt/myapp/lib" | sudo tee /etc/ld.so.conf.d/myapp.conf
sudo ldconfig
ldconfig -p | grep libcustom
/opt/myapp/bin/myapp --version
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `ls /opt/myapp/lib/` | See what's there — confirms only the versioned file exists |
| `file` the lib | Verify it's `ELF 64-bit x86-64` so it matches the binary |
| `file` the binary | Verify it's also 64-bit — architecture must match |
| `ln -sf libcustom.so.1.0.0 libcustom.so.1` | Create the soname symlink the binary expects |
| `echo .../myapp.conf` | Tell `ldconfig` to scan `/opt/myapp/lib` |
| `sudo ldconfig` | Rebuild the cache |
| `ldconfig -p \| grep libcustom` | Verify the library is now cached |
| Run the binary | Confirm the fix |

**Expected final output:**

```
myapp version 1.2.3
```

> **The Story:** This is the most common "custom software" scenario on RHEL servers. Vendors ship `.tar.gz` archives whose `.so` files don't follow the standard install pattern. You **never** modify the binary or the library — you give the linker the context it needs by:
> 1. Creating the soname symlink (binaries link against `libcustom.so.1`, not `.so.1.0.0`)
> 2. Telling `ldconfig` about the new directory
> 3. Refreshing the cache
>
> Three changes outside the vendor files. Everything works.

---

## ✅ Lab Checklist — 5 Labs, 18 Tasks

**Lab 1 — Navigate**
- [ ] 1.1 `cd /lib64 && pwd && pwd -P` shows the symlink
- [ ] 1.2 `ls -la / \| grep lib64` shows `lib64 -> usr/lib64`
- [ ] 1.3 `uname -m`, `arch`, `getconf LONG_BIT` all confirm 64-bit
- [ ] 1.4 Found and identified `/lib64/ld-linux-x86-64.so.2`

**Lab 2 — Inspect libraries**
- [ ] 2.1 `ls /lib64 \| head -20` shows `.so` versioned filenames
- [ ] 2.2 `file /lib64/libc.so.6` confirms `ELF 64-bit ... x86-64`
- [ ] 2.3 `nm -D /lib64/libc.so.6` reveals exported symbols

**Lab 3 — Dynamic linker**
- [ ] 3.1 `/lib64/ld-linux-x86-64.so.2 --version` works
- [ ] 3.2 `--list /bin/ls` matches `ldd /bin/ls`
- [ ] 3.3 `--library-path /nonexistent /bin/ls` produces a controlled failure

**Lab 4 — Multilib**
- [ ] 4.1 Identified whether multilib is enabled with `rpm -qa \| grep glibc`
- [ ] 4.2 Compared 32-bit and 64-bit `libc.so.6` with `file`
- [ ] 4.3 `ldconfig -p` shows the `x86-64` ABI tags

**Lab 5 — Troubleshooting**
- [ ] 5.1 Identified an architecture mismatch with `file` + `uname -m`
- [ ] 5.2 Five-command fix: `ldd` → `dnf provides` → `dnf reinstall` → `ldconfig` → restart
- [ ] 5.3 Read `/proc/<pid>/maps` to see what's loaded right now
- [ ] 5.4 Solved the exam scenario: soname symlink + `ld.so.conf.d` + `ldconfig`

---

## 🧠 /lib vs /lib64 — Architecture Cheat Sheet

| Question | `/lib` | `/lib64` |
|---|---|---|
| **What's inside?** | 32-bit `.so` files + arch-independent dirs | 64-bit `.so` files |
| **Default on RHEL 9?** | Symlink to `/usr/lib` (mostly empty of `.so` files on pure 64-bit) | Symlink to `/usr/lib64` (where the action is) |
| **Used by 64-bit binaries?** | Only for arch-independent content | **Yes — primary location** |
| **Used by 32-bit binaries?** | **Yes — primary location** | No (wrong architecture) |
| **Contains the dynamic linker?** | 32-bit linker: `/lib/ld-linux.so.2` | 64-bit linker: `/lib64/ld-linux-x86-64.so.2` |
| **Required to install separately?** | Yes for multilib: `glibc.i686` | No — default on every 64-bit install |

---

## ⚠️ Common Pitfalls

| Mistake | Fix |
|---|---|
| Confusing "library not found" with "interpreter not found" | Read the error carefully — `cannot execute: required file not found` = wrong arch interpreter |
| Copying x86-64 binaries to aarch64 (or vice versa) | Reinstall the package on the target architecture, or use `qemu-user-static` |
| Manually placing a 32-bit `.so` in `/lib64` | `ldconfig` will refuse to cache it (wrong ABI tag); the file is invisible to 64-bit binaries |
| Running `ldd` on untrusted binaries | Security risk — use `/lib64/ld-linux-x86-64.so.2 --list` instead |
| Deleting `/lib64/ld-linux-x86-64.so.2` "to clean up" | **Bricks the entire system.** Boot rescue media to restore |
| Mixing `LD_LIBRARY_PATH` and `/etc/ld.so.conf.d/` | `LD_LIBRARY_PATH` overrides; prefer the system file for permanent changes |
| Expecting `/lib` and `/lib64` to be the same on all distros | Pre-UsrMerge systems keep them as separate real directories |
| Forgetting `sudo ldconfig` after dropping a `.so` into a configured path | Cache stays stale; binaries report "not found" even though the file exists |

---

## 📌 Exam Tips (RHCSA / RHCE / CKA)

- **RHCSA EX200:** Know that on RHEL 9, `/lib64 -> usr/lib64` is a symlink. Confirm with `ls -la /`.
- **RHCSA EX200:** The five-command library fix (`ldd → dnf provides → dnf reinstall → ldconfig → re-test`) is the standard answer for any "this binary won't start" task.
- **RHCSA EX200:** Memorize the interpreter path: `/lib64/ld-linux-x86-64.so.2`. If a `file` output mentions this, the binary is x86-64.
- **RHCE EX294 (Ansible):** When playbooks fail because a Python C extension can't load, the missing `.so` is usually in `/usr/lib64/python3.X/site-packages/...` — use `find` to locate it.
- **CKA:** Container images often strip `/lib64` aggressively to save space. `kubectl exec` failures with "no such file" usually mean a missing 64-bit lib inside the container.
- **RHCA RH342:** First `journalctl` line that says `error while loading shared libraries: libXXX` → immediately `dnf provides '*/libXXX'` on the host.
- **Security audit:** Never run `ldd` on a binary downloaded from the internet — use `/lib64/ld-linux-x86-64.so.2 --list` instead.

---

## 🔍 /lib64 Decision Guide

```
What architecture is this system?  → uname -m
What architecture is this binary?  → file /path/to/binary
What interpreter does it need?     → readelf -l /path/to/binary | grep interp
What libs are loaded right now?    → cat /proc/<pid>/maps | grep '/lib64'
What does the system know about?   → ldconfig -p | grep <libname>
What package owns this .so?        → rpm -qf /lib64/libX.so.Y
What package would provide it?     → dnf provides '*/libX.so.Y'
What does ld.so search?            → cat /etc/ld.so.conf.d/*.conf
Refresh the cache?                 → sudo ldconfig

Wrong arch?    → install the right package, or use qemu-user-static
Missing .so?   → dnf provides → dnf install → ldconfig
Wrong soname?  → ln -s <real-file> <soname-symlink> → ldconfig
```


## 🔗 Part of Linux Ops Mastery

- [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)
- [Linux Ops Mastery](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
