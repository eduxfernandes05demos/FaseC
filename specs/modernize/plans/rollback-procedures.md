# Rollback Procedures — Quake Engine Modernization

> Rollback and contingency plans: how to revert each migration step if something fails.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

Each modernization step is designed to be independently reversible. This document provides step-by-step rollback procedures for every phase of the migration, including trigger conditions, rollback steps, and validation after rollback.

---

## 2. Rollback Principles

1. **Every change is reversible**: No migration step burns bridges
2. **Feature flags before removal**: Old code is disabled, not deleted, until the replacement is proven
3. **Git-based rollback**: All changes tracked in version control with clear commit boundaries
4. **Blue-green for deployment**: Both old and new versions run simultaneously during transition
5. **Data compatibility**: Schema changes are backward-compatible

---

## 3. Phase 0 — Build System Rollback

### Trigger

- CMake builds produce incorrect binaries
- CMake builds missing source files or flags
- CI pipeline unreliable

### Rollback Steps

```bash
# 1. Restore original build files
git checkout main -- Quake/WinQuake/Makefile.linuxi386
git checkout main -- Quake/QW/Makefile.Linux
git checkout main -- Quake/WinQuake/WinQuake.dsp

# 2. Build with original Makefiles
cd Quake/WinQuake && make -f Makefile.linuxi386

# 3. Validate binary output matches known-good
md5sum quake && diff <(md5sum quake) known_good.md5
```

### Mitigation

- Original build files preserved in `Quake/legacy-build/` directory
- Both build systems coexist during transition
- CI runs both builds in parallel for comparison

### Validation After Rollback

- [ ] Original Makefile build succeeds
- [ ] Binary output matches pre-migration checksum
- [ ] Engine runs and completes timedemo

---

## 4. Phase 1 — Security Remediation Rollback

### Trigger

