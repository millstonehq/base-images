# Millstone Base Images

Public base container images for Millstone projects, built with [Earthly](https://earthly.dev) on [Chainguard Wolfi](https://github.com/wolfi-dev).

## Available Images

All images are published to GitHub Container Registry (GHCR) with multi-arch support (amd64, arm64).

**Language Images:**
- **Go:** `go:1.25`, `go:1.25-runtime`
- **Python:** `python:3.14`, `python:3.14-runtime`
- **Java:** `java:25`, `java:25-runtime`

**Foundation Images:**
- **Builder:** `base:builder`
- **Runtime:** `base:runtime`

### Base Builder (`base:builder`)

Foundation image with essential build tools. Use this as the base for language-specific builders.

**Includes:** bash, git, jq, yq, wget, curl, build-base, bison, flex

```dockerfile
FROM ghcr.io/millstonehq/base:builder
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-builder
```

### Base Runtime (`base:runtime`)

Minimal runtime image with no build tools. Use for production containers.

```dockerfile
FROM ghcr.io/millstonehq/base:runtime
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-runtime
```

### Go (`go:1.25`)

Extends `base:builder` with Go toolchain.

**Versions:** 1.25 (configurable via `GOLANG_VERSION` arg)

```dockerfile
FROM ghcr.io/millstonehq/go:1.25
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-go --GOLANG_VERSION=1.25
```

### Go Runtime (`go:1.25-runtime`)

Minimal runtime for statically-compiled Go binaries (no Go toolchain included).

```dockerfile
FROM ghcr.io/millstonehq/go:1.25-runtime
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-go-runtime --GOLANG_VERSION=1.25
```

### Python (`python:3.14`)

Extends `base:builder` with Python toolchain and uv package manager.

**Versions:** 3.14 (configurable via `PYTHON_VERSION` arg)

**Includes:** python, pip, uv (configured for Docker-optimized usage)

```dockerfile
FROM ghcr.io/millstonehq/python:3.14
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-python --PYTHON_VERSION=3.14
```

### Python Runtime (`python:3.14-runtime`)

Minimal runtime for Python applications (no build tools, pip included).

```dockerfile
FROM ghcr.io/millstonehq/python:3.14-runtime
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-python-runtime --PYTHON_VERSION=3.14
```

### Java (`java:25`)

Extends `base:builder` with JDK, Maven, and Gradle.

**Versions:** 25 (LTS, configurable via `JAVA_VERSION` arg)

**Includes:** OpenJDK, Maven, Gradle

```dockerfile
FROM ghcr.io/millstonehq/java:25
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-java --JAVA_VERSION=25
```

### Java Runtime (`java:25-runtime`)

Minimal runtime for Java applications (JRE only, no build tools).

```dockerfile
FROM ghcr.io/millstonehq/java:25-runtime
```

**Earthfile usage:**
```earthfile
IMPORT github.com/millstonehq/base-images AS base
FROM base+base-java-runtime --JAVA_VERSION=25
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

### Building a Python Application

```earthfile
VERSION 0.8

IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

build:
    FROM base+base-python --PYTHON_VERSION=3.14
    WORKDIR /app

    COPY pyproject.toml uv.lock ./
    RUN uv sync --frozen --no-dev

    COPY . .
    RUN uv sync --frozen

    SAVE ARTIFACT .venv

image:
    FROM base+base-python-runtime
    COPY +build/.venv /app/.venv
    COPY . /app
    ENTRYPOINT ["python", "main.py"]
    SAVE IMAGE --push ghcr.io/myorg/myapp:latest
```

### Building a Java Application

```earthfile
VERSION 0.8

IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

build:
    FROM base+base-java --JAVA_VERSION=25
    WORKDIR /app

    COPY pom.xml ./
    RUN mvn dependency:resolve

    COPY src ./src
    RUN mvn package -DskipTests

    SAVE ARTIFACT target/*.jar app.jar

image:
    FROM base+base-java-runtime
    COPY +build/app.jar /app/app.jar
    ENTRYPOINT ["java", "-jar", "/app/app.jar"]
    SAVE IMAGE --push ghcr.io/myorg/myapp:latest
```

### Custom Builder Image

```earthfile
IMPORT github.com/millstonehq/base-images:v1.0.0 AS base

custom-builder:
    FROM base+base-builder

    # Add custom build tools
    USER root
    RUN apk add nodejs npm
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

## Contributing

1. Create PR with changes
2. CI tests base images
3. Merge triggers automated release
4. Update downstream projects to new version

## License

Apache 2.0 - See [LICENSE](LICENSE)

## Support

- **Issues:** https://github.com/millstonehq/base-images/issues



## Test
This is a test to verify the feedback loop is fixed.
