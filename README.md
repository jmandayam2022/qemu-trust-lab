# qemu-trust-lab

A hands-on lab for TF-A, OP-TEE, OpenSSL, PKCS#11, and post-quantum cryptography, run on QEMU from a Mac. Public components and public specs only.

## What you get

A booted TF-A (BL1, BL2, BL31) + OP-TEE (BL32) + U-Boot (BL33) + Linux stack on QEMU `virt`, with the normal-world and secure-world consoles side by side, and `xtest 4002` passing.

Tested versions: OP-TEE 4.11.0 (`qemu_v8.xml`), TF-A 2.14.0, U-Boot 2025.07, Linux 6.18.0, QEMU 10.0.0.

## Tested on

One machine: Apple M3 Max MacBook Pro, Docker Desktop 4.99.0 with a VM of 14 CPUs, 15.6 GiB RAM and about 98 GB disk. Other Apple Silicon Macs should work. Intel Macs are untested. These steps have not been scripted yet, so run the commands by hand.

## Prerequisites

- An Apple Silicon Mac with Docker Desktop installed and running.
- In Docker Desktop, under Settings and Resources: at least 8 GB RAM (more is better) and a disk limit of at least 60 GB (about 100 GB is comfortable).
- About 25 GB of free space for the source and build output.

## Steps

**1. Clone and build the image.** In a Mac terminal:

```sh
git clone https://github.com/jmandayam2022/qemu-trust-lab.git
cd qemu-trust-lab
docker build --platform linux/arm64 -t qemu-trust-lab -f docker/Dockerfile docker/
```

**2. Create the source volume and start the container.** The source lives on a Docker volume, not on the Mac filesystem:

```sh
docker volume create optee-src
docker run -it --name optee-dev --platform linux/arm64 -v optee-src:/optee qemu-trust-lab bash
```

You are now in a shell inside the container, in `/optee`. To come back later, use `docker start -ai optee-dev`.

**3. Fetch the source.** Inside the container (the tag pins the release; if `repo init` asks for a name and email, any values work):

```sh
cd /optee
repo init -u https://github.com/OP-TEE/manifest.git -m qemu_v8.xml -b 4.11.0
repo sync -j8
```

**4. Download the toolchains and build.** The toolchain step only downloads. The build took about 10.5 minutes on the machine above:

```sh
cd /optee/build
make -j3 toolchains
time make -j8 2>&1 | tee /optee/build.log
```

On success, `/optee/out/bin` contains `bl1.bin`, `bl2.bin`, `bl31.bin`, `bl32.bin`, `bl33.bin`, `Image` and the root filesystem images. If memory is tight and you see `Killed`, lower `-j`.

**5. Boot.** The consoles need tmux. Start a session, confirm it is active, and check that the console ports are free:

```sh
tmux set -g history-limit 50000     # optional, keeps the whole boot in scrollback
tmux new -s lab
echo $TMUX                          # must print a path
nc -z 127.0.0.1 54320 && echo busy || echo free
nc -z 127.0.0.1 54321 && echo busy || echo free
cd /optee/build
make run-only
```

Both ports must report `free`. `make run-only` creates a second tmux window (`OPTEE_<n>`) with two panes: normal world (port 54320) and secure world (port 54321). QEMU starts halted at a `(qemu)` prompt.

1. At `(qemu)`, type `c` and press Enter. The guest starts.
2. Switch windows with `Ctrl-b` then `w`. TF-A, U-Boot and Linux messages appear in the normal-world pane. OP-TEE messages appear in the secure-world pane.
3. At `buildroot login:` in the normal-world pane, type `root` (no password).
4. Run the test:

```sh
xtest 4002
```

Expected: `regression_4002 OK` and `3272 subtests of which 0 failed`.

**6. Shut down.** In order: type `quit` at the `(qemu)` prompt, run `pkill -f soc_term.py`, then exit tmux (`tmux kill-session -t lab`) and the container shell. The source and build stay in the `optee-src` volume.

## Common problems

- **`docker version` shows no Server section:** Docker Desktop is not running. Start it and retry.
- **`echo $TMUX` prints nothing:** you are not inside tmux, so no console window will be created. Run `tmux new -s lab` (or `tmux attach -d -t lab`) first.
- **A console port reports `busy`:** a console script from an earlier run is still alive. Run `pkill -f soc_term.py`.
- **Do not start your own `nc -l` listeners for the consoles.** The build's port check connects and disconnects, which makes a one-shot listener exit.
- **Scrolling and copying inside tmux:** `Ctrl-b` then `[` scrolls (`q` exits), and `Ctrl-b` then `z` zooms a pane.

## Layout

- `docker/`: build image
- `scripts/`: build and run scripts (planned)
- `docs/`: build notes, concepts glossary, boot logs and their annotation
- `ta/`: trusted applications (planned)
- `posts/`: write-ups

## Documentation

- `docs/build-notes.md`: what was run and how it differed from the upstream docs.
- `docs/boot-log-annotated.md`: stage-by-stage annotation of a full boot, with raw captures from both consoles.
- `docs/concepts.md`: glossary of the terms used.
- `posts/00-lab-notes.md`: Week 1 write-up.
