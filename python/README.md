# Python Dockerfile Template

Multi-stage build using [Astral uv](https://github.com/astral-sh/uv) to build the application virtual environment, producing a lean production runtime image without `uv`.

## Build Arguments (`ARG`)

| Argument | Default | Description |
| --- | --- | --- |
| `PYTHON_VERSION` | `3.12` | Python runtime version (must match between builder and runtime) |
| `DEBIAN_SUITE` | `bookworm` | Debian suite (`bookworm`, `trixie`) |
| `PORT` | `8000` | Application port |
| `APP_USER` | `appuser` | Unprivileged service user |
| `APP_UID` | `10001` | Non-root UID / GID |

---

## Build Examples

**Standard build:**

```bash
docker build -t my-python-app:latest .
```

**Custom Python version (e.g., 3.11) and Port:**

```bash
docker build \
  --build-arg PYTHON_VERSION=3.11 \
  --build-arg PORT=5000 \
  -t my-python-app:latest .
```

---

## Framework Entrypoints

Edit the `CMD` directive in Stage 2 to match your framework:

- **FastAPI** (Default):

  ```dockerfile
  CMD ["fastapi", "run", "--host", "0.0.0.0", "--port", "8000"]
  ```

- **Uvicorn**:

  ```dockerfile
  CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
  ```

- **Gunicorn (with Uvicorn workers)**:

  ```dockerfile
  CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "app.main:app", "-b", "0.0.0.0:8000"]
  ```

- **Standalone script or worker**:

  ```dockerfile
  CMD ["python", "-m", "app.main"]
  ```

---

## Development with Docker Compose

Use [docker-compose.yml](file:///home/gabrigas/Selene/Adventure/dockerfiles/python/docker-compose.yml) with [Compose Watch](https://docs.docker.com/compose/file-watch/) for live file syncing and automatic lockfile rebuilds:

```bash
docker compose up --watch 
```

- **File Sync**: Local code changes sync instantly to `/app` inside the container without requiring rebuilds.
- **Auto-Rebuild**: Editing `uv.lock` automatically rebuilds the image.

---

## Recommended `.dockerignore`

Copy from [.dockerignore](./.dockerignore):

```text
.git
.gitignore
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
.ruff_cache
.mypy_cache
dist
build
.env
.env.*
```
