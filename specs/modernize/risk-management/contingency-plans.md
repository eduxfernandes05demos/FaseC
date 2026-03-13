# Contingency Plans — Quake Engine Modernization

> Emergency procedures and fallbacks for critical failure scenarios.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document provides contingency plans for the most severe failure scenarios during modernization. Each contingency is a pre-planned response that can be executed immediately when a failure condition is detected, minimizing downtime and data loss.

---

## 2. Contingency Matrix

| ID | Scenario | Severity | Trigger | Response Time |
|----|----------|----------|---------|---------------|
| C1 | Container networking failure | Critical | No UDP traffic | < 15 minutes |
| C2 | Rendering breaks in headless mode | High | Black/corrupt frames | < 1 hour |
| C3 | Security fix breaks compatibility | High | Client connection failure | < 30 minutes |
| C4 | Memory exhaustion in container | High | OOMKill events | < 15 minutes |
| C5 | Streaming latency unacceptable | Medium | Latency > 100ms | < 4 hours |
| C6 | Azure infrastructure failure | Critical | Service unavailable | < 30 minutes |
| C7 | Build system migration failure | Medium | Cannot compile | < 2 hours |
| C8 | Data loss or corruption | Critical | Missing/corrupt game state | < 1 hour |

---

## 3. C1: Container Networking Failure

### Scenario

Containerized Quake engine cannot receive or send UDP game traffic. Kubernetes Service or Network Policy blocks game packets on port 26000 (defined at `legacy-src/desktop-engine/net_main.c:34`).

### Detection

- Health endpoint responsive but no player connections
- `tcpdump` shows no UDP traffic to container
- Client reports "Connection timed out"

### Response

```bash
# Step 1: Verify network policy
kubectl get networkpolicy -n quake
kubectl describe networkpolicy quake-engine-policy -n quake

# Step 2: Check service configuration
kubectl get svc quake-engine -n quake -o yaml
# Verify: type: LoadBalancer and port: 26000/UDP

# Step 3: Quick fix — expose directly
kubectl delete svc quake-engine -n quake
kubectl expose deployment quake-engine -n quake \
  --type=LoadBalancer --port=26000 --protocol=UDP

# Step 4: If still failing — use hostNetwork
# Edit deployment to use hostNetwork: true
kubectl patch deployment quake-engine -n quake \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true}}}}'
```

### Long-term Fix

- Use Azure Application Gateway with UDP support
- Configure proper Kubernetes Service with `externalTrafficPolicy: Local`
- Test UDP connectivity in staging before production promotion

---

## 4. C2: Rendering Breaks in Headless Mode

### Scenario

Headless video driver (`vid_headless.c`) produces black, corrupt, or incorrect frames. Server simulation may be correct but frame extraction fails.

### Detection

- Streaming gateway receives frames with all-black or garbled pixels
- Visual comparison with reference images fails
- Frame extraction returns incorrect dimensions

### Response

```bash
# Step 1: Verify software renderer works outside container
./quake-headless -timedemo demo1 -framebuffer-dump /tmp/frames/
# Check: ls /tmp/frames/ — should contain frame images

# Step 2: If frames are black — check renderer initialization
# Verify VID_Init allocates framebuffer correctly
# Check vid.buffer pointer is valid in r_main.c:1049 (R_RenderView)

# Step 3: Fallback to original renderer with X11 forwarding
docker run -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix \
  quake-engine:previous-version

# Step 4: If headless rendering is broken, use screenshot approach
# Modify screen.c to capture frames via SCR_ScreenShot mechanism
```

### Root Cause Investigation

1. Check `VID_AllocBuffers()` equivalent in headless driver — must allocate Z-buffer and surface cache
2. Verify `vid.buffer` points to valid writeable memory
3. Check `vid.width` and `vid.height` are set correctly
4. Verify `d_pzbuffer` (Z-buffer) is allocated — referenced in rendering pipeline

### Long-term Fix

- Add frame validation test: render known scene, compare against golden reference
- Add assertions in headless driver for buffer validity
- Integration test that extracts frames and validates non-black content

---

## 5. C3: Security Fix Breaks Compatibility

### Scenario

Replacing `sprintf`/`strcpy` in network code changes packet format or truncates data that must match exactly. QuakeWorld clients cannot connect to modernized server.

### Detection

