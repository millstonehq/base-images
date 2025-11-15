VERSION 0.8
PROJECT millstonehq/base-images

# ========================================
# Base Builder Image
# ========================================
# Wolfi-based image with essential build tools
# Use this as the foundation for language-specific builders

base-builder:
    FROM cgr.dev/chainguard/wolfi-base:latest
    RUN apk add \
        bash \
        git \
        jq \
        wget \
        yq \
        curl \
        build-base \
        bison \
        flex

    # /app is the generic folder set up by default for all workspace content
    WORKDIR /app
    RUN chown nonroot:nonroot /app

    # Lockfile and similar concept well known folder. Used for installation optimization
    RUN mkdir -p /_lock && chown nonroot:nonroot /_lock

    RUN mkdir -p /home/nonroot/.local/bin /home/nonroot/.local/share
    RUN chown -R nonroot:nonroot /home/nonroot/.local

    # Add ~/.local/bin to PATH so that nonroot user can see installed packages
    ENV PATH="/home/nonroot/.local/bin:$PATH"

    # https://github.com/earthly/lib/pull/58
    # This is fixed in earthly-lib 3.0.3 release, but at the time of this writing the
    # earthly release that pins earthly-lib is only 3.0.1
    # https://github.com/earthly/earthly/blob/main/Earthfile#L908
    # Once earthly release that pins >= 3.0.3 is released we can remove this
    ENV OTEL_TRACES_EXPORTER=none

    USER nonroot

    SAVE IMAGE ghcr.io/millstonehq/base-builder:latest

# ========================================
# Base Runtime Image
# ========================================
# Minimal runtime base - no build tools, only essential runtime components
# Use this for production container images that need minimal footprint

base-runtime:
    FROM cgr.dev/chainguard/wolfi-base:latest

    # Set up workspace and user directories (minimal setup)
    WORKDIR /app
    RUN chown nonroot:nonroot /app

    RUN mkdir -p /home/nonroot/.local/bin /home/nonroot/.local/share
    RUN chown -R nonroot:nonroot /home/nonroot/.local

    # Add ~/.local/bin to PATH for consistency
    ENV PATH="/home/nonroot/.local/bin:$PATH"

    # Earthly OTEL fix (same as base-builder)
    ENV OTEL_TRACES_EXPORTER=none

    USER nonroot

    SAVE IMAGE ghcr.io/millstonehq/base-runtime:latest

# ========================================
# Go Builder Image
# ========================================
# Inherits from base-builder and adds Go toolchain
# Use this for building Go applications

base-go:
    ARG GOLANG_VERSION=1.22
    FROM +base-builder

    USER root
    RUN apk add go~=${GOLANG_VERSION}
    USER nonroot

    # Verify Go installation
    RUN go version

    ENV PATH="${PATH}:$(go env GOPATH)/bin"

    SAVE IMAGE ghcr.io/millstonehq/base-go:latest

# ========================================
# Go Runtime Image
# ========================================
# Minimal Go runtime base - just base-runtime (no Go needed)
# Use this for compiled Go binaries that are statically linked

base-go-runtime:
    FROM +base-runtime

    SAVE IMAGE ghcr.io/millstonehq/base-go-runtime:latest

# ========================================
# Build All Images
# ========================================
# Target to build all base images

all:
    BUILD +base-builder
    BUILD +base-runtime
    BUILD +base-go
    BUILD +base-go-runtime

# ========================================
# Publish All Images
# ========================================
# Target to publish all base images to GHCR
# Run with: earthly --push +publish --VERSION=v1.0.0

publish:
    ARG VERSION=latest

    # Build and tag all images with version
    BUILD +base-builder --tag=$VERSION
    BUILD +base-runtime --tag=$VERSION
    BUILD +base-go --tag=$VERSION
    BUILD +base-go-runtime --tag=$VERSION

# ========================================
# Versioned Image Targets
# ========================================
# These allow building images with specific tags

base-builder-versioned:
    ARG tag=latest
    FROM +base-builder
    SAVE IMAGE --push ghcr.io/millstonehq/base-builder:${tag}

base-runtime-versioned:
    ARG tag=latest
    FROM +base-runtime
    SAVE IMAGE --push ghcr.io/millstonehq/base-runtime:${tag}

base-go-versioned:
    ARG tag=latest
    ARG GOLANG_VERSION=1.22
    FROM +base-go --GOLANG_VERSION=${GOLANG_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/base-go:${tag}

base-go-runtime-versioned:
    ARG tag=latest
    FROM +base-go-runtime
    SAVE IMAGE --push ghcr.io/millstonehq/base-go-runtime:${tag}

# Multi-platform publishing
publish-multiarch:
    ARG tag=latest
    ARG GOLANG_VERSION=1.22
    FROM alpine:latest

    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-builder-versioned --tag=${tag}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-runtime-versioned --tag=${tag}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-go-versioned --tag=${tag} --GOLANG_VERSION=${GOLANG_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-go-runtime-versioned --tag=${tag}
