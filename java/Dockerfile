# syntax=docker/dockerfile:1

# ==============================================================================
# Java / Spring Boot Multi-Stage Dockerfile Template
#
# Highlights:
# 1. Multi-Stage Build: Separates build tooling (JDK) from production runtime (JRE).
# 2. BuildKit Cache: Caches dependencies across builds (/root/.gradle or /root/.m2).
# 3. Layered JARs: Separates dependencies from application code for fast rebuilds.
# 4. Security: Runs as an unprivileged non-root user (UID 10001).
# 5. Production JVM Flags: Configures container-aware RAM limits and graceful shutdown.
# ==============================================================================

# ------------------------------------------------------------------------------
# Global Build Arguments
# Customize Java version or OS flavor (e.g., jammy, noble, alpine).
# ------------------------------------------------------------------------------
ARG JAVA_VERSION=21
ARG BASE_OS=jammy

# ==============================================================================
# Stage 1: Build & Package
# ==============================================================================
FROM eclipse-temurin:${JAVA_VERSION}-jdk-${BASE_OS} AS builder
WORKDIR /workspace

# ------------------------------------------------------------------------------
# Build Tool Configuration
# DEFAULT: Gradle.
# TO USE MAVEN: Comment out the Gradle section and uncomment the Maven section below.
# ------------------------------------------------------------------------------

# --- [Option A: Gradle (Default)] ---
# Copy build scripts & wrapper first to leverage Docker layer caching for dependencies.
# The glob patterns handle both Groovy (*.gradle) and Kotlin (*.gradle.kts) DSL,
# and gracefully ignore gradle.properties if not present in your project.
COPY gradlew settings.gradle* build.gradle* gradle.properties* ./
COPY gradle/ ./gradle/
RUN chmod +x gradlew

# Pre-fetch dependencies into BuildKit cache mount (does not compile source code).
# Note: 'dependencies' warms the cache for main configurations; for a complete warm-up including plugins,
# you can alternatively use: ./gradlew --no-daemon build -x test -x bootJar --dry-run
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon dependencies

# Copy application source code and compile
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon clean bootJar -x test

# Normalize the produced JAR into a fixed name (excludes plain/unbundled JARs)
RUN cp $(find build/libs -name "*.jar" ! -name "*-plain.jar" | head -n 1) /workspace/app.jar


# --- [Option B: Maven (uncomment below and comment Option A if using Maven)] ---
# COPY mvnw pom.xml ./
# COPY .mvn/ ./.mvn/
# RUN chmod +x mvnw
# RUN --mount=type=cache,target=/root/.m2 ./mvnw dependency:go-offline -B
# COPY src/ ./src/
# RUN --mount=type=cache,target=/root/.m2 ./mvnw clean package -DskipTests
# RUN cp $(find target -name "*.jar" ! -name "*original*.jar" | head -n 1) /workspace/app.jar


# ==============================================================================
# Stage 2: Spring Boot Layer Extraction
# Deconstructs the fat JAR into layered folders for optimal Docker layer caching:
# dependencies -> spring-boot-loader -> snapshot-dependencies -> application code
# ==============================================================================
FROM eclipse-temurin:${JAVA_VERSION}-jre-${BASE_OS} AS extractor
WORKDIR /workspace

COPY --from=builder /workspace/app.jar app.jar

# Spring Boot 3.2+ / 4.x (uses 'tools' mode instead of legacy 'layertools'):
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted

# NOTE: If using Spring Boot 3.1 or earlier, use this command instead:
# RUN java -Djarmode=layertools -jar app.jar extract


# ==============================================================================
# Stage 3: Lean Production Runtime
# ==============================================================================
FROM eclipse-temurin:${JAVA_VERSION}-jre-${BASE_OS} AS runtime

ARG APP_USER=appuser
ARG APP_UID=10001
ARG PORT=8080

WORKDIR /app

# Install curl for robust healthchecking, and create dedicated non-root service user
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/* && \
    groupadd --system --gid ${APP_UID} ${APP_USER} && \
    useradd --system --uid ${APP_UID} --gid ${APP_USER} --no-create-home --shell /usr/sbin/nologin ${APP_USER}

# Copy extracted layers from extractor stage.
# Placed in order of change frequency: dependencies change rarely, application changes often.
COPY --from=extractor --chown=${APP_USER}:${APP_USER} /workspace/extracted/dependencies/ ./
COPY --from=extractor --chown=${APP_USER}:${APP_USER} /workspace/extracted/spring-boot-loader/ ./
COPY --from=extractor --chown=${APP_USER}:${APP_USER} /workspace/extracted/snapshot-dependencies/ ./
COPY --from=extractor --chown=${APP_USER}:${APP_USER} /workspace/extracted/application/ ./

# Switch to unprivileged user
USER ${APP_UID}:${APP_UID}

# Synchronize Spring Boot port with EXPOSE and healthcheck probe
ENV SERVER_PORT=${PORT}
EXPOSE ${PORT}

# JVM Tuning for Container Environments:
# - MaxRAMPercentage: automatically limits heap size to 75% of container RAM limit.
# - ExitOnOutOfMemoryError: fails fast on OOM so the orchestrator can restart the container cleanly.
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError"

# Healthcheck: Default targets standard Spring Boot Actuator endpoint.
# Uses shell form to expand ${SERVER_PORT} dynamically at runtime.
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
    CMD curl -f http://localhost:${SERVER_PORT}/actuator/health || exit 1

STOPSIGNAL SIGTERM

# Spring Boot 3.2+ / 4.x Entrypoint (direct execution without shell; PID 1 receives SIGTERM cleanly)
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]

# ------------------------------------------------------------------------------
# ALTERNATIVE ENTRYPOINTS:
#
# Spring Boot 3.1 or earlier:
# ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
#
# Non-Spring Boot application (Standard Fat JAR without layer extraction):
# In Stage 3, replace the layer COPY lines with:
#   COPY --from=builder --chown=${APP_USER}:${APP_USER} /workspace/app.jar /app/app.jar
# And use entrypoint:
#   ENTRYPOINT ["java", "-jar", "/app/app.jar"]
# ------------------------------------------------------------------------------