- Client reports "Connection refused" or "Bad server message"
- Network protocol tests fail
- Packet capture shows different byte sequences

### Response

```bash
# Step 1: Identify which commit broke compatibility
git bisect start
git bisect bad HEAD
git bisect good <last-known-good-commit>
# Run: ./test_network_compat.sh
# git bisect will find the breaking commit

# Step 2: Revert the breaking commit
git revert <breaking-commit>
cmake --build build
ctest --output-on-failure

# Step 3: Fix the specific replacement
# Common issue: snprintf truncates data that must be exact length
# Fix: Use exact-length buffers or validate snprintf return value
```

### Common Fixes

| Issue | File | Fix |
|-------|------|-----|
| Address string truncated | `net_dgrm.c:132-134` | Ensure buffer ≥ 22 chars for IP:port |
| Host name truncated | `net_dgrm.c:558` | Ensure buffer matches `MAX_OSPATH` |
| Entity field truncated | `sv_ents.c` | Validate delta encoding byte-exact |
| Command string truncated | `cl_main.c:328-336` | Ensure RCON buffer sufficient |

### Long-term Fix

- Add protocol compatibility test: modernized server ↔ original client
- Add network packet golden tests with known byte sequences
- Document exact buffer size requirements for each network operation

---

## 6. C4: Memory Exhaustion in Container

### Scenario

Engine's Zone/Hunk allocator requests more memory than container limit allows. Container is OOMKilled by Kubernetes.

### Detection

- Pod status: `OOMKilled`
- `kubectl describe pod` shows memory limit exceeded
- Engine log shows "Not enough memory" from `zone.c`

### Response

```bash
# Step 1: Increase container memory limit
kubectl patch deployment quake-engine -n quake \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"quake","resources":{"limits":{"memory":"256Mi"}}}]}}}}'

# Step 2: Configure engine memory to fit container
kubectl set env deployment/quake-engine QUAKE_MEMORY=32

# Step 3: If still OOMKill, check for memory leaks
# Build with AddressSanitizer
cmake -B build -DCMAKE_C_FLAGS="-fsanitize=address" && cmake --build build
./quake-headless -dedicated -leak-check
```

### Memory Sizing Guide

| Container Limit | QUAKE_MEMORY | Overhead | Use Case |
|----------------|-------------|----------|----------|
| 64 MB | 32 MB | 32 MB OS/runtime | Minimal server |
| 128 MB | 64 MB | 64 MB | Standard server |
| 256 MB | 128 MB | 128 MB | Server + encoding |
| 512 MB | 256 MB | 256 MB | Full streaming |

### Long-term Fix

- Implement cgroup-aware memory auto-configuration
- Add memory usage monitoring and alerting
- Replace Zone allocator with standard `malloc` for better container integration

---

## 7. C5: Streaming Latency Unacceptable

### Scenario

End-to-end streaming latency exceeds 100ms, making gameplay feel unresponsive.

### Detection

- Client-side latency measurement exceeds threshold
- User reports "laggy" or "unresponsive" gameplay
- Frame timing metrics show encoding bottleneck

### Response

```bash
# Step 1: Identify bottleneck component
# Measure each stage independently:
echo "Frame render: $(./measure render_time)ms"
echo "Frame extract: $(./measure extract_time)ms"
echo "Encode: $(./measure encode_time)ms"
echo "Network: $(./measure network_time)ms"

# Step 2: Quick fix — reduce quality
# Lower resolution
kubectl set env deployment/quake-engine QUAKE_WIDTH=320 QUAKE_HEIGHT=240

# Lower framerate
kubectl set env deployment/quake-engine QUAKE_FPS=30

# Step 3: Switch to faster encoding preset
kubectl set env deployment/quake-gateway ENCODE_PRESET=ultrafast

# Step 4: If encoding is bottleneck — use hardware encoding
# Deploy to nodes with GPU
kubectl label nodes gpu-node-1 gpu=true
# Add nodeSelector: { gpu: "true" } to deployment
```

### Latency Budget Breakdown

| Component | Target | Acceptable | Critical |
|-----------|--------|-----------|----------|
| Frame render | 5ms | 10ms | 16ms |
| Frame extract | 1ms | 2ms | 5ms |
| Video encode | 5ms | 10ms | 20ms |
| Network transit | 10ms | 20ms | 50ms |
| Client decode | 5ms | 10ms | 20ms |
| **Total** | **26ms** | **52ms** | **111ms** |

