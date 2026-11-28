# OP-TEE + TF-A on QEMU from a Mac: Week 1 lab notes

*Draft. Everything here was run on one machine (an Apple M3 Max MacBook Pro), so treat the timings as a data point, not a benchmark. All components are public: OP-TEE 4.11.0, TF-A 2.14.0, U-Boot 2025.07, Linux 6.18.0, QEMU 10.0.0.*

This is a record of getting a full secure boot chain (TF-A, OP-TEE, U-Boot, Linux) running on QEMU from a Mac, including what surprised me along the way. The goal of the larger project is a hands-on lab for TrustZone software, so the first job was a boot that anyone can reproduce.

## What are the moving parts?

| Component | Version | Role in the boot |
|---|---|---|
| TF-A | 2.14.0 | BL1, BL2 and BL31 (the EL3 runtime) |
| OP-TEE OS | 4.11.0 | BL32, the secure OS at S-EL1 |
| U-Boot | 2025.07 | BL33, the normal-world bootloader |
| Linux | 6.18.0 | Normal-world OS, with the OP-TEE driver |
| QEMU | 10.0.0 | Emulates an Armv8-A `virt` machine with `secure=on` |
| Build host | Ubuntu 22.04, linux/arm64 | A container, not the Mac itself |

The OP-TEE manifest (`qemu_v8.xml`, pinned to the 4.11.0 tag) fetches all of them, and one `make` builds them.

## Why does the build run in a container?

OP-TEE's build expects Linux, a long list of packages, and a case-sensitive filesystem. While macOS can run some of these tools, the combination is fragile, and the source trees contain filenames that differ only by case.

On a Mac, Docker Desktop runs a small hidden Linux VM, and containers are processes inside that VM. Two consequences shaped the setup:

- The VM has its own CPU, memory and disk limits. I gave it 14 CPUs, 15.6 GiB of RAM and roughly 98 GB of disk. The default memory (7.75 GiB) looked tight for the kernel and QEMU builds, so I raised it before starting.
- The source tree lives on a named Docker volume (`optee-src`, mounted at `/optee`), not a macOS bind mount. The volume sits on the VM's Linux filesystem, so it is fast and case-sensitive, and it survives deleting the container or rebuilding the image.

The Dockerfile starts from the Ubuntu 22.04 package list on the OP-TEE prerequisites page. I removed the `repo sync` and toolchain steps from the image, because those belong on the volume, and added `tmux` for the consoles later.

QEMU runs inside the same container. Because it emulates the CPU in software (TCG), it can provide EL3 and the secure world, which Apple's hypervisor does not expose to guests.

*Implication: the container is the build host, and the QEMU guest is the board.*

## What did the first full build cost?

```
repo sync -j8          3.1 GB on disk afterwards
make -j3 toolchains    downloads only, nothing compiled
time make -j8          real 10m34s, user 80m41s, sys 8m58s
                       19 GB on disk afterwards
```

The build was clean on the first attempt. A search of the log for make failures found only three `(ignored)` errors, from a documentation step of e2fsprogs in Buildroot, which do not matter for the lab. I used `-j8` and not `-j$(nproc)` (14 here) to leave memory headroom, and saw no out-of-memory kills.

`make toolchains` surprised me. I expected compiler builds. In `build/toolchain.mk` it downloads prebuilt Arm GNU Toolchain 14.3.rel1 archives for aarch64 and aarch32 with `curl`, unpacks them with `tar`, and installs a Rust toolchain through the standard rustup installer. Only the full `make` that follows does real compiling.

*Implication: the first build took ten minutes of wall time on a 14-core VM, so a fresh rebuild is cheap enough to test reproducibility.*

## Why do the consoles only appear inside tmux?

`make run-only` starts QEMU with two serial ports attached to TCP sockets:

```
-serial tcp:127.0.0.1:54320 -serial tcp:127.0.0.1:54321
```

QEMU is the TCP client, so something must already listen on each port. The makefile helper that does this (`launch-terminal` in `build/common.mk`) picks a method by checking the environment. If the `TMUX` variable is set, it creates a tmux window named `OPTEE_<n>` with one pane per port, each running a small Python script (`soc_term.py`) that listens and shows the UART. Otherwise it tries `gnome-terminal`, `konsole` or `xterm`, none of which exist in the container.

So the fix for a headless container is to run `make run-only` inside tmux.

My first attempt was different, and it failed in an instructive way. I started my own `nc -l 54320` and `nc -l 54321` listeners first. They exited as soon as the helper ran. The helper checks each port with `nc -z`, which connects and hangs up, and a one-shot `nc -l` quits when its connection closes. That is consistent with the first lines in the captured console log, where `soc_term` reports an accepted connection that closed at once (`read fd EOF`) before QEMU's real connection arrived. The helper also skips creating a window for any port that already accepts connections, so no console appeared at all.

