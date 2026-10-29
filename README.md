# OpenMPI 6 Binary Builder

An automated, resource-efficient CI pipeline designed to pre-compile OpenMPI 6 Release Candidate (RC) binaries on Linux x86_64 and publish them as persistent GitHub Release assets. This eliminates redundant 15–20 minute source builds across downstream workflows, maximizing developer productivity and CI compute efficiency.

## Overview

- **Storage & Distribution**: Binaries and SHA-256 checksums are stored as GitHub Release assets (`openmpi-<version>-linux-x86_64.tar.gz`), eliminating ephemeral artifact retention limits.
- **Resource Optimization**: Features an idempotency skip-guard that inspects existing releases before compiling, preventing unnecessary compute usage when assets already exist.
- **Portable Standard Profile**: Built with `--prefix=/opt/openmpi`, internal PMIx/PRRTE runtimes, C/C++/Fortran bindings, and shared memory/TCP transports on standard Ubuntu LTS runners.

## Workflows

1. **`build-openmpi.yml`**: Compiles, packages, checksums, and publishes OpenMPI release assets. Triggered via `workflow_dispatch` (with version/custom URL inputs) or version tags (e.g. `v6.0.0rc1`).
2. **`verify-openmpi.yml`**: Downstream consumption reference workflow that downloads the release asset, verifies cryptographic checksums, configures runtime paths, and validates multi-process C and Fortran MPI execution.

## Downstream Usage

To consume pre-built OpenMPI binaries in downstream GitHub Actions workflows:

```yaml
- name: Install Pre-built OpenMPI 6
  run: |
    VERSION="6.0.0rc1"
    TAG="v${VERSION}"
    ASSET="openmpi-${VERSION}-linux-x86_64.tar.gz"
    CHECKSUM="${ASSET}.sha256"
    REPO="<owner>/<repo>"

    # Download release asset and checksum
    curl -fSL "https://github.com/${REPO}/releases/download/${TAG}/${ASSET}" -o "${ASSET}"
    curl -fSL "https://github.com/${REPO}/releases/download/${TAG}/${CHECKSUM}" -o "${CHECKSUM}"

    # Verify integrity and unpack
    sha256sum -c "${CHECKSUM}"
    sudo mkdir -p /opt/openmpi
    sudo tar -xzf "${ASSET}" -C /opt

    # Configure environment
    echo "/opt/openmpi/bin" >> $GITHUB_PATH
    echo "LD_LIBRARY_PATH=/opt/openmpi/lib:${LD_LIBRARY_PATH:-}" >> $GITHUB_ENV
    echo "OPAL_PREFIX=/opt/openmpi" >> $GITHUB_ENV
```

## Triggering a Build

### Via GitHub CLI
```bash
gh workflow run build-openmpi.yml -f version="6.0.0rc1"
```

### Via Git Tag
```bash
git tag v6.0.0rc1
git push origin v6.0.0rc1
```
