# Testing Strategy — Quake Engine Modernization

> Comprehensive testing approach: unit tests, integration tests, performance benchmarks, regression suite.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

The Quake codebase currently has **zero automated tests**. This strategy establishes a comprehensive testing framework covering unit tests, integration tests, performance benchmarks, and regression testing. Testing is critical because every modernization change risks breaking the engine's functionality.

---

## 2. Test Framework Selection

| Level | Framework | Rationale |
|-------|-----------|-----------|
| Unit | **Unity** (ThrowTheSwitch) | Lightweight C test framework, single-header, CMake compatible |
| Integration | **Custom harness** + Docker | Engine subsystem interaction tests |
| E2E | **pytest** + WebSocket client | Full streaming pipeline validation |
| Performance | **Custom benchmark** + CTest | Frame timing, latency measurement |
| Fuzzing | **AFL++** + **libFuzzer** | Protocol and input fuzzing |
| Memory | **Valgrind** + **ASan** | Memory safety validation |

---

## 3. Unit Tests

### 3.1 Math Library (`legacy-src/desktop-engine/mathlib.c`)

| Test | Function | Validation |
|------|----------|-----------|
| Vector addition | `VectorAdd()` | Known result vectors |
| Vector subtraction | `VectorSubtract()` | Known result vectors |
| Dot product | `DotProduct()` | Scalar result verification |
| Cross product | `CrossProduct()` | Orthogonality and magnitude |
| Vector normalize | `VectorNormalize()` | Unit length result |
| Angle vectors | `AngleVectors()` | Forward/right/up from angles |
| Box on plane side | `BoxOnPlaneSide()` | Correct classification |
| Length calculation | `Length()` | Known distance results |

### 3.2 Common Utilities (`legacy-src/desktop-engine/common.c`)

| Test | Function/Area | Validation |
|------|--------------|-----------|
| String comparison | `Q_strcmp()` at line 188 | Case sensitivity, empty strings |
| String copy safety | `Q_strlcpy()` (new) | Buffer overflow prevention |
| String concat safety | `Q_strlcat()` (new) | Buffer overflow prevention |
| Byte ordering | `BigShort()`, `BigLong()`, etc. | Endian conversion correctness |
| Message parsing | `MSG_ReadByte()`, `MSG_ReadShort()` | Correct byte extraction |
| Path handling | `COM_FileBase()`, `COM_DefaultExtension()` | Path manipulation correctness |
| Path sanitization | New path validation functions | Reject `../`, null bytes |
| Argument parsing | `COM_CheckParm()` | Finds arguments correctly |

### 3.3 Console Variables (`legacy-src/desktop-engine/cvar.c`)

| Test | Function | Validation |
|------|----------|-----------|
| Registration | `Cvar_RegisterVariable()` | Cvar appears in list |
| Set value | `Cvar_Set()` | String and float values update |
| Get value | `Cvar_VariableValue()` | Returns correct float |
| Get string | `Cvar_VariableString()` | Returns correct string |
| Find by name | `Cvar_FindVar()` | Finds registered cvars |
| Command integration | `Cvar_Command()` | Console command sets cvar |

### 3.4 Memory Allocator (`legacy-src/desktop-engine/zone.c`)

| Test | Function | Validation |
|------|----------|-----------|
| Zone allocate/free | `Z_Malloc()` at line 142, `Z_Free()` at line 99 | Alloc returns non-NULL, free succeeds |
| Zone overflow | `Z_Malloc()` with oversized request | Graceful error handling |
| Hunk allocate | `Hunk_AllocName()` at line 399 | Returns aligned memory |
| Hunk high/low | `Hunk_HighAllocName()` | Non-overlapping allocations |
| Cache allocate/evict | `Cache_Alloc()` at line 871 | LRU eviction works correctly |
| Heap integrity | `Z_CheckHeap()` at line 247 | No corruption after operations |

### 3.5 Command System (`legacy-src/desktop-engine/cmd.c`)

| Test | Function | Validation |
|------|----------|-----------|
| Command registration | `Cmd_AddCommand()` | Command appears in list |
| Command execution | `Cmd_ExecuteString()` | Callback invoked |
| Argument parsing | `Cmd_Argc()`, `Cmd_Argv()` | Correct tokenization |
| Command aliasing | `Cmd_Alias_f()` | Alias executes correctly |
| Buffer execution | `Cbuf_Execute()` | Commands execute in order |

### 3.6 Network Protocol

| Test | Source | Validation |
|------|--------|-----------|
| Packet encoding | `legacy-src/QW/server/sv_ents.c:155` (`SV_WriteDelta`) | Correct bitmask encoding |
| Packet decoding | `legacy-src/QW/client/cl_ents.c:265` (`CL_ParsePacketEntities`) | Correct field extraction |
| Sequence numbers | `legacy-src/desktop-engine/net_dgrm.c` | Ordering and ACK correctness |
| Message overflow | Network buffer limits | Graceful rejection of oversized messages |

---

## 4. Integration Tests

### 4.1 Server-Client Communication

