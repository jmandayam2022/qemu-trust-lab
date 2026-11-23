# Annotated boot log: TF-A + OP-TEE + U-Boot + Linux on QEMU `virt`

Build: OP-TEE manifest tag 4.11.0, `qemu_v8.xml`. QEMU 10.0.0 (software emulation, `secure=on`). Captured inside tmux with `tmux capture-pane`, then cleaned of the console script's own `soc_term:` lines.

Raw files:

- `boot-log-normal.txt`: normal-world UART (QEMU `-serial` #1, port 54320). 378 lines.
- `boot-log-secure.txt`: secure-world UART (QEMU `-serial` #2, port 54321). 222 lines.

Line references below use `N:` for the normal-world file and `S:` for the secure-world file. Statements marked *(inference)* come from reading the log, not from reading the source. Open questions are collected at the end.

## Key finding: two UARTs, different stages

TF-A (BL1, BL2, BL31) and U-Boot (BL33) and Linux all print on the normal-world UART. OP-TEE (BL32) prints on the secure-world UART. The backlog expected the secure UART to show the whole chain. To see the full boot you need both files.

There are no timestamps on the TF-A, U-Boot, or OP-TEE lines (Linux has them), so the cross-pane order below is inferred from cause and effect.

## Timeline

| Order | Stage | Where | Lines | What it shows |
|---|---|---|---|---|
| 1 | BL1 (TF-A) | normal | N:1-4 | `Booting Trusted Firmware`, `BL1: v2.14.0(release)`, built 23:37:33 Nov 16 2026. First stage, runs at EL3 from the `-bios bl1.bin` image. |
| 2 | BL2 (TF-A) | normal | N:5-11 | `BL1: Booting BL2`, then the BL2 banner. Four `Firmware Image Package header check failed` warnings follow (see questions). |
| 3 | BL1 launches BL31 | normal | N:12-14 | `BL1: Booting BL31`, then the BL31 banner. BL2 does not jump to BL31 itself: it returns to BL1, which transfers control. BL31 stays resident at EL3. |
| 4 | BL32 (OP-TEE) | secure | S:1-140 | BL31 (with `SPD=opteed`) starts OP-TEE at S-EL1. Memory map, DT discovery, version banner at S:54, initcalls, TA store check. Details below. |
| 5 | OP-TEE returns to BL31 | secure | S:141 | `Primary CPU switching to normal world boot`. This is the handoff from BL32 back to BL31 and on to BL33. |
| 6 | BL33 (U-Boot) | normal | N:15-61 | `U-Boot 2025.07` at N:20. Loads the kernel and ramdisk, then `Starting kernel ...` at N:61. |
| 7 | Linux | normal | N:63-364 | Kernel 6.18.0 boots. PSCI, OP-TEE driver, then `Run /init` at N:364. |
| 8 | Userspace | normal | N:365-378 | Buildroot init, `tee-supplicant`, network, login prompt. |

## BL1, BL2, BL31 (TF-A 2.14.0)

- N:1-3: banner and build time. The build time matches the Day 4 build. Release build, so only `NOTICE` and above are printed.
- N:4, N:8-11: `WARNING: Firmware Image Package header check failed.` Boot continues, so this does not stop the boot. See questions.
- N:12: `BL1: Booting BL31`. In the classic TF-A flow, BL2 loads BL31 and BL32, then asks BL1 to run BL31.
- No TF-A line announces BL32 or BL33 at this log level. Their start is visible only through their own output (S:1 and N:20).

## BL32: OP-TEE 4.11.0 (secure UART)

- S:1-31: memory map from `collect_mem_ranges` and `add_phys_mem`. Key values:
  - Core TEE RAM at `0x0e100000` (RX part size `0xb6000`, RW part from `0x0e1b6000`).
  - Secure RAM overall `0x0e1b6000` to `0x0effffff`.
  - GIC at `0x08000000`, console UART region at `0x09000000`.
  - Non-secure shared memory `TEE_SHMEM_START` at `0x42000000`, size `0x200000`.
- S:22: `Mapping core at 0x60f6a000 offs 0x52e6a000`: core ASLR is active (Day 13 plans `CFG_CORE_ASLR=n`).
- S:47: `Non-secure external DT found`, S:52: `Embedded DTB found`. OP-TEE reads a DT passed in from firmware (BL31 or earlier).
- S:54: `OP-TEE version: 4.11.0 (gcc 14.3.1 ... Arm GNU Toolchain 14.3.Rel1) #1 Mon Nov 16 23:37:29 UTC 2026 aarch64`. Matches the pinned tag and the Day 3 toolchain.
- S:55-56: `WARNING: This OP-TEE configuration might be insecure!` Expected for a development build (see questions).
- S:57-58: `Primary CPU initializing`, then execution at offset `0x52e6a000`.
- S:61-140: `do_init_calls` runs initcalls in levels (preinit, early, service, driver, final). Includes DT driver probing of the `dt-test-*` nodes, which are OP-TEE's built-in DT driver tests (S:68-104, the deferred and probe-list lines), `tee_cryp_init`, `notif_init`, `tee_fs_init_key_manager`, the GIC and console interrupt setup, and boot-time self tests (`ftmn_boot_tests`, S:126-135).
- S:107-110: `check_ta_store` lists three TA stores in this order: `early TA`, `Secure Storage TA`, `REE`. This is the same order the loader uses later when it looks for a TA.
- S:111-113: `early_ta_init` finds two TAs built into the OP-TEE image: `bc50d971-d4c9-42c4-82cb-343fb7f37896` and `f04a0fe7-1f5d-4b9b-abf7-619b85b4ce8c`. See questions.
- S:141: `Primary CPU switching to normal world boot`.
- S:144-145: `Secondary CPU 1 initializing`, then switching to normal world. This happens after Linux starts the second core (N:149-153, via PSCI *(inference)*).
- S:147-150: `Reserved shared memory`, `Dynamic shared memory`, `Normal World virtualization support is disabled`, `Asynchronous notifications are enabled`. These appear after the kernel OP-TEE driver probes (N:343-347) *(inference)*.
- S:153-157: `device.pta` (UUID `7011a688-ddde-4053-a5a9-7b3c4ddf13b8`) is opened and closed.
- S:158-171: the two early TAs are loaded through `ldelf` (the user-mode ELF loader). Look at the lookup lines: `Lookup user TA ELF ... (early TA)` returns `res=0`, so these are found without touching the normal world.
- S:172-173: `WARNING (insecure configuration): Failed to get monotonic counter for REE FS, using 0` and `Failed to commit dirh counter 2`. REE FS storage has no replay protection in this setup (relevant to Days 25 and 26).
- S:174-217: `periodic_callback` and `test_wd_callback` lines come from a periodic callout and a watchdog self-test. They stop at S:218 (`Disabling periodic callout`).
- S:201-207: `system.pta` (UUID `3a2f8978-5dc0-11e8-9c2d-fa7ae01bbebc`) is opened, followed by `tee_ta_invoke_command:839 Error: ffff000a of 4`. See questions.

## BL33: U-Boot 2025.07 (normal UART)

- N:15-17: `Bloblist at 0 not found (err=-2)` twice, with `alloc space exhausted ptr 400 limit 0` between them. These appear before the U-Boot banner, so they come from U-Boot's very early init *(inference)*.
- N:20: `U-Boot 2025.07 (Nov 16 2026 - 23:36:02 +0000)`. BL33 is U-Boot, as `BL33_BIN ?= $(UBOOT_BIN)` in `qemu_v8.mk` predicted. This answers the Day 5 question about `-kernel Image`: the guest does run U-Boot.
- N:22-32: DRAM 1 GiB, 32 MiB flash, environment `bad CRC, using default environment`, serial console, no USB controllers, network `virtio-net`.
- N:34: `Hit any key to stop autoboot:  0`. The autoboot delay is 0, so there is no pause.
- N:35-36: two reads, 41413184 bytes (the size of `uImage`) and 13839481 bytes (the size of `rootfs.cpio.uboot`). Both match the file sizes in `/optee/out/bin`. How U-Boot reads them is an open question.
- N:37-52: `Booting kernel from Legacy Image at 42200000`, kernel entry `0x42200000`, ramdisk legacy image at `0x46000000` with load address `0x45000000`, both checksums OK.
- N:53-59: device tree blob at `0x40000000`, relocated for the kernel (ramdisk to `0x7ea25000`, DT to `0x7ea1f000`).
- N:61: `Starting kernel ...`.

## Linux 6.18.0 and the handoff to userspace

- N:64: built in the container (`root@d4b3f96ed57c`) with `aarch64-linux-gnu-gcc` from Arm GNU Toolchain 14.3.Rel1.
- N:67: `Machine model: linux,dummy-virt`. The QEMU `virt` device tree.
- N:69-70: reserved memory `optee_core@e100000` (about 15 MiB, nomap) and `optee_shm@42000000` (2 MiB, nomap). These match the OP-TEE addresses in S:1-31: core at `0x0e100000`, shared memory at `0x42000000`.
- N:85-89: PSCI v1.1 detected in firmware, `Trusted OS migration not required`, SMC calling convention v1.5. This is BL31 answering PSCI calls.
- N:92-190: CPU features. Many extensions are present because QEMU runs with `-cpu max`.
- N:98: kernel command line `console=ttyAMA0,115200 earlyprintk=serial,ttyAMA0,115200 root=/dev/ram`. This is not the QEMU `-append` string (`console=ttyAMA0,38400 keep_bootcon root=/dev/vda2`). See questions.
- N:156: `CPU: All CPU(s) started at EL1`. No EL2 is available (`virtualization=false`), which also explains N:266 `kvm: HYP mode not available`.
- N:220: PL011 UART at `0x9000000` is the kernel console (`ttyAMA0`).
- N:264: `ARM FF-A: FFA_VERSION returned not supported`. Expected with `SPD=opteed`, which does not provide FF-A.
- N:282: PL061 GPIO registered at `0x9030000`. The earlier `reboot` test printed `QEMU System Reset: with GPIO`, so this GPIO is probably what resets the machine *(inference)*.
- N:343-347: `optee: probing for conduit method`, `revision 4.11 (f54190f22611dd5a)`, asynchronous notifications enabled, dynamic shared memory enabled, `initialized driver`. The normal world has asked OP-TEE for its version and capabilities over SMC.
- N:364: `Run /init as init process`.

## Userspace (Buildroot)

- N:365-375: init scripts. `Set permissions on /dev/tee*`, `Create/set permissions on /var/lib/tee`, and `Starting tee-supplicant: Using device /dev/teepriv0`. `tee-supplicant` is what serves OP-TEE's requests for the normal world, such as loading TAs from `/lib/optee_armtz` (the REE store seen on Day 5).
- N:377-378: login prompt.

## Cross-check: addresses that appear in both logs

| Item | Secure log | Normal log |
|---|---|---|
| OP-TEE core RAM | `0x0e100000` (S:2) | `optee_core@e100000` (N:69) |
| Shared memory | `0x42000000`, size `0x200000` (S:18) | `optee_shm@42000000` (N:70) |
| Kernel entry | none | `0x42200000` (N:42-43) |
| Non-secure RAM seen by OP-TEE | `0x40000000..0x41ffffff` and `0x42200000..0x820fffff` (S:50-51) | Early memory node ranges (N:79-81) |

## Open questions

1. `Firmware Image Package header check failed` (N:4, N:8-11): where does TF-A look for a FIP here, and how are the images actually loaded? The QEMU command line has semihosting enabled. Check `plat/qemu` in TF-A.
2. `Bloblist at 0 not found` and `alloc space exhausted ptr 400 limit 0` (N:15-17): U-Boot early init messages. Harmless here? Check U-Boot bloblist handling.
3. How does U-Boot read `uImage` and `rootfs.cpio.uboot` (N:35-36)? Semihosting, flash, or something else? Check the default environment and boot command.
4. The QEMU command line passes `-kernel Image`, `-initrd rootfs.cpio.gz` and `-append '...'`, but the kernel booted with a different command line (N:98). Which of these options does QEMU use, and which are ignored?
5. Who creates the reserved-memory nodes for `optee_core` and `optee_shm` in the device tree the kernel sees (N:69-70)? OP-TEE, TF-A, or U-Boot?
6. What are the early TAs `bc50d971-d4c9-42c4-82cb-343fb7f37896` and `f04a0fe7-1f5d-4b9b-abf7-619b85b4ce8c` (S:112-113), and why are they loaded right after Linux boots (S:158-171)? Possibly the kernel enumerating devices on the TEE bus through `device.pta` *(inference)*.
7. `tee_ta_invoke_command:839 Error: ffff000a of 4` (S:205), right after `system.pta` is opened. Which TA asked for what, and what does `0xffff000a` mean?
8. Which config options trigger `This OP-TEE configuration might be insecure!` (S:55)? The debug lines (`D/TC`) show the core log level is at least 3 (levels are defined in `optee_os/lib/libutils/ext/include/trace_levels.h`: 0 min, 1 error, 2 info, 3 debug, 4 flow). Look up the current value of `CFG_TEE_CORE_LOG_LEVEL` in the build.
9. Confirm that `reboot` goes through the PL061 GPIO to a full machine reset (N:282, plus the earlier `QEMU System Reset: with GPIO` line).
10. Which UART base does OP-TEE use for its console, and which does TF-A use? The log shows TF-A on the first UART and OP-TEE on the second, but the build constants have not been read (for OP-TEE, start with `core/arch/arm/plat-vexpress/platform_config.h`).
