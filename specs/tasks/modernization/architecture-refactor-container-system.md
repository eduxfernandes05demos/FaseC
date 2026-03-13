# Task: Architecture Refactor — Container System

> Containerize the engine: Dockerfile, health checks, graceful shutdown, config via env vars.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P1 — Cloud-Ready |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 2-3 weeks |
| **Dependencies** | Headless video, Headless audio/input, Build system (CMake) |
| **Blocks** | Session manager, Streaming gateway, Azure deployment |
| **Phase** | Phase 3 — Cloud-Ready |

---

## Objective

Package the headless Quake engine as a container image with HTTP health endpoints, structured JSON logging, graceful SIGTERM shutdown, and environment variable configuration.

---

## Implementation Steps

### Step 1: Create Dockerfile

**New file**: `Dockerfile`

```dockerfile
# Stage 1: Build environment
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y \
    gcc cmake make \
    && rm -rf /var/lib/apt/lists/*
COPY . /src/Quake
WORKDIR /src/Quake
RUN cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DHEADLESS_MODE=ON \
    -DUSE_ASM=OFF \
    -DHEALTH_ENDPOINTS=ON \
    -DSTRUCTURED_LOGGING=ON \
    && cmake --build build -j$(nproc)

# Stage 2: Runtime
FROM ubuntu:22.04 AS runtime
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -s /bin/bash -u 1000 quake
COPY --from=builder /src/legacy-src/build/quake-headless /usr/local/bin/
COPY --from=builder /src/legacy-src/build/qwsv /usr/local/bin/
USER quake
WORKDIR /home/quake
EXPOSE 26000/udp
EXPOSE 8080/tcp
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s \
    CMD curl -f http://localhost:8080/healthz || exit 1
ENTRYPOINT ["quake-headless"]
CMD ["-dedicated", "+map", "start"]
```

### Step 2: Create docker-compose.yml

**New file**: `docker-compose.yml`

```yaml
version: '3.8'
services:
  quake-server:
    build: .
    ports:
      - "26000:26000/udp"
      - "8080:8080/tcp"
    environment:
      - QUAKE_PORT=26000
      - QUAKE_GAMEDIR=id1
      - QUAKE_MEMORY=32
      - QUAKE_MAXPLAYERS=8
      - LOG_FORMAT=json
      - HEALTH_PORT=8080
    volumes:
      - ./data/id1:/home/quake/id1:ro
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '1.0'
```

### Step 3: Implement Health Endpoints

**New file**: `legacy-src/desktop-engine/health.c`

Lightweight HTTP server for Kubernetes health probes:

```c
#include "quakedef.h"
#include <sys/socket.h>
#include <netinet/in.h>

static int health_socket = -1;
static int health_port = 8080;
static qboolean engine_ready = false;

void Health_Init(void) {
    const char *port_env = getenv("HEALTH_PORT");
    if (port_env) health_port = atoi(port_env);

    health_socket = socket(AF_INET, SOCK_STREAM, 0);
    /* Bind and listen on health_port */
    /* Non-blocking accept in Health_Frame() */
}

void Health_Frame(void) {
    /* Accept connection, read HTTP request */
    /* Respond to:
     * GET /healthz  → 200 OK (always, if process is alive)
     * GET /readyz   → 200 OK (only after Host_Init completes)
     * GET /metrics  → Prometheus format metrics (optional)
     */
}

void Health_SetReady(qboolean ready) {
    engine_ready = ready;
}

void Health_Shutdown(void) {
    if (health_socket >= 0) close(health_socket);
}
```

**Integration point**: Call `Health_Frame()` from `Host_Frame()` at `host.c:729`

### Step 4: Implement Structured Logging

**Modified file**: `legacy-src/desktop-engine/console.c`

Replace `Con_Printf` output (line 360) with JSON structured logging:

```c
/* When STRUCTURED_LOGGING is enabled: */
void Con_Printf(char *fmt, ...) {
    /* ... existing code ... */
    if (structured_logging_enabled) {
        /* Output JSON to stdout */
        fprintf(stdout, "{\"timestamp\":\"%s\",\"level\":\"INFO\",\"message\":\"%s\"}\n",
                get_iso_timestamp(), sanitized_msg);
    } else {
        /* Original behavior */
        Sys_Printf("%s", msg);
    }
}
```