### Long-term Fix

- Implement adaptive streaming: dynamically adjust resolution/FPS based on latency
- Use WebRTC for lower-latency delivery than WebSocket
- Deploy encoder on GPU-equipped nodes
- Pre-encode static elements, only encode dynamic changes

---

## 8. C6: Azure Infrastructure Failure

### Scenario

AKS cluster, ACR, or supporting Azure services become unavailable.

### Detection

- `kubectl` commands fail
- Azure Portal shows service degradation
- Health monitoring alerts fire

### Response

```bash
# Step 1: Check Azure status
az aks show -g quake-rg -n quake-aks --query "powerState"

# Step 2: If AKS down — failover to backup region
az aks get-credentials -g quake-rg-backup -n quake-aks-backup
kubectl apply -f k8s/production/

# Step 3: Update DNS to backup cluster
az network dns record-set a update -g quake-rg \
  -z quake.example.com -n @ \
  --set aRecords[0].ipv4Address=<backup-cluster-ip>

# Step 4: If no backup cluster — run on VM
az vm create -g quake-rg -n quake-emergency \
  --image Ubuntu2204 --size Standard_D4s_v3
# SSH in and run directly
scp quake-headless user@<vm-ip>:/opt/quake/
ssh user@<vm-ip> "cd /opt/quake && ./quake-headless -dedicated"
```

### Long-term Fix

- Multi-region deployment with automatic failover
- Azure Traffic Manager for DNS-based failover
- Regular disaster recovery drills (quarterly)
- Infrastructure recreation from IaC (Bicep) in < 30 minutes

---

## 9. C7: Build System Migration Failure

### Scenario

CMake migration is incomplete or incorrect — cannot compile certain targets.

### Detection

- `cmake --build .` fails with missing source files or wrong compiler flags
- Compiled binary behaves differently from Makefile build
- CI fails to build

### Response

```bash
# Step 1: Fall back to original build system
cd legacy-src/desktop-engine
make -f Makefile.linuxi386 clean
make -f Makefile.linuxi386

# Step 2: Compare with CMake build to find differences
diff <(nm quake-makefile | sort) <(nm quake-cmake | sort)

# Step 3: Fix CMake configuration based on differences
# Common issues:
# - Missing .c files in CMake target
# - Wrong compiler flags (optimization level, defines)
# - Missing library links
```

### Long-term Fix

- Automate CMake validation: compare object files with Makefile build
- Keep original Makefiles as reference until CMake is fully validated
- Add CI job that builds with both systems and compares output

---

## 10. C8: Data Loss or Corruption

### Scenario

Game save data, demo recordings, or configuration are lost or corrupted during migration.

### Detection

- Players report lost save games
- Demo playback fails
- Configuration resets to defaults

### Response

```bash
# Step 1: Check persistent volume
kubectl get pvc -n quake
kubectl describe pvc quake-data -n quake

# Step 2: Restore from backup
az storage blob download -c quake-backups -n latest-backup.tar.gz \
  --account-name quakebackups -f /tmp/restore.tar.gz
tar xzf /tmp/restore.tar.gz -C /mnt/quake-data/

# Step 3: Restart pods with restored data
kubectl rollout restart deployment quake-engine -n quake
```

### Prevention

- Azure Blob Storage versioning enabled
- Hourly backups of persistent volumes
- File integrity checks (checksums) on save/load operations
- Separate persistent volume for game data (not ephemeral container storage)

---

## 11. Contingency Communication Plan

| Severity | Notification | Channel | Audience |
|----------|-------------|---------|----------|
| Critical | Immediate | Slack + Email + PagerDuty | On-call team |
| High | Within 15 min | Slack + Email | Engineering team |
| Medium | Within 1 hour | Slack | Engineering team |
| Low | Next standup | Standup meeting | Engineering team |

### Incident Response Template

```
## Incident Report
- **Time**: [Detection time]
- **Severity**: [Critical/High/Medium/Low]
- **Impact**: [What is affected]
- **Trigger**: [What condition was detected]
- **Response**: [Actions taken]
- **Root Cause**: [After investigation]
- **Prevention**: [What will prevent recurrence]
- **Status**: [Ongoing/Resolved]
```
