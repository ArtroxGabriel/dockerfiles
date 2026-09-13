# Java Dockerfile Template

Multi-stage template for Java and Spring Boot applications with layered JAR extraction and BuildKit cache mounts.

## Customization Guide

### 1. Build Arguments (`ARG`)

| Argument | Default | Description |
|---|---|---|
| `JAVA_VERSION` | `21` | Temurin base image version |
| `BASE_OS` | `jammy` | OS variant (`jammy`, `noble`, etc.) |
| `PORT` | `8080` | Container port and Spring Boot `SERVER_PORT` |
| `APP_USER` | `appuser` | Unprivileged service user |
| `APP_UID` | `10001` | Non-root UID / GID |

Example:
```bash
docker build --build-arg JAVA_VERSION=17 --build-arg PORT=9090 -t my-app:latest .
```

### 2. Switching to Maven
Default is Gradle. For Maven:
1. Comment out **Option A: Gradle** in Stage 1.
2. Uncomment **Option B: Maven** in Stage 1.

### 3. Spring Boot Versions
- **Spring Boot 3.2+, 3.4+, & 4.x** (Default): Uses `java -Djarmode=tools -jar app.jar extract` and launcher `org.springframework.boot.loader.launch.JarLauncher`.
- **Spring Boot 3.1 or earlier**:
  - In Stage 2: `RUN java -Djarmode=layertools -jar app.jar extract`
  - In Stage 3: `ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]`

### 4. Non-Spring Boot / Plain Fat JAR
For standard executable JARs (Quarkus, Micronaut, Dropwizard, plain Java):
1. Remove or skip Stage 2 (extractor).
2. In Stage 3, copy the JAR directly and run with `-jar`:
   ```dockerfile
   COPY --from=builder --chown=${APP_USER}:${APP_USER} /workspace/app.jar /app/app.jar
   ENTRYPOINT ["java", "-jar", "/app/app.jar"]
   ```

### 5. Healthcheck
- Probes `http://localhost:${SERVER_PORT}/actuator/health` via `curl`.
- Automatically syncs with `--build-arg PORT=...` via `SERVER_PORT`.
- If using Kubernetes probes, the `HEALTHCHECK` directive can be removed.

### 6. Dependency Cache Warming (Gradle)
`./gradlew --no-daemon dependencies` warms standard configurations. For full warm-up including plugins:
```dockerfile
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon build -x test -x bootJar --dry-run
```

---

## Recommended `.dockerignore`

Copy from [.dockerignore](./.dockerignore):

```text
.git
.gitignore
.gradle
build
target
out
.idea
*.iml
.vscode
```
