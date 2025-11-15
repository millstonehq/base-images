# Millstone Base Images

Public base container images for Millstone projects, built with [Earthly](https://earthly.dev) on [Chainguard Wolfi](https://github.com/wolfi-dev).

## Available Images

All images are published to GitHub Container Registry (GHCR) with multi-arch support (amd64, arm64).

### Base Builder (`base-builder`)

Foundation image with essential build tools. Use this as the base for language-specific builders.

**Includes:** bash, git, jq, yq, wget, curl, build-base, bison, flex

```dockerfile
FROM ghcr.io/millstonehq/base-builder:v1.0.0
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base
FROM base+base-builder
```

### Base Runtime (`base-runtime`)

Minimal runtime image with no build tools. Use for production containers.

```dockerfile
FROM ghcr.io/millstonehq/base-runtime:v1.0.0
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base
FROM base+base-runtime
```

### Go Builder (`base-go`)

Extends `base-builder` with Go toolchain.

**Default:** Go 1.22 (configurable via `GOLANG_VERSION` arg)

```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base
FROM base+base-go --GOLANG_VERSION=1.25
```

### Go Runtime (`base-go-runtime`)

Minimal runtime for statically-compiled Go binaries (no Go toolchain included).

```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base
FROM base+base-go-runtime
```

## Usage Examples

### Building a Go Application

```earthfile
VERSION 0.8

IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

build:
    FROM base+base-go --GOLANG_VERSION=1.25
    WORKDIR /app

    COPY go.mod go.sum ./
    RUN go mod download

    COPY . .
    RUN CGO_ENABLED=0 go build -o myapp ./cmd/myapp

    SAVE ARTIFACT myapp

image:
    FROM base+base-go-runtime
    COPY +build/myapp /app/myapp
    ENTRYPOINT ["/app/myapp"]
    SAVE IMAGE --push ghcr.io/myorg/myapp:latest
```

### Custom Builder Image

```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

custom-builder:
    FROM base+base-builder

    # Add custom build tools
    USER root
    RUN apk add python3 nodejs npm
    USER nonroot

    SAVE IMAGE myorg/custom-builder:latest
```

## Versioning

We follow [Semantic Versioning](https://semver.org/):

- **Major (v2.0.0):** Breaking changes (different base image, removed packages)
- **Minor (v1.1.0):** New packages, non-breaking updates
- **Patch (v1.0.1):** Security updates, Wolfi base updates

### Version Pinning

**Recommended:** Pin to minor versions for stability

```earthfile
# Good: receives patch updates automatically
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

# Also valid: pin to exact version
IMPORT github.com/millstonehq/base-images@sha256:abc123... AS base
```

### Latest Tags

- `latest` - tracks the most recent release (not recommended for production)
- `v1.0.0` - specific version (recommended)

## Security

- **Base:** Chainguard Wolfi (minimal, CVE-aware)
- **User:** All images run as `nonroot` user by default
- **Scanning:** Automated security scanning on every release
- **Updates:** Patch releases published within 48 hours of Wolfi updates

## Contributing

This repository is public to support our FOSS projects. Internal Millstone developers should:

1. Create PR with changes
2. CI tests base images + downstream projects (mill)
3. Merge triggers automated release
4. Update downstream projects to new version

## License

Apache 2.0 - See [LICENSE](LICENSE)

## Support

- **Issues:** https://github.com/millstonehq/base-images/issues
- **Internal:** Slack #platform-engineering
