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
        curl

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

    SAVE IMAGE ghcr.io/millstonehq/base:builder

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

    SAVE IMAGE ghcr.io/millstonehq/base:runtime

# ========================================
# Go Builder Image
# ========================================
# Inherits from base-builder and adds Go toolchain
# Use this for building Go applications

base-go:
    ARG GOLANG_VERSION=1.25
    FROM +base-builder

    USER root
    RUN apk add go~=${GOLANG_VERSION}
    USER nonroot

    # Verify Go installation
    RUN go version

    ENV PATH="${PATH}:$(go env GOPATH)/bin"

    SAVE IMAGE ghcr.io/millstonehq/go:${GOLANG_VERSION}

# ========================================
# Go Runtime Image
# ========================================
# Minimal Go runtime base - just base-runtime (no Go needed)
# Use this for compiled Go binaries that are statically linked

base-go-runtime:
    ARG GOLANG_VERSION=1.25
    FROM +base-runtime

    SAVE IMAGE ghcr.io/millstonehq/go:${GOLANG_VERSION}-runtime

# ========================================
# Python Builder Image
# ========================================
# Inherits from base-builder and adds Python toolchain + uv
# Use this for building Python applications

base-python:
    ARG PYTHON_VERSION=3.14
    FROM +base-builder

    USER root
    RUN apk add python-${PYTHON_VERSION} py${PYTHON_VERSION}-pip
    USER nonroot

    # Install uv package manager
    RUN pip install -Iv "uv==0.7.10"

    # Configure uv for optimal Docker usage
    # - Use copy mode instead of hard links
    # - Byte-compile packages for faster startups
    # - Prevent downloading isolated Python builds
    # - Set Python version and project environment
    ENV UV_LINK_MODE=copy
    ENV UV_COMPILE_BYTECODE=1
    ENV UV_PYTHON_DOWNLOADS=never
    ENV UV_PYTHON=python${PYTHON_VERSION}
    ENV UV_PROJECT_ENVIRONMENT=/app

    # Verify Python installation
    RUN python${PYTHON_VERSION} --version && uv --version

    SAVE IMAGE ghcr.io/millstonehq/python:${PYTHON_VERSION}

# ========================================
# Python Runtime Image
# ========================================
# Minimal Python runtime base - just runtime + Python, no build tools
# Use this for production Python applications

base-python-runtime:
    ARG PYTHON_VERSION=3.14
    FROM +base-runtime

    USER root
    RUN apk add python-${PYTHON_VERSION}
    USER nonroot

    ENV UV_PYTHON=python${PYTHON_VERSION}

    SAVE IMAGE ghcr.io/millstonehq/python:${PYTHON_VERSION}-runtime

# ========================================
# Java Builder Image
# ========================================
# Inherits from base-builder and adds JDK + Maven + Gradle
# Use this for building Java applications

base-java:
    ARG JAVA_VERSION=25
    FROM +base-builder

    USER root
    RUN apk add openjdk-${JAVA_VERSION}-default-jdk maven gradle

    # Create maven repository in nonroot user's home directory
    RUN mkdir -p /home/nonroot/.m2 && \
        chown -R nonroot:nonroot /home/nonroot/.m2

    # Configure Maven to use the cache directory
    ENV MAVEN_OPTS="-Dmaven.repo.local=/home/nonroot/.m2"
    ENV JAVA_HOME=/usr/lib/jvm/java-${JAVA_VERSION}-openjdk

    USER nonroot

    # Verify Java installation
    RUN java -version && mvn -version && gradle -version

    SAVE IMAGE ghcr.io/millstonehq/java:${JAVA_VERSION}

# ========================================
# Java Runtime Image
# ========================================
# Minimal Java runtime base - just runtime + JRE, no build tools
# Use this for production Java applications

base-java-runtime:
    ARG JAVA_VERSION=25
    FROM +base-runtime

    USER root
    RUN apk add openjdk-${JAVA_VERSION}-jre
    USER nonroot

    ENV JAVA_HOME=/usr/lib/jvm/java-${JAVA_VERSION}-openjdk

    SAVE IMAGE ghcr.io/millstonehq/java:${JAVA_VERSION}-runtime

# ========================================
# OpenTofu Builder Image
# ========================================
# OpenTofu builder base - builder + tofu for schema extraction and builds
# Use this for building Crossplane providers that need terraform schema

