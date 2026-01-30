# Multi-Architecture Image Build Best Practices

This document provides best practices for building and distributing OCI images that support multiple CPU architectures, in accordance with the [OCI Image Index Specification](image-index.md).

## Overview

Multi-architecture (multi-arch) images allow a single image reference to support multiple CPU architectures and operating systems.
This is achieved through the use of an [image index](image-index.md) (also known as a "fat manifest") that references platform-specific [image manifests](manifest.md).

## Common Architectures

The following CPU architectures are commonly supported in multi-arch images:

- `amd64` (x86-64, Intel/AMD 64-bit)
- `arm64` (ARM 64-bit, also known as aarch64)
- `arm` (ARM 32-bit with variants v6, v7, v8)
- `ppc64le` (PowerPC 64-bit little-endian)
- `s390x` (IBM System z)
- `riscv64` (RISC-V 64-bit)

See the [Platform Variants](image-index.md#platform-variants) table for variant specifications.

## Building Multi-Architecture Images

### Using Docker Buildx

Docker Buildx provides native support for building multi-platform images:

```bash
# Create a new builder instance (one-time setup)
docker buildx create --name multiarch --use

# Build and push a multi-arch image
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myregistry/myimage:latest \
  --push .
```

### Platform-Specific Dependencies

When building images that install platform-specific binaries or packages, use build arguments provided by Docker Buildx to determine the target architecture:

- `TARGETPLATFORM` - The target platform in format `os/arch[/variant]` (e.g., `linux/amd64`, `linux/arm64`)
- `TARGETOS` - The target operating system (e.g., `linux`, `windows`)
- `TARGETARCH` - The target architecture (e.g., `amd64`, `arm64`)
- `TARGETVARIANT` - The target variant (e.g., `v7` for ARM)

#### Example: Installing Architecture-Specific Packages

**❌ Incorrect Approach:**

```dockerfile
# This will fail for arm64 builds
RUN curl -o package.deb https://example.com/amd64/package.deb && \
    dpkg -i package.deb
```

**✅ Correct Approach:**

```dockerfile
# Automatically set by Docker Buildx
ARG TARGETARCH

RUN case "${TARGETARCH}" in \
      amd64) \
        PACKAGE_URL="https://example.com/x86_64/package.deb" \
        ;; \
      arm64) \
        PACKAGE_URL="https://example.com/aarch64/package.deb" \
        ;; \
      *) \
        echo "Unsupported architecture: ${TARGETARCH}" && exit 1 \
        ;; \
    esac && \
    curl -o package.deb "${PACKAGE_URL}" && \
    dpkg -i package.deb && \
    rm package.deb
```

#### Example: AWS Session Manager Plugin

The [AWS Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) requires architecture-specific downloads:

**❌ Incorrect Approach:**

```dockerfile
# This downloads amd64 version regardless of build platform
RUN curl -o session-manager-plugin.deb \
      "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" && \
    dpkg -i session-manager-plugin.deb
```

**✅ Correct Approach:**

```dockerfile
ARG TARGETARCH

RUN case "${TARGETARCH}" in \
      amd64) \
        ARCH_PATH="ubuntu_64bit" \
        ;; \
      arm64) \
        ARCH_PATH="ubuntu_arm64" \
        ;; \
      *) \
        echo "Unsupported architecture for Session Manager Plugin: ${TARGETARCH}" && exit 1 \
        ;; \
    esac && \
    curl -o session-manager-plugin.deb \
      "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/${ARCH_PATH}/session-manager-plugin.deb" && \
    dpkg -i session-manager-plugin.deb && \
    rm session-manager-plugin.deb
```

### Using Base Images with Multi-Arch Support

Most official Docker images provide multi-arch support:

```dockerfile
# This automatically pulls the correct architecture
FROM ubuntu:22.04

# The base image will match the target platform
```

Verify that your base images support all target architectures before building.

## Testing Multi-Architecture Images

### Local Testing with QEMU

Docker Desktop includes QEMU emulation for testing non-native architectures:

```bash
# List supported platforms
docker buildx ls

# Build and load an arm64 image for local testing
docker buildx build --platform linux/arm64 \
  -t myimage:arm64 \
  --load .

# Run the arm64 image (will use QEMU emulation on amd64 hosts)
docker run --rm myimage:arm64
```

### Inspecting Multi-Arch Images

Use `docker buildx imagetools` to inspect the manifest list:

```bash
# View the manifest list
docker buildx imagetools inspect myregistry/myimage:latest
```

Example output:

```text
Name:      myregistry/myimage:latest
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:1234567890abcdef...

Manifests:
  Name:      myregistry/myimage:latest@sha256:abcdef...
  MediaType: application/vnd.oci.image.manifest.v1+json
  Platform:  linux/amd64

  Name:      myregistry/myimage:latest@sha256:fedcba...
  MediaType: application/vnd.oci.image.manifest.v1+json
  Platform:  linux/arm64
```

## Common Pitfalls

### 1. Architecture Mismatch

**Problem:** Installing packages built for one architecture on a different architecture.

**Example Error:**

```text
dpkg: error processing archive package.deb (--install):
 package architecture (amd64) does not match system (arm64)
```

**Solution:** Always use `TARGETARCH` or `TARGETPLATFORM` to determine the correct package variant.

### 2. Missing Platform Support

**Problem:** Using dependencies that don't support all target architectures.

**Solution:**

- Verify package availability for all target platforms before adding them
- Use conditional installation with error handling for unsupported platforms
- Document supported architectures in your image documentation

### 3. Hardcoded Architecture References

**Problem:** Using hardcoded architecture names like `x86_64`, `amd64`, or `aarch64` inconsistently.

**Solution:** Create a mapping function to normalize architecture names:

```dockerfile
ARG TARGETARCH

RUN case "${TARGETARCH}" in \
      amd64) \
        # Map to various naming conventions
        ARCH_ALT1="x86_64" \
        ARCH_ALT2="amd64" \
        ARCH_ALT3="x64" \
        ;; \
      arm64) \
        ARCH_ALT1="aarch64" \
        ARCH_ALT2="arm64" \
        ;; \
    esac
```

### 4. Build-Time vs Runtime Architecture

**Problem:** Using `uname -m` or similar commands to detect architecture during build.

**Issue:** These commands return the build host architecture, not the target architecture when cross-compiling.

**Solution:** Always use Docker build arguments (`TARGETARCH`, `TARGETPLATFORM`) instead of runtime detection during build.

## Registry Support

Ensure your container registry supports OCI image indexes:

- Docker Hub: ✅ Supported
- GitHub Container Registry (GHCR): ✅ Supported
- Google Container Registry (GCR): ✅ Supported
- Amazon Elastic Container Registry (ECR): ✅ Supported
- Azure Container Registry (ACR): ✅ Supported
- Harbor: ✅ Supported (v2.0+)

See [OCI Image Implementations](implementations.md) for a full list.

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Build Multi-Arch Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

## Best Practices Summary

1. **Use Build Arguments:** Always use `TARGETARCH`, `TARGETPLATFORM`, etc., instead of runtime detection
2. **Conditional Installation:** Use case statements to install architecture-specific packages
3. **Verify Support:** Check that all dependencies support your target architectures
4. **Test Locally:** Use QEMU emulation for testing non-native architectures
5. **Inspect Manifests:** Use `docker buildx imagetools inspect` to verify multi-arch images
6. **Document Limitations:** Clearly document which architectures are supported
7. **Handle Failures:** Provide clear error messages for unsupported architectures
8. **Normalize Names:** Be aware of architecture naming variations across different ecosystems

## Additional Resources

- [OCI Image Index Specification](image-index.md)
- [OCI Image Manifest Specification](manifest.md)
- [Docker Buildx Documentation](https://docs.docker.com/build/building/multi-platform/)
- [QEMU User Emulation](https://www.qemu.org/docs/master/user/main.html)
- [Go Environment Variables (GOARCH/GOOS)](https://go.dev/doc/install/source#environment)
