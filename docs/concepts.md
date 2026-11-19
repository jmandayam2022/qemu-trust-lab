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