base-tofu-builder:
    FROM +base-builder

    USER root
    RUN apk add opentofu
    USER nonroot

    SAVE IMAGE ghcr.io/millstonehq/tofu:builder

# ========================================
# OpenTofu Runtime Image
# ========================================
# Minimal tofu runtime base - just runtime + tofu, no build tools
# Use this for running Crossplane providers in production
# Includes terraform→tofu symlink for Upjet compatibility

base-tofu-runtime:
    FROM +base-runtime

    USER root
    # Install OpenTofu (Upjet downloads provider plugins at runtime)
    RUN apk add opentofu
    # Upjet looks for "terraform" binary, create symlink
    RUN ln -sf /usr/bin/tofu /usr/bin/terraform
    USER nonroot

    SAVE IMAGE ghcr.io/millstonehq/tofu:runtime

# ========================================
# Build All Images
# ========================================
# Target to build all base images

all:
    BUILD +base-builder
    BUILD +base-runtime
    BUILD +base-go
    BUILD +base-go-runtime
    BUILD +base-python
    BUILD +base-python-runtime
    BUILD +base-java
    BUILD +base-java-runtime
    BUILD +base-tofu-builder
    BUILD +base-tofu-runtime

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
    BUILD +base-python --tag=$VERSION
    BUILD +base-python-runtime --tag=$VERSION
    BUILD +base-java --tag=$VERSION
    BUILD +base-java-runtime --tag=$VERSION

# ========================================
# Versioned Image Targets
# ========================================
# These allow building images with specific tags

base-builder-versioned:
    FROM +base-builder
    SAVE IMAGE --push ghcr.io/millstonehq/base:builder

base-runtime-versioned:
    FROM +base-runtime
    SAVE IMAGE --push ghcr.io/millstonehq/base:runtime

base-go-versioned:
    ARG GOLANG_VERSION=1.25
    FROM +base-go --GOLANG_VERSION=${GOLANG_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/go:${GOLANG_VERSION}

base-go-runtime-versioned:
    ARG GOLANG_VERSION=1.25
    FROM +base-go-runtime --GOLANG_VERSION=${GOLANG_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/go:${GOLANG_VERSION}-runtime

base-python-versioned:
    ARG PYTHON_VERSION=3.14
    FROM +base-python --PYTHON_VERSION=${PYTHON_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/python:${PYTHON_VERSION}

base-python-runtime-versioned:
    ARG PYTHON_VERSION=3.14
    FROM +base-python-runtime --PYTHON_VERSION=${PYTHON_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/python:${PYTHON_VERSION}-runtime

base-java-versioned:
    ARG JAVA_VERSION=25
    FROM +base-java --JAVA_VERSION=${JAVA_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/java:${JAVA_VERSION}

base-java-runtime-versioned:
    ARG JAVA_VERSION=25
    FROM +base-java-runtime --JAVA_VERSION=${JAVA_VERSION}
    SAVE IMAGE --push ghcr.io/millstonehq/java:${JAVA_VERSION}-runtime

base-tofu-builder-versioned:
    FROM +base-tofu-builder
    SAVE IMAGE --push ghcr.io/millstonehq/tofu:builder

base-tofu-runtime-versioned:
    FROM +base-tofu-runtime
    SAVE IMAGE --push ghcr.io/millstonehq/tofu:runtime

# Multi-platform publishing
publish-multiarch:
    ARG GOLANG_VERSION=1.25
    ARG PYTHON_VERSION=3.14
    ARG JAVA_VERSION=25
    FROM cgr.dev/chainguard/wolfi-base:latest

    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-builder-versioned
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-runtime-versioned
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-go-versioned --GOLANG_VERSION=${GOLANG_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-go-runtime-versioned --GOLANG_VERSION=${GOLANG_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-python-versioned --PYTHON_VERSION=${PYTHON_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-python-runtime-versioned --PYTHON_VERSION=${PYTHON_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-java-versioned --JAVA_VERSION=${JAVA_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-java-runtime-versioned --JAVA_VERSION=${JAVA_VERSION}
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-tofu-builder-versioned
    BUILD --platform=linux/amd64 --platform=linux/arm64 +base-tofu-runtime-versioned
