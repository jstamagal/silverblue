# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository builds **Universal Blue "Main"** - a set of enhanced Fedora Atomic Desktop container images that serve as the foundation layer for downstream Universal Blue projects (Aurora, Bazzite, Bluefin). The images are bootable OCI containers based on Fedora Silverblue (GNOME), Kinoite (KDE), and Base-Atomic, with "batteries included" modifications.

**Key Philosophy:** Build secure, reproducible, signed container images that users can rebase their systems to using `rpm-ostree` or `bootc`.

**Note:** This is a personal fork focused on Silverblue (GNOME) only. Kinoite (KDE) support has been removed. Fedora 42 support has been removed in favor of Rawhide (development version).

## Common Commands

All development commands use the `just` command runner (Justfile).

### Building Images

```bash
# Build default image (silverblue, latest version, main variant)
just build

# Build specific image/version/variant
just build silverblue 43 main
just build base rawhide nvidia
just run silverblue rawhide main
```

### Verification and Security

```bash
# Verify container signature
just verify-container <container> <registry> <key>

# Check kernel secureboot signatures
just secureboot silverblue 43 main
```

### Linting and Formatting

```bash
# Check code formatting (uses treefmt)
just check

# Auto-fix formatting issues
just fix
```

### Registry Operations

```bash
# Generate image tags (with timestamp)
just gen-tags silverblue 42 main

# Push to registry
just push-to-registry silverblue 42 main

# Sign image with cosign
just cosign-sign silverblue 42 main
```

### Testing Individual Build Steps

The Containerfile uses a multi-stage build with mounts. To test individual scripts:

```bash
# The build scripts are in build_files/ and run in this order:
# 1. install.sh - Core system setup, DNF5, COPR repos, kernel replacement
# 2. packages.sh - Install/remove packages from packages.json
# 3. nvidia-install.sh - NVIDIA drivers (if BUILD_NVIDIA=Y)
# 4. initramfs.sh - Generate initramfs with dracut
# 5. post-install.sh - Service enablement and cleanup
```

## Architecture

### Multi-Stage Container Build

The build process uses a 4-stage container build pattern:

1. **ctx** stage: Copies build context (`sys_files/` and `build_files/`)
2. **akmods** stage: Pulls signed kernel modules from `ublue-os/akmods`
3. **akmods_nvidia** stage: Pulls NVIDIA drivers from `ublue-os/akmods-nvidia-open`
4. **final** stage: Builds the actual image with all components mounted

All stages are mounted during the final build to avoid inflating image size.

### Image Variants

- **main**: Standard build with open-source drivers
- **nvidia**: Includes NVIDIA proprietary drivers via akmods

### Supported Images

- **base**: Minimal base-atomic with essential tools
- **silverblue**: GNOME desktop environment

### Supported Fedora Versions

- **43** (Latest): Current stable release
- **rawhide**: Development version (pulls latest, no digest pinning)

### Configuration-Driven Package Management

`packages.json` uses hierarchical structure:

```json
{
  "all": {
    "include": {
      "all": [...],           // All images
      "silverblue": [...]     // GNOME-specific
    },
    "exclude": {...}
  }
}
```

Packages are installed/removed by `build_files/packages.sh` using `jq` to parse this manifest.

### Dependency Pinning and Security

- **`image-versions.yaml`**: Contains SHA256 digests for all base images and akmods
- **Cosign verification**: All upstream images are cryptographically verified
- **Signed kernels**: Kernels from ublue-os/akmods are signed for secure boot
- **Kernel versionlock**: Prevents automatic kernel updates to maintain driver compatibility

### Build Process Flow

```
Pull & Verify Base Images (Fedora Atomic Desktop)
    ↓
Pull & Verify Signed Kernels (ublue-os/akmods)
    ↓
Copy System Files (sys_files/ → /)
    ↓
Install DNF5 & Setup COPR repos (ublue-os/packages, ublue-os/staging)
    ↓
Replace Kernel with Signed Version + Versionlock
    ↓
Override Mesa/Multimedia (negativo17 repo - patent-unrestricted)
    ↓
Install/Remove Packages (packages.json)
    ↓
Install NVIDIA Drivers (nvidia variant only)
    ↓
Generate Initramfs (dracut)
    ↓
Enable Update Services (rpm-ostreed, flatpak updates)
    ↓
Validate with bootc container lint
```

