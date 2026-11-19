# Build notes

Record every command, path, or flag that differed from the docs, with the fix.

## Day 4: first full build (OP-TEE 4.11.0, qemu_v8.xml)

- Host: MacBook Pro, Apple M3 Max; Docker Desktop VM with 14 CPUs, 15.63 GiB RAM, ~98 GB disk; linux/arm64 container (Ubuntu 22.04).
- Command (from `/optee/build`): `time make -j8 2>&1 | tee /optee/build-day4.log`
- Result: clean build on the first attempt. real 10m34s, user 80m41s, sys 8m58s.
- `make` (no target) uses `all`, so QEMU is not launched. `build/Makefile` is expected to link to `qemu_v8.mk` via the manifest.
- Tree size after build: 19 GB (3.1 GB after `repo sync`).
- Boot images are in `/optee/out/bin` (mostly symlinks): bl1, bl2, bl31, bl32 (header + pager + pageable), bl33 (U-Boot), Image, rootfs.cpio.gz.

## Day 3: what `make toolchains` does (OP-TEE 4.11.0)

- `toolchains:` depends on `aarch32-toolchain`, `aarch64-toolchain` and `rust-toolchain` (`build/toolchain.mk`).
- Nothing is compiled on this path. The `dltc` macro downloads the Arm GNU toolchain 14.3.rel1 archives with `curl` and unpacks them with `tar xf ... --strip-components=1` into `/optee/toolchains/aarch32` and `/optee/toolchains/aarch64`.
- `dl-rust-toolchain` runs the standard rustup installer (`curl ... https://sh.rustup.rs | sh -s -- -y --no-modify-path`) with `RUSTUP_HOME` and `CARGO_HOME` under `/optee/toolchains/rust`. It installs prebuilt binaries.
- A separate `build_toolchain` macro unpacks a Buildroot-built SDK (this one does compile through Buildroot). It belongs to the alternative variant that uses `.done` markers. It was not used here, inferred from the Arm archives being present.
- Observation, to confirm before acting on it: `dltc` uses `curl -k`, which disables TLS certificate verification, and the lines read so far show no checksum comparison, only a check that `tar` can unpack the file. Review `sed -n '15,30p' build/toolchain.mk` and current upstream before raising it (possible Day 33 topic).

## Day 5: first boot

- Command (inside tmux, from `/optee/build`): `make run-only`. At the `(qemu)` prompt type `c`. Log in as `root` on the normal-world console.
- The console helper (`launch-terminal` in `build/common.mk`) uses tmux when the `TMUX` environment variable is set: it creates a window `OPTEE_<random>` with one `soc_term.py` pane per UART port (54320 normal world, 54321 secure world), skipping any port that already accepts connections (`nc -z`). So the build works in a headless container as long as it runs inside tmux.
- Pitfall: starting your own `nc -l` listeners first fails. The helper probes each port with `nc -z`, which connects and disconnects, and a one-shot OpenBSD `nc -l` exits after that connection. The helper then sees the port as taken and creates no window.
- Result: `xtest 4002` printed `regression_4002 OK` (3272 subtests, 0 failed). QEMU 10.0.0.