*Implication: start tmux first, check that both ports are free, and do not start your own listeners.*

## Which UART shows which boot stage?

I expected the secure-world UART to show the whole chain. It does not. This is the start of the normal-world console:

```
NOTICE:  Booting Trusted Firmware
NOTICE:  BL1: v2.14.0(release):v2.14.0
NOTICE:  BL1: Built : 23:37:33, Nov 16 2026
WARNING: Firmware Image Package header check failed.
NOTICE:  BL1: Booting BL2
NOTICE:  BL2: v2.14.0(release):v2.14.0
...
NOTICE:  BL1: Booting BL31
NOTICE:  BL31: v2.14.0(release):v2.14.0
...
U-Boot 2025.07 (Nov 16 2026 - 23:36:02 +0000)
```

TF-A, U-Boot and Linux all print on the first UART. OP-TEE prints on the second:

```
I/TC: OP-TEE version: 4.11.0 (gcc version 14.3.1 20250623 (Arm GNU Toolchain 14.3.Rel1 ...)) aarch64
I/TC: WARNING: This OP-TEE configuration might be insecure!
I/TC: Primary CPU initializing
...
I/TC: Primary CPU switching to normal world boot
```

Reading the two logs together gives the order: BL1, then BL2, then BL1 launches BL31 (BL2 returns to BL1, which transfers control), BL31 starts OP-TEE, OP-TEE reports `switching to normal world boot`, and BL31 hands over to U-Boot, which loads the kernel and ramdisk. Linux then reports the secure OS from its own side:

```
optee: probing for conduit method.
optee: revision 4.11 (f54190f22611dd5a)
optee: Asynchronous notifications enabled
optee: dynamic shared memory is enabled
optee: initialized driver
```

Two details stand out. First, the QEMU command line passes `-kernel Image` and `-append '...'`, but the kernel booted with a different command line (`console=ttyAMA0,115200 ... root=/dev/ram`), so U-Boot, not QEMU, is what actually starts it. Second, the reserved-memory nodes in the kernel's device tree (`optee_core@e100000`, `optee_shm@42000000`) match the addresses in OP-TEE's own memory map (`0x0e100000` and `0x42000000`), which is a useful cross-check that both sides agree on the layout.

*Implication: capture both consoles when documenting a boot, and annotate them together. The full annotation is in `docs/boot-log-annotated.md`.*

## What does a passing test look like?

After the login prompt, `root` logs in without a password. Then:

```
# xtest 4002
Result of testsuite regression+pkcs11 filtered by "4002":
regression_4002 OK
3272 subtests of which 0 failed
1 test case of which 0 failed
148 test cases were skipped
```

The secure console shows what OP-TEE did for the call. OP-TEE looked for the test TA (the `crypt` TA from `optee_test`) in three stores, in order: the early TAs built into the OP-TEE image, the Secure Storage TA, and finally the normal-world filesystem (REE), where `tee-supplicant` found it. The first two lookups returned `res=0xffff0008`, which I read as item not found, and the third returned `res=0`. The TA was then loaded by `ldelf`, run, and destroyed.

I also tried `reboot` in the guest. While I expected only the normal world to restart, the whole machine reset from BL1, and the TF-A lines appeared again. The log shows `QEMU System Reset: with GPIO`, so the reset goes through a GPIO device rather than staying in Linux.

*Implication: one command exercises the full path from a normal-world client through BL31 to a TA in the secure world.*

## What is still unclear?

Open questions from reading the logs, none of them blocking:

1. `Firmware Image Package header check failed` appears five times in TF-A's output, and the boot continues. Where does TF-A look for a FIP here, and how are the images actually loaded?
2. How does U-Boot read the kernel and ramdisk (the log shows two reads that match the sizes of `uImage` and `rootfs.cpio.uboot`)? Which of QEMU's `-kernel`, `-initrd` and `-append` options are used at all?
3. Who creates the OP-TEE reserved-memory nodes in the device tree the kernel sees: OP-TEE, TF-A or U-Boot?
4. Which build options produce `This OP-TEE configuration might be insecure!`, and which setting gives the debug-level (`D/TC`) core log?
5. What are the two early TAs (`bc50d971-...` and `f04a0fe7-...`) that load right after Linux boots, and what is behind the `Error: ffff000a` returned for a `system.pta` command?

The next steps are scripting the build and run (so that a fresh clone needs two commands), then moving to a first trusted application. The reproducible steps for everything above are in the repository README.