| Test | Description | Components |
|------|-------------|-----------|
| Connection handshake | Client connects to server | `net_main.c` + `cl_main.c` + `sv_main.c` |
| Entity synchronization | Entity state transfers correctly | `sv_ents.c` + `cl_ents.c` |
| Disconnection cleanup | Clean disconnect frees resources | `net_main.c` + `host.c` |
| Multi-client | Multiple clients connect simultaneously | Full network stack |

### 4.2 QuakeC VM

| Test | Description | Source |
|------|-------------|--------|
| Opcode execution | All 80 opcodes execute correctly | `legacy-src/desktop-engine/pr_exec.c` |
| Built-in functions | Engine built-ins return correct results | `legacy-src/desktop-engine/pr_cmds.c` |
| Entity spawn/remove | Entity lifecycle works | `legacy-src/desktop-engine/pr_edict.c` |
| Think/touch callbacks | Entity callbacks fire at correct times | `sv_phys.c` + `pr_exec.c` |

### 4.3 Configuration & Startup

| Test | Description |
|------|-------------|
| Headless startup | Engine starts with `--backend=headless` |
| Environment config | Engine reads `QUAKE_PORT`, `QUAKE_GAMEDIR` from env |
| Config file loading | `config.cfg` values apply correctly |
| PAK file loading | `pak0.pak` loads and provides game data |

---

## 5. Performance Benchmarks

### 5.1 Rendering Benchmarks

| Benchmark | Metric | Baseline | Target |
|-----------|--------|----------|--------|
| Frame render time | ms/frame | Establish | No regression > 5% |
| Timedemo throughput | FPS | Establish | No regression > 5% |
| BSP traversal time | μs/frame | Establish | No regression > 10% |

**Method**: Run `timedemo demo1` and measure frame times.

### 5.2 Network Benchmarks

| Benchmark | Metric | Target |
|-----------|--------|--------|
| Packet encode time | μs/packet | < 100μs |
| Packet decode time | μs/packet | < 100μs |
| Round-trip latency | ms | < 5ms (localhost) |
| Concurrent clients | count | 16+ stable |

### 5.3 Memory Benchmarks

| Benchmark | Metric | Target |
|-----------|--------|--------|
| Zone allocation speed | ns/alloc | < 1000ns |
| Hunk allocation speed | ns/alloc | < 100ns |
| Peak memory usage | MB | Establish baseline |
| Memory leak detection | bytes leaked | 0 |

### 5.4 Streaming Benchmarks

| Benchmark | Metric | Target |
|-----------|--------|--------|
| Frame extraction latency | ms | < 2ms |
| Video encoding latency | ms | < 16ms (60 FPS budget) |
| End-to-end stream latency | ms | < 50ms |
| Concurrent streams | count | 10+ on 8-core machine |

---

## 6. Regression Testing

### 6.1 Timedemo Regression

Run the standard Quake timedemo benchmark after every change:

```bash
# Baseline
./quake-headless -timedemo demo1 > baseline.txt

# After changes
./quake-headless -timedemo demo1 > current.txt

# Compare (fail if >5% regression)
python compare_timedemo.py baseline.txt current.txt --threshold 5
```

### 6.2 Network Regression

Automated client-server test that validates:
- Connection establishment
- Player movement synchronization
- Entity state consistency
- Clean disconnection

### 6.3 Visual Regression (Future)

Compare rendered frame output:
- Render specific scenes to framebuffer
- Compare against golden reference images (pixel diff)
- Flag regressions above threshold

---

## 7. Test Infrastructure

### 7.1 Directory Structure

```
legacy-src/tests/
├── CMakeLists.txt          # Test build configuration
├── unity/                  # Unity test framework (vendored)
├── unit/
│   ├── test_mathlib.c      # Math library tests
│   ├── test_common.c       # Common utilities tests
│   ├── test_cvar.c         # Console variable tests
│   ├── test_zone.c         # Memory allocator tests
│   ├── test_cmd.c          # Command system tests
│   ├── test_network.c      # Network protocol tests
│   └── test_security.c     # Security validation tests
├── integration/
│   ├── test_server_client.c
│   ├── test_quakec_vm.c
│   └── test_startup.c
├── benchmark/
│   ├── bench_render.c
│   ├── bench_network.c
│   └── bench_memory.c
└── fixtures/
    ├── demo1.dem           # Timedemo replay file
    └── test_config.cfg     # Test configuration
```

### 7.2 CI Integration

```yaml
# In .github/workflows/ci.yml
- name: Run unit tests
  run: cd build && ctest -L unit --output-on-failure

- name: Run integration tests
  run: cd build && ctest -L integration --output-on-failure

- name: Run benchmarks
  run: cd build && ctest -L benchmark --output-on-failure

- name: Check for regressions
  run: cd build && ctest -L regression --output-on-failure
```

---

## 8. Test Coverage Goals

| Phase | Unit Tests | Integration | Benchmarks | Coverage Target |
|-------|-----------|-------------|-----------|----------------|
| Phase 0 (Foundation) | 10+ | 0 | 0 | 5% |
| Phase 1 (Security) | 30+ | 5+ | 0 | 15% |
| Phase 2 (Modular) | 60+ | 15+ | 5+ | 30% |
| Phase 3 (Container) | 80+ | 25+ | 10+ | 40% |
| Phase 4 (Cloud) | 100+ | 30+ | 15+ | 50% |