- `snprintf` replacement changes output behavior (truncation where `sprintf` didn't truncate)
- `strncpy` introduces bugs (missing null termination)
- Performance regression from bounds checking

### Rollback Steps

```bash
# 1. Revert security commits (each sub-step is a separate commit)
git revert <security-sprintf-commit>
git revert <security-strcpy-commit>
git revert <security-strcat-commit>

# 2. Or granular per-file revert
git checkout main -- Quake/WinQuake/common.c
git checkout main -- Quake/WinQuake/net_dgrm.c
# ... etc for each affected file

# 3. Rebuild
cmake --build build

# 4. Run tests
cd build && ctest --output-on-failure
```

### Mitigation

- Security fixes applied in small, independent commits (one per file or function group)
- Each commit message references the specific unsafe pattern replaced
- Regression tests run after each file change

### Validation After Rollback

- [ ] Engine compiles without errors
- [ ] Timedemo runs without crashes
- [ ] Network play works (client connects to server)
- [ ] Console commands execute correctly

---

## 5. Phase 2 — Modularization Rollback

### Trigger

- Platform abstraction breaks existing functionality
- Headless mode doesn't simulate correctly
- Interface overhead causes performance regression
- Renderer modularization breaks visual output

### Rollback Steps

#### 5a. Platform Abstraction Rollback

```bash
# Revert interface headers and wrappers
git revert <platform-abstraction-commit>

# Restore direct function calls
# Build with original platform selection (compile-time)
cmake -B build -DUSE_INTERFACES=OFF
cmake --build build
```

#### 5b. Headless Mode Rollback

```bash
# Remove headless backends
git rm Quake/WinQuake/vid_headless.c
git rm Quake/WinQuake/sys_headless.c

# Build with original video driver
cmake -B build -DHEADLESS=OFF
cmake --build build
```

#### 5c. Renderer Modularization Rollback

```bash
# Revert renderer interface
git revert <renderer-interface-commit>

# Build with compile-time renderer selection (original behavior)
cmake -B build -DRENDERER_INTERFACE=OFF
cmake --build build
```

### Mitigation

- Old code preserved behind `#ifdef USE_INTERFACES` / `#ifndef USE_INTERFACES`
- CMake options control new vs. old code paths
- Feature flags allow gradual rollout

### Validation After Rollback

- [ ] Engine starts with original video driver
- [ ] Rendering output matches pre-modularization
- [ ] Sound plays correctly
- [ ] Input works (keyboard + mouse)
- [ ] Timedemo FPS matches baseline (±5%)

---

## 6. Phase 3 — Containerization Rollback

### Trigger

- Container fails to start
- Health endpoints not functioning
- Graceful shutdown doesn't work
- Performance degradation in container

### Rollback Steps

```bash
# 1. Stop container deployment
docker-compose down
# or
kubectl delete deployment quake-engine

# 2. Run engine directly on host
./quake-headless -dedicated +map start

# 3. If structured logging causes issues, revert
git revert <structured-logging-commit>

# 4. If SIGTERM handling causes issues, revert
git revert <graceful-shutdown-commit>
```

### Mitigation

- Container image tagged with version; can roll back to any previous image
- Kubernetes deployment supports `kubectl rollout undo`
- Engine can always run outside container

### Validation After Rollback

- [ ] Engine runs directly on host
- [ ] Original logging format works
- [ ] Server accepts connections
- [ ] No memory leaks after 24h run

---

## 7. Phase 4 — Cloud Deployment Rollback

### Trigger

- Azure deployment fails
- Streaming pipeline not functional
- Session manager loses state
- Unacceptable latency or cost

### Rollback Steps

#### 7a. Streaming Gateway Rollback

```bash
# Scale down gateway, route traffic to direct connections
kubectl scale deployment quake-gateway --replicas=0

# Clients connect directly to engine containers
kubectl expose deployment quake-engine --type=LoadBalancer --port=26000
```

#### 7b. Session Manager Rollback

```bash
# Scale down session manager, manage sessions manually
kubectl scale deployment quake-session-mgr --replicas=0

# Deploy fixed number of engine pods
kubectl scale deployment quake-engine --replicas=4
```

#### 7c. Full Azure Rollback

```bash
# 1. Remove Kubernetes deployments
kubectl delete -f k8s/production/

# 2. Optionally destroy Azure resources
az deployment group delete -g quake-rg -n quake-deployment

# 3. Run engine on traditional VMs or bare metal
scp quake-headless user@server:/opt/quake/
ssh user@server "cd /opt/quake && ./quake-headless -dedicated"
```

### Mitigation

- Infrastructure as Code allows recreation from scratch
- Container images preserved in ACR
- Game data in Azure Blob Storage has versioning enabled
- DNS can be pointed to alternative infrastructure

### Validation After Rollback

- [ ] Game server accessible from clients
- [ ] Players can connect and play
- [ ] No data loss (save games, recordings preserved)

---

## 8. Emergency Procedures

### 8.1 Complete System Failure

If the entire modernized stack fails:

1. **Immediate**: Route DNS to backup traditional server
2. **Short-term**: Deploy original engine binary on VM
3. **Medium-term**: Diagnose and fix modernized stack
4. **Long-term**: Implement missing monitoring/alerting

### 8.2 Security Incident

1. **Isolate**: Remove affected containers from network (`kubectl cordon`)
2. **Assess**: Check logs for breach indicators
3. **Contain**: Rotate all credentials, revoke sessions
4. **Recover**: Deploy patched images
5. **Review**: Post-incident analysis and remediation

### 8.3 Data Corruption

1. **Stop**: Halt all write operations
2. **Backup**: Snapshot current state before recovery
3. **Restore**: Restore from last known-good backup
4. **Validate**: Run integrity checks on restored data
5. **Resume**: Restart services after validation

---

## 9. Rollback Testing Schedule

| Test | Frequency | Scope |
|------|-----------|-------|
| Build system rollback | Once (after migration) | Verify original Makefiles still work |
| Container rollback | Monthly | Test `kubectl rollout undo` |
| Azure failover | Quarterly | Full infrastructure recreation |
| Security incident drill | Bi-annually | Simulate breach response |
