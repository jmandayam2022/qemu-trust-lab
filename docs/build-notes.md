# Build notes

Record every command, path, or flag that differed from the docs, with the fix.

## Day 4: first full build (OP-TEE 4.11.0, qemu_v8.xml)

- Host: MacBook Pro, Apple M3 Max; Docker Desktop VM with 14 CPUs, 15.63 GiB RAM, ~98 GB disk; linux/arm64 container (Ubuntu 22.04).
- Command (from `/optee/build`): `time make -j8 2>&1 | tee /optee/build-day4.log`
- Result: clean build on the first attempt. real 10m34s, user 80m41s, sys 8m58s.
- `make` (no target) uses `all`, so QEMU is not launched. `build/Makefile` is expected to link to `qemu_v8.mk` via the manifest.
- Tree size after build: 19 GB (3.1 GB after `repo sync`).
- Boot images are in `/optee/out/bin` (mostly symlinks): bl1, bl2, bl31, bl32 (header + pager + pageable), bl33 (U-Boot), Image, rootfs.cpio.gz.
