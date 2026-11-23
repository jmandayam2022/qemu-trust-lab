# Concepts glossary, mapped to lab tasks

Each term lists what it means and where it appears in the backlog. Terms are grouped by the day that introduced them. Items marked *(verify)* were expected but not yet confirmed on this checkout (OP-TEE 4.11.0, `qemu_v8.xml`).

Add to this file as new terms come up.

## Big picture: how the pieces fit

```
Mac (macOS, Apple M3 Max)
  Docker Desktop app
    hidden Linux VM (linuxkit kernel)
      Docker engine
        container "optee-dev" (Ubuntu 22.04 userland)
          /optee  <- Docker volume "optee-src"
          you build here, then run qemu-system-aarch64
            QEMU guest: BL1 -> BL2 -> BL31 -> OP-TEE (BL32) -> U-Boot (BL33) -> Linux
```

Build host: the container. Target board: the QEMU guest.

## Day 1: environment

| Term | Meaning | Where it shows up |
|---|---|---|
| Container | A normal Linux process with a private root filesystem, process list, network and resource limits. It shares the host kernel. | `docker run ... bash` gives a shell in one. |
| VM | A full virtual computer with its own kernel. Docker Desktop on a Mac runs one hidden Linux VM, and containers run inside it. | `uname -r` in the container shows `linuxkit`. |
| Image | A read-only stack of filesystem layers used as the template for containers. | `docker build` creates `qemu-trust-lab`. |
| Layer | A set of filesystem changes. Each Dockerfile `RUN` creates one. Layers are cached and shared. | Put slow, stable steps (the big `apt-get install`) before changing steps. |
| Dockerfile | The recipe an image is built from. | `docker/Dockerfile`, adapted from the OP-TEE Prerequisites page. |
| Userland vs kernel | Userland is the programs and libraries (`bash`, `gcc`, `apt`). The kernel manages hardware and processes. The container has Ubuntu userland on Docker's kernel. | `head -2 /etc/os-release` shows Ubuntu, `uname -r` shows linuxkit. |
| Namespace | A kernel feature that gives a process its own view of filesystems, PIDs or network. | Why the container has its own IP, `172.17.0.2`. |
| cgroup | A kernel feature that limits CPU, memory and I/O for a group of processes. | The VM limits you set in Docker Desktop. |
| Bridge network | Docker's default virtual switch (`172.17.0.0/16`). Containers get addresses on it and reach the internet through NAT. | `hostname -I` showed `172.17.0.2`. |
| `--platform linux/arm64` | Selects the arm64 image variant. Matches the M3 Max, so no CPU emulation for the container. | Day 1 build and run. |
| git remote | A named URL for a hosted copy of the repo. `origin` is the conventional name. | `git remote -v` shows where `push` goes. |
| `gh` | GitHub CLI. Manages GitHub objects (repos, PRs) and acts as git's credential helper. | `gh repo create ... --push` made the repo. |
| OAuth token (`gho_`) | Credential `gh` stores in the macOS Keychain after browser login. | `gh auth status`. Each GitHub host has its own separate login entry. |

## Day 2: getting the source

| Term | Meaning | Where it shows up |
|---|---|---|
| Docker volume | Storage managed by the Docker engine inside the VM. Survives container deletion and image rebuilds. Linux filesystem, so no case-insensitivity problem. | `docker volume create optee-src`, mounted with `-v optee-src:/optee`. |
| `repo` | Google tool that clones many git repositories listed in a manifest. | `repo init`, `repo sync`. |
| Manifest | XML file that lists repositories and the revision of each. | `qemu_v8.xml` from `OP-TEE/manifest`. |
| Pinning a tag | Fetching a fixed release instead of a moving branch, for reproducibility. | `-b 4.11.0`. `repo info` shows `Manifest merge branch: refs/heads/4.11.0`. |
| Annotated tag `^{}` | `git ls-remote` lists annotated tags twice. The `^{}` line is the commit the tag points to. Ignore it. | Choosing the release tag. |
| `.repo/` | Hidden directory where `repo` keeps manifest and project data. | `/optee/.repo/manifests/qemu_v8.xml`. |
| `<linkfile>` *(verify)* | Manifest entry that makes `repo sync` create a symlink, expected to make `build/Makefile` point to `qemu_v8.mk`. | Why `make` in `build/` builds the QEMU v8 board. |

## Day 3: toolchains and the build system