### CI/CD Architecture

Uses GitHub Actions with a reusable workflow pattern (`.github/workflows/reusable-build.yml`):

- **Smart Build Detection**: Only builds when dependencies change (base image digests, akmods digests, or source files)
- **Matrix Builds**: Builds 2 images × 2 variants × 2 public versions = 8 images per workflow run
- **BTRFS Caching**: Uses BTRFS with zstd compression for efficient build caching
- **Signing Pipeline**: All images signed with cosign after successful build
- **SBOM Generation**: Software Bill of Materials generated for supply chain security

### Key System Customizations

1. **Flathub Configuration**: Replaces Fedora Flatpaks with Flathub, sets up automatic updates
2. **Enhanced Hardware Support**: Extensive udev rules for gaming peripherals, RGB devices, racing wheels
3. **Multimedia Codecs**: Uses negativo17's unrestricted Mesa and FFmpeg (vs. Fedora's patent-crippled versions)
4. **Update Services**: Automatic system and Flatpak updates via systemd timers
5. **Security**: pam-u2f and pam_yubico for hardware key authentication

### File Structure

```
Containerfile              # Main build definition (multi-stage)
Justfile                   # Build automation (537 lines, 20+ commands)
packages.json              # Package inclusion/exclusion manifest
image-versions.yaml        # SHA digest pinning for reproducible builds
build_files/               # Installation scripts (run in order):
  ├── install.sh           # Core system setup (142 lines)
  ├── packages.sh          # Package management (42 lines)
  ├── nvidia-install.sh    # NVIDIA drivers (111 lines)
  ├── initramfs.sh         # Boot image generation (10 lines)
  └── post-install.sh      # Cleanup and service enablement (56 lines)
sys_files/                 # System configuration files copied to /
.github/workflows/         # CI/CD automation
  └── reusable-build.yml   # Main build workflow
```

### Important Notes

- **Never modify kernels without updating versionlock**: The kernel is locked to ensure akmods compatibility
- **Test with `just run`**: Always test images locally before pushing
- **Update digests in image-versions.yaml**: When bumping base image versions, update SHA digests
- **NVIDIA builds require matching akmods**: NVIDIA variant must use akmods built for the same kernel version
- **Bootc compliance required**: All images must pass `bootc container lint` validation
- **Security-first**: All components must be cryptographically signed and verified
- **Rawhide pulls latest**: Rawhide version doesn't use digest pinning and pulls :rawhide tag directly (for bleeding edge testing)

### Dependency Chain

```
Fedora Base Images (quay.io/fedora-ostree-desktops)
    ↓ (verified via SHA digest + cosign)
Signed Kernels (ublue-os/akmods)
    ↓ (verified via SHA digest + cosign)
Main Images (this repo)
    ↓ (signed with cosign)
Downstream Projects (Aurora, Bazzite, Bluefin)
    ↓
End Users
```

### Development Workflow

1. Make changes to `build_files/`, `sys_files/`, or `packages.json`
2. Run `just check` to verify formatting
3. Run `just build <image> <version> <variant>` to build locally
4. Run `just run <image> <version> <variant>` to test the built image
5. Run `just secureboot <image> <version> <variant>` to verify kernel signatures
6. Commit changes - CI will automatically build and sign images if dependencies changed

### Common Issues

- **Build fails with "kernel module mismatch"**: Update `KERNEL_VERSION` in Containerfile to match akmods
- **Image signature verification fails**: Check that `image-versions.yaml` has correct digest for the base image
- **Package conflicts during build**: Check `packages.json` for conflicting include/exclude rules
- **NVIDIA build fails**: Ensure `AKMODS_NVIDIA_DIGEST` matches the kernel version being used
