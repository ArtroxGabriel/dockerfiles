# Go Dockerfile Template

Multi-stage build compiling a statically linked Go binary into a minimal `scratch` image.

## Build Arguments (`ARG`)

| Argument | Default | Description |
|---|---|---|
| `GO_VERSION` | `1.27.1` | Go base image version (`golang:${GO_VERSION}-alpine`) |
| `APP_NAME` | `app` | Output binary name |
| `TARGET_PATH` | `./cmd/api` | Path to package containing `main.go` |

---

## Build Examples

**Standard build:**
```bash
docker build -t my-go-app:latest .
```

**Custom Go version and target:**
```bash
docker build \
  --build-arg GO_VERSION=1.23.1 \
  --build-arg TARGET_PATH=./cmd/server \
  -t my-go-app:latest .
```

**Multi-architecture build:**
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t my-go-app:latest .
```

---

## Recommended `.dockerignore`

Copy from [.dockerignore](./.dockerignore):

```text
.git
.gitignore
bin
dist
*.exe
*.test
*.out
.idea
.vscode
```