| Term | Meaning | Where it shows up |
|---|---|---|
| Toolchain | Compiler, assembler, linker and debugger for a target. | `make toolchains` fetches Arm GNU toolchain 14.3.rel1 into `/optee/toolchains/`. |
| Cross-toolchain | A toolchain that builds for a target different from the host. The prefix names the target (`aarch64-none-linux-gnu-`). | `toolchains/aarch64/bin` has both `aarch64-linux-gnu-*` and `aarch64-none-linux-gnu-*`. |
| `make -jN` | Run N build jobs in parallel. | Day 3 used `-j3`. Day 4 used `-j8`. |
| `grep -n` | Print the line number of each match. | Used to read `qemu_v8.mk`. |
| `sed -n 'A,Bp'` | Print only lines A to B of a file. | Reading around a `grep` hit. |
| `SPD` | TF-A build option that selects the Secure Payload Dispatcher in BL31. | `qemu_v8.mk` lines 239 to 259. |
| `opteed` | The SPD for OP-TEE. BL31 uses it to start BL32 and route SMCs to it. | Default here, because `SPMC_AT_EL ?= n`. |
| `spmd` | SPD for the FF-A SPM dispatcher. Used when `SPMC_AT_EL` is 1, 2 or 3. | Not used in the default build. |
| BL1, BL2, BL31, BL32, BL33 | TF-A boot stages: BL1 (ROM-like first stage), BL2 (loads others), BL31 (EL3 runtime), BL32 (secure OS, OP-TEE), BL33 (normal-world bootloader, U-Boot). | `/optee/out/bin/bl*.bin`. |
| SMC | Secure Monitor Call. How the normal world calls into EL3 and the secure world. | Traced end to end on Day 32. |
| EL0 to EL3 | Arm exception levels. EL3 is the most privileged and runs BL31. | Why QEMU needs full emulation for this lab. |

## Day 4: the build

| Term | Meaning | Where it shows up |
|---|---|---|
| `make` default goal | With no target, make builds the first rule or the one named `all`. | `all:` is in `common.mk:241` and `qemu_v8.mk:173,321`. Plain `make` builds without launching QEMU. |
| `time` | Prints real, user and sys time when a command ends. User is total CPU time. | Day 4: real 10m34s, user 80m41s. |
| `2>&1` | Redirect stderr (stream 2) to wherever stdout (stream 1) goes. | Capturing errors in the build log. |
| `\| tee file` | Copy a stream to the screen and a file. | `tee /optee/build-day4.log`. |
| `2>/dev/null` | Discard stderr. `/dev/null` is a file that throws away what is written to it. | `repo info 2>/dev/null \| head -5`. |
| pipe `\|` | Connect one command's stdout to the next command's stdin. | Used throughout. |
| `(ignored)` error | make prints this when a recipe is marked to continue after failure. Not a build failure. | Three e2fsprogs doc errors in the Day 4 log. |
| `mkimage` | U-Boot tool that wraps a file with a header (type, load address). | Final lines of the build: rootfs RAMDisk image at `0x45000000`. |
| OP-TEE pager | OP-TEE split into header, pager and pageable parts, loaded as BL32 plus extras. | `bl32.bin`, `bl32_extra1.bin`, `bl32_extra2.bin`. |

## Day 5 onward: QEMU (preview)

| Term | Meaning | Where it shows up |
|---|---|---|
| QEMU | Emulator that pretends to be a computer. The software it runs is the guest, and the machine running QEMU is the host. | Built from source by the lab. |
| `-machine virt,secure=on` | QEMU's generic Armv8 board with EL3 and the secure world enabled. | `qemu_v8.mk` line 684. |
| TCG | Tiny Code Generator. QEMU's software CPU emulation. Translates guest instructions to host instructions. Slow but gives full control. | Why boot and `xtest` are slow. |
| HVF | Apple's hardware virtualization used by QEMU. Fast, but guests run at EL1 and below only, so no EL3 or secure world. | Why this lab cannot use it. |
| `-serial tcp:127.0.0.1:PORT` | QEMU connects out to a TCP port as a client. Something must already be listening. | Ports 54320 (normal world) and 54321 (secure world), from `common.mk`. |
| `-s -S` | `-s` opens a GDB server on port 1234. `-S` halts the CPU at start. | Why QEMU waits for `c` at the `(qemu)` monitor. |
| `(qemu)` monitor | QEMU's control prompt. `c` continues execution. | Day 5. |
| `launch-terminal`, `wait-for-ports` | Helpers in `common.mk` used by `make run-only` to open terminal windows and wait for the ports. | Fail in a headless container. Day 9 replaces them. |
| `xtest` | OP-TEE's regression test suite, run in the normal world. | `xtest 4002` on Day 5. |

