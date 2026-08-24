# Parameterized GNU Radio Docker Images for CI

## Overview

This update introduces a new **parameterized Dockerfile** for building GNU Radio images from different Git references. The goal is to make the CI workflow more flexible by allowing a specific branch, tag, or commit to be selected at build time.

## What Was Added?

A new `Dockerfile.pr` was added. It is based on **Debian Bookworm** and is configured specifically for use with CortexLab.

Key features include:

- Sets up an SSH server on port `2222` with root access.
- Installs the CortexLab build toolchain and required dependencies.
- Installs **UHD**, **VOLK**, and the required Python dependencies.
- Builds additional GNU Radio modules:
  - `gr-bokehgui`
  - `gr-iqbal`
- Configures the image for execution in CortexLab environments.

## Parameterized GNU Radio Reference

The main feature is the configurable GNU Radio Git reference:

```dockerfile
ARG GNURADIO_REF=maint-3.10
```

Instead of hardcoding a single GNU Radio branch or version, the Docker build can now receive the desired Git reference using `--build-arg`.

For example:

```bash
docker build \
  -f Dockerfile.pr \
  --build-arg GNURADIO_REF=maint-3.10 \
  -t gnuradio-test .
```

A specific branch, tag, or commit can also be used:

```bash
--build-arg GNURADIO_REF=<branch>
```

```bash
--build-arg GNURADIO_REF=<tag>
```

```bash
--build-arg GNURADIO_REF=<commit-sha>
```

## Why Is This Important?

Previously, testing different GNU Radio versions could require separate or modified Dockerfiles. With this approach, the same Dockerfile can build different GNU Radio revisions.

This is particularly useful for the **CI-based Hardware-in-the-Loop testing workflow**. A pull request can provide a specific GNU Radio Git reference, and the CI pipeline can build an image containing exactly that revision before running the hardware tests on CortexLab.

The workflow becomes:

```text
GNU Radio PR
     │
     ▼
Get Git reference / commit SHA
     │
     ▼
Build Docker image
(--build-arg GNURADIO_REF=...)
     │
     ▼
Push image
     │
     ▼
Run CortexLab HIL tests
     │
     ▼
Report test result
```

## Impact

This change makes the Docker-based CI infrastructure **more reusable and flexible**.

The same Dockerfile can now be used to build and test:

- Different GNU Radio branches
- Release tags
- Specific commits
- Pull request revisions

Most importantly, it removes the need to maintain multiple hardcoded Dockerfiles for different GNU Radio versions and provides a foundation for automatically testing the **exact GNU Radio revision associated with a pull request**.

## Next Step

The next step is to integrate this parameterized Docker build into the production CI workflow so that the workflow automatically builds the requested GNU Radio revision, publishes the resulting image, passes the image reference to the CortexLab scenario, and reports the Hardware-in-the-Loop test result back to the originating pull request.