Log levels:
- `Con_Printf` → `INFO`
- `Con_DPrintf` (line 384) → `DEBUG`
- `Sys_Error` (`sys_linux.c:367`) → `ERROR`
- `Host_Error` → `ERROR`

### Step 5: Implement Graceful Shutdown

**Modified file**: `legacy-src/desktop-engine/host.c`

Add SIGTERM handler for clean container shutdown:

```c
#include <signal.h>

static volatile sig_atomic_t shutdown_requested = 0;

static void handle_sigterm(int sig) {
    (void)sig;
    shutdown_requested = 1;
}

/* In Host_Init (line 835): */
void Host_Init(quakeparms_t *parms) {
    signal(SIGTERM, handle_sigterm);
    signal(SIGINT, handle_sigterm);
    /* ... existing init ... */
    Health_SetReady(true);
}

/* In Host_Frame (line 729): */
void Host_Frame(float time) {
    if (shutdown_requested) {
        Con_Printf("Received shutdown signal, draining connections...\n");
        Host_ShutdownServer(true);  /* line 405 */
        Health_Shutdown();
        Sys_Quit();
    }
    Health_Frame();  /* Process health check requests */
    /* ... existing frame logic ... */
}
```

### Step 6: Implement Environment Variable Configuration

**Modified file**: `legacy-src/desktop-engine/host.c` (Host_Init, line 835)

Read configuration from environment variables before command-line parsing:

```c
void Host_Init(quakeparms_t *parms) {
    /* Read env vars */
    const char *env_port = getenv("QUAKE_PORT");
    if (env_port) {
        /* Override net_hostport (net_main.c:34 DEFAULTnet_hostport) */
        Cvar_Set("hostport", env_port);
    }

    const char *env_gamedir = getenv("QUAKE_GAMEDIR");
    /* Override default "id1" directory (common.c:1775) */

    const char *env_memory = getenv("QUAKE_MEMORY");
    /* Override memory allocation size */

    const char *env_maxplayers = getenv("QUAKE_MAXPLAYERS");
    if (env_maxplayers) {
        Cvar_Set("maxplayers", env_maxplayers);
    }
}
```

---

## Files Affected

| File | Change Type | Description |
|------|------------|-------------|
| `Dockerfile` | **New** | Multi-stage container build |
| `docker-compose.yml` | **New** | Local development orchestration |
| `.dockerignore` | **New** | Exclude unnecessary files from build context |
| `legacy-src/desktop-engine/health.c` | **New** | HTTP health endpoint server |
| `legacy-src/desktop-engine/health.h` | **New** | Health module header |
| `legacy-src/desktop-engine/host.c` | **Edit** | SIGTERM handler (line 729, 835), env var config, health integration |
| `legacy-src/desktop-engine/console.c` | **Edit** | Structured JSON logging (line 360, 384) |
| `legacy-src/desktop-engine/net_main.c` | **Edit** | Environment variable for port (line 34) |
| `legacy-src/desktop-engine/common.c` | **Edit** | Environment variable for game directory (line 1775) |
| `legacy-src/desktop-engine/CMakeLists.txt` | **Edit** | Add health.c, container build options |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_docker_build` | `docker build .` succeeds |
| `test_docker_run` | Container starts and health endpoint responds |
| `test_health_liveness` | `GET /healthz` returns 200 |
| `test_health_readiness` | `GET /readyz` returns 200 after init, 503 before |
| `test_structured_logging` | Container logs are valid JSON |
| `test_sigterm` | `docker stop` causes clean exit (exit code 0) within 30s |
| `test_env_port` | `QUAKE_PORT=27000` starts server on port 27000 |
| `test_env_gamedir` | `QUAKE_GAMEDIR=custom` loads from custom directory |
| `test_env_memory` | `QUAKE_MEMORY=32` allocates 32 MB |
| `test_stability_24h` | Container runs 24 hours without crash or memory leak |

---

## Acceptance Criteria

- [ ] `docker build` produces image ≤ 200 MB
- [ ] Container starts in headless mode and listens on UDP port
- [ ] `GET /healthz` returns HTTP 200
- [ ] `GET /readyz` returns HTTP 200 after initialization
- [ ] Container logs are structured JSON to stdout
- [ ] `docker stop` causes graceful shutdown (exit 0) within 30 seconds
- [ ] All configuration controllable via environment variables
- [ ] Container runs as non-root user (uid 1000)
- [ ] Container stable for 24-hour run without crashes or leaks
- [ ] QW client can connect to containerized server
