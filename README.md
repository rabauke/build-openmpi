# OpenMPI 6 and MPICH Binary Builders

An automated, resource-efficient CI pipeline designed to pre-compile OpenMPI 6 Release Candidate (RC) binaries on Linux x86_64 and publish them as persistent GitHub Release assets. This eliminates redundant 15–20 minute source builds across downstream workflows, maximizing developer productivity and CI compute efficiency.

The repository also provides a manually triggered MPICH build pipeline and a standalone downstream verifier for Linux x86_64.

## Overview

- **Storage & Distribution**: Binaries and SHA-256 checksums are stored as GitHub Release assets (`openmpi-<version>-linux-x86_64.tar.gz`), eliminating ephemeral artifact retention limits.
- **Resource Optimization**: Features an idempotency skip-guard that inspects existing releases before compiling, preventing unnecessary compute usage when assets already exist.
- **Portable Standard Profile**: Built with `--prefix=/opt/openmpi`, internal PMIx/PRRTE runtimes, C/C++/Fortran bindings, and shared memory/TCP transports on standard Ubuntu LTS runners.

## Workflows

1. **`build-openmpi.yml`**: Compiles, packages, checksums, and publishes OpenMPI release assets. Triggered via `workflow_dispatch` (with version/custom URL inputs) or version tags (e.g. `v6.0.0rc1`).
2. **`verify-openmpi.yml`**: Downstream consumption reference workflow that downloads the release asset, verifies cryptographic checksums, configures runtime paths, and validates multi-process C and Fortran MPI execution.
3. **`build-mpich.yml`**: Manually triggered only (`workflow_dispatch`); accepts an MPICH version, optional source URL override, and force-rebuild option. It publishes a versioned archive and SHA-256 checksum, then dispatches the MPICH verifier after a successful build or valid asset skip.
4. **`verify-mpich.yml`**: Standalone downstream-consumption verifier that can be started manually or automatically by the MPICH builder. It downloads the selected release assets, checks the SHA-256 checksum, installs MPICH, and runs C and Fortran MPI smoke tests.

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

MPICH release assets use the names `mpich-<version>-linux-x86_64.tar.gz` and `mpich-<version>-linux-x86_64.tar.gz.sha256`, attached to the `v<version>` release. For example, this downstream GitHub Actions step downloads and installs MPICH 4.2.3:

```yaml
- name: Install Pre-built MPICH
  run: |
    VERSION="4.2.3"
    TAG="v${VERSION}"
    ASSET="mpich-${VERSION}-linux-x86_64.tar.gz"
    CHECKSUM="${ASSET}.sha256"
    REPO="<owner>/<repo>"

    sudo apt-get update
    sudo apt-get install -y --no-install-recommends build-essential gfortran libfabric-dev libhwloc-dev

    curl -fSL "https://github.com/${REPO}/releases/download/${TAG}/${ASSET}" -o "${ASSET}"
    curl -fSL "https://github.com/${REPO}/releases/download/${TAG}/${CHECKSUM}" -o "${CHECKSUM}"
    sha256sum -c "${CHECKSUM}"
    sudo tar -xzf "${ASSET}" -C /opt

    echo "/opt/mpich/bin" >> "$GITHUB_PATH"
    echo "LD_LIBRARY_PATH=/opt/mpich/lib:${LD_LIBRARY_PATH:-}" >> "$GITHUB_ENV"
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

### MPICH (manual dispatch only)

The MPICH builder has no tag-push trigger. Start it from the GitHub Actions UI or with GitHub CLI; it defaults to version `4.2.3` and skips rebuilding when both MPICH release assets already exist unless `force_rebuild` is enabled.

```bash
gh workflow run build-mpich.yml -f version="4.2.3"
```

To override the source tarball URL or force a rebuild:

```bash
gh workflow run build-mpich.yml \
  -f version="4.2.3" \
  -f download_url="https://example.com/mpich-4.2.3.tar.gz" \
  -f force_rebuild=true
```

The build automatically starts `verify-mpich.yml` after a successful build or a valid asset skip. To run the verifier independently, specify the version and, when checking another repository, its `owner/repo` name:

```bash
gh workflow run verify-mpich.yml -f version="4.2.3" -f repository="<owner>/<repo>"
```