## Day 5-6: boot, consoles, and logs

| Term | Meaning | Where it shows up |
|---|---|---|
| tmux | Terminal multiplexer. A session holds windows, and a window holds panes. Keys: `Ctrl-b w` (window list), `Ctrl-b [` (scroll or copy mode), `Ctrl-b z` (zoom a pane), `Ctrl-b :` (tmux command prompt), `Ctrl-b c` (new window). | The lab runs inside a tmux session named `lab`. |
| `$TMUX` | Environment variable that tmux sets for every shell inside it. The build's `launch-terminal` helper checks it. | `echo $TMUX` must print a path before `make run-only`. |
| `launch-terminal`, `wait-for-ports` | Makefile macros in `build/common.mk` that open one console per UART port and wait until both accept connections. | In tmux they create a window named `OPTEE_<n>` with two panes. |
| `soc_term.py` | Console script run in each pane. Listens on a port, accepts QEMU's connection, shows the UART output, and (inferred from its log lines, script not read) sends keystrokes back. | Ports 54320 (normal world) and 54321 (secure world). |
| Socket and TCP endpoint | A socket is the OS handle for one end of a connection. An endpoint is an IP address plus a port. | QEMU is the client, `soc_term.py` is the server, both on `127.0.0.1`. |
| `-serial tcp:127.0.0.1:PORT` | QEMU option that attaches an emulated UART to a TCP connection, with QEMU as the client. | First `-serial` is the first UART (normal world), second is the second UART (secure world). |
| PL011 UART | The serial port model used by QEMU `virt`. The kernel console is the PL011 at `0x9000000` (`ttyAMA0`). The secure-world UART is believed to be a second PL011 at `0x9040000` (not confirmed). | Day 6 annotation, N:220. |
| `nc -z host port` | Connects to a port and disconnects at once, exiting 0 if something was listening. | The helper's port check. A one-shot `nc -l` listener quits after this probe. |
| `-S` and `c` | `-S` freezes the CPU at startup. `c` at the `(qemu)` prompt continues, releasing the CPU so BL1 starts. | Like releasing a board's reset line. |
| `system_reset`, guest `reboot` | `system_reset` at `(qemu)` resets the machine. `reboot` in the guest ended in a full reset from BL1. | Day 5 reboot test. |
| `tmux capture-pane -p -J -S - -t <pane>` | Dumps a pane's whole scrollback. `-p` prints to stdout, `-J` joins wrapped lines, `-S -` starts at the top. | How the Day 6 boot logs were saved. |
| OP-TEE log prefix | `<level>/<component>:<cpu> <thread> <function>:<line> message`. Levels: `E` error (1), `I` info (2), `D` debug (3), `F` flow (4), and level 0 `TRACE_MIN` (the letter `M` is an inference). Confirmed in `trace_levels.h`. | `D/TC:`, `I/TC:`, `D/LD:` in the secure log. |
| Log components | `TC` is TEE Core, `LD` is `ldelf`, `TA` is a trusted application's own output. | Day 19 adds `TA` lines from your own TA. |
| `CFG_TEE_CORE_LOG_LEVEL`, `CFG_TEE_TA_LOG_LEVEL` | Build options for core and TA log verbosity (0 to 4). The `D` lines in the log show the core level is at least 3. | Day 19. Current value not yet looked up. |
| `ldelf` | User-mode ELF loader that OP-TEE runs to load a TA. | `D/LD: ldelf:...` lines. |
| TA store | Where OP-TEE looks for a TA binary. Order seen in the log: early TA (built into the OP-TEE image), Secure Storage TA, REE (the normal-world filesystem, served by `tee-supplicant`). | S:107-110, and the `crypt` TA lookup on Day 5. |
| PTA (pseudo TA) | A TA that runs inside the OP-TEE core, not as a user-mode TA. `device.pta` and `system.pta` appeared in the log. | S:153-157, S:201-207. |
| `tee-supplicant` | Normal-world daemon that serves OP-TEE's requests, such as loading TA files from `/lib/optee_armtz`. | Started in userspace, N:371. |
| REE FS | OP-TEE secure storage backed by the normal-world filesystem. The log warns that no replay-protection counter is available. | S:172-173. Relevant to Days 25 and 26. |
