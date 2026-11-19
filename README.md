# qemu-trust-lab

A hands-on lab for TF-A, OP-TEE, OpenSSL, PKCS#11, and post-quantum cryptography, run on QEMU from a Mac.

Public components and public specs only.

## Status

Phase 1, Week 1. Build image first, then source sync, then full build.

## Quick start (Day 1)

```sh
docker build --platform linux/arm64 -t qemu-trust-lab -f docker/Dockerfile docker/
docker run --rm -it --platform linux/arm64 qemu-trust-lab bash
# inside: repo --version && make --version
```

## Layout

- `docker/`: build image
- `scripts/`: `build.sh`, `run.sh`, `attach-consoles.sh` (later)
- `docs/`: build notes, annotated boot logs
- `ta/`: trusted applications
- `posts/`: write-ups
