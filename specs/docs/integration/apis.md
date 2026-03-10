# Internal APIs & Module Interfaces — Quake Engine

> Reverse-engineered from the id Software Quake source code (1996-1997).
> All file references point to actual source files in this repository.

---

## 1. Overview

The Quake engine communicates between subsystems through well-defined C function interfaces declared in header files. There are no formal API versioning or interface contracts — the interfaces are defined by convention through header file declarations.

---

## 2. System Interface (`Quake/WinQuake/sys.h`)

Platform abstraction layer — every platform implements these functions.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Sys_FileOpenRead` | `int Sys_FileOpenRead(char *path, int *hndl)` | Open file for reading, return size |
| `Sys_FileOpenWrite` | `int Sys_FileOpenWrite(char *path)` | Open file for writing |
| `Sys_FileClose` | `void Sys_FileClose(int handle)` | Close file handle |
| `Sys_FileRead` | `void Sys_FileRead(int handle, void *dest, int count)` | Read bytes from file |
| `Sys_FileWrite` | `void Sys_FileWrite(int handle, void *data, int count)` | Write bytes to file |
| `Sys_FileTime` | `int Sys_FileTime(char *path)` | Get file modification time |
| `Sys_mkdir` | `void Sys_mkdir(char *path)` | Create directory |
| `Sys_FloatTime` | `double Sys_FloatTime(void)` | High-resolution time in seconds |
| `Sys_Error` | `void Sys_Error(char *error, ...)` | Fatal error — shows dialog/message, exits |
| `Sys_Printf` | `void Sys_Printf(char *fmt, ...)` | Print to system console |
| `Sys_Quit` | `void Sys_Quit(void)` | Clean shutdown |
| `Sys_SendKeyEvents` | `void Sys_SendKeyEvents(void)` | Pump platform input events into engine |
| `Sys_ConsoleInput` | `char *Sys_ConsoleInput(void)` | Read line from stdin (dedicated server) |
| `Sys_MakeCodeWriteable` | `void Sys_MakeCodeWriteable(unsigned long startaddr, unsigned long length)` | Make memory region executable (for self-modifying assembly) |
| `Sys_DebugLog` | `void Sys_DebugLog(char *file, char *fmt, ...)` | Write to debug log file |

**Implementations**: `sys_win.c`, `sys_linux.c`, `sys_dos.c`, `sys_sun.c`, `sys_null.c`

---

## 3. Video Interface (`Quake/WinQuake/vid.h`)

Display initialization and framebuffer management.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `VID_Init` | `void VID_Init(unsigned char *palette)` | Initialize video subsystem with color palette |
| `VID_Shutdown` | `void VID_Shutdown(void)` | Shutdown video, restore display mode |
| `VID_SetMode` | `int VID_SetMode(int modenum, unsigned char *palette)` | Switch to specific video mode |
| `VID_Update` | `void VID_Update(vrect_t *rects)` | Copy framebuffer to screen (dirty rect list) |
| `VID_SetPalette` | `void VID_SetPalette(unsigned char *palette)` | Set 256-color palette |
| `VID_ShiftPalette` | `void VID_ShiftPalette(unsigned char *palette)` | Palette shift for damage/underwater effects |
| `VID_HandlePause` | `void VID_HandlePause(qboolean pause)` | Handle pause state (show/hide cursor) |

**Key data structure — `viddef_t`**:
| Field | Type | Purpose |
|-------|------|---------|
| `buffer` | `pixel_t *` | Main framebuffer |
| `colormap` | `pixel_t *` | Palette-based color lookup table |
| `rowbytes` | `unsigned` | Bytes per scanline |
| `width`, `height` | `unsigned` | Current resolution |
| `aspect` | `float` | Pixel aspect ratio |
| `numpages` | `int` | Double/triple buffering page count |

**Implementations**: `vid_win.c`, `vid_dos.c`, `vid_svgalib.c`, `vid_x.c`, `vid_sunx.c`, `vid_sunxil.c`, `vid_null.c`

---

## 4. Input Interface (`Quake/WinQuake/input.h`)

Input device abstraction.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `IN_Init` | `void IN_Init(void)` | Initialize input devices |
| `IN_Shutdown` | `void IN_Shutdown(void)` | Shutdown input devices |
| `IN_Commands` | `void IN_Commands(void)` | Generate button events from device state |
| `IN_Move` | `void IN_Move(usercmd_t *cmd)` | Read mouse/joystick and apply to movement command |
| `IN_ClearStates` | `void IN_ClearStates(void)` | Reset all input state |

**Implementations**: `in_win.c`, `in_dos.c`, `in_sun.c`, `in_null.c`

---

## 5. Sound Interface (`Quake/WinQuake/sound.h`)

Audio system API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `S_Init` | `void S_Init(void)` | Initialize sound system |
| `S_Shutdown` | `void S_Shutdown(void)` | Shutdown sound system |
| `S_StartSound` | `void S_StartSound(int entnum, int entchannel, sfx_t *sfx, vec3_t origin, float fvol, float attenuation)` | Play 3D positioned sound |
| `S_StopSound` | `void S_StopSound(int entnum, int entchannel)` | Stop specific sound |
| `S_StaticSound` | `void S_StaticSound(sfx_t *sfx, vec3_t origin, float vol, float attenuation)` | Play ambient looping sound |
| `S_Update` | `void S_Update(vec3_t origin, vec3_t forward, vec3_t right, vec3_t up)` | Update listener position and mix audio |
| `S_StopAllSounds` | `void S_StopAllSounds(qboolean clear)` | Stop all playing sounds |
| `S_PrecacheSound` | `sfx_t *S_PrecacheSound(char *name)` | Load and cache a sound file |
| `S_ClearBuffer` | `void S_ClearBuffer(void)` | Fill DMA buffer with silence |
| `S_TouchSound` | `void S_TouchSound(char *name)` | Mark sound as used (prevent cache eviction) |

**Sound backends**: `snd_win.c`, `snd_linux.c`, `snd_sun.c`, `snd_dos.c`, `snd_gus.c`, `snd_null.c`

---

## 6. Network Interface (`Quake/WinQuake/net.h`)

Network subsystem API (WinQuake).

| Function | Signature | Purpose |
|----------|-----------|---------|
| `NET_Init` | `void NET_Init(void)` | Initialize network drivers |
| `NET_Shutdown` | `void NET_Shutdown(void)` | Shutdown all network connections |
| `NET_CheckNewConnections` | `qsocket_t *NET_CheckNewConnections(void)` | Accept incoming connection |
| `NET_Connect` | `qsocket_t *NET_Connect(char *host)` | Connect to remote host |
| `NET_GetMessage` | `int NET_GetMessage(qsocket_t *sock)` | Receive message (1=reliable, 2=unreliable) |
| `NET_SendMessage` | `int NET_SendMessage(qsocket_t *sock, sizebuf_t *data)` | Send reliable message |
| `NET_SendUnreliableMessage` | `int NET_SendUnreliableMessage(qsocket_t *sock, sizebuf_t *data)` | Send unreliable datagram |
| `NET_SendToAll` | `int NET_SendToAll(sizebuf_t *data, int blocktime)` | Broadcast to all clients |
| `NET_CanSendMessage` | `qboolean NET_CanSendMessage(qsocket_t *sock)` | Check if send buffer is available |
| `NET_Close` | `void NET_Close(qsocket_t *sock)` | Close connection |

**Network drivers**: `net_loop.c` (loopback), `net_dgrm.c` (datagram protocol), `net_udp.c` (BSD), `net_wins.c` (Winsock), `net_wipx.c` (IPX), `net_ser.c` (serial)

---

## 7. Rendering Interface (`Quake/WinQuake/render.h`)

Rendering subsystem API — shared by software and OpenGL renderers.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `R_Init` | `void R_Init(void)` | Initialize renderer |
| `R_InitTextures` | `void R_InitTextures(void)` | Load default textures |
| `R_RenderView` | `void R_RenderView(void)` | Render complete 3D scene |
| `R_NewMap` | `void R_NewMap(void)` | Handle new map load |
| `R_SetVrect` | `void R_SetVrect(vrect_t *pvrectin, vrect_t *pvrect, int lineadj)` | Set rendering viewport |
| `R_AddEfrags` | `void R_AddEfrags(entity_t *ent)` | Add entity fragments to BSP leaves |
| `R_RemoveEfrags` | `void R_RemoveEfrags(entity_t *ent)` | Remove entity fragments |

---

## 8. Command System Interface (`Quake/WinQuake/cmd.h`)

Console command execution API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Cbuf_Init` | `void Cbuf_Init(void)` | Initialize command buffer |
| `Cbuf_AddText` | `void Cbuf_AddText(char *text)` | Append text to command buffer |
| `Cbuf_InsertText` | `void Cbuf_InsertText(char *text)` | Insert text at front of buffer |
| `Cbuf_Execute` | `void Cbuf_Execute(void)` | Execute all pending commands |
| `Cmd_AddCommand` | `void Cmd_AddCommand(char *cmd_name, xcommand_t function)` | Register command handler |
| `Cmd_Argc` | `int Cmd_Argc(void)` | Get argument count |
| `Cmd_Argv` | `char *Cmd_Argv(int arg)` | Get argument by index |
| `Cmd_Args` | `char *Cmd_Args(void)` | Get all arguments as single string |
| `Cmd_TokenizeString` | `void Cmd_TokenizeString(char *text)` | Parse command line into tokens |
| `Cmd_ExecuteString` | `void Cmd_ExecuteString(char *text, cmd_source_t src)` | Execute a single command string |
| `Cmd_CompleteCommand` | `char *Cmd_CompleteCommand(char *partial)` | Tab-completion for commands |

---

## 9. CVar Interface (`Quake/WinQuake/cvar.h`)

Console variable API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Cvar_RegisterVariable` | `void Cvar_RegisterVariable(cvar_t *variable)` | Register new cvar |
| `Cvar_Set` | `void Cvar_Set(char *var_name, char *value)` | Set cvar string value |
| `Cvar_SetValue` | `void Cvar_SetValue(char *var_name, float value)` | Set cvar numeric value |
| `Cvar_VariableValue` | `float Cvar_VariableValue(char *var_name)` | Get cvar as float |
| `Cvar_VariableString` | `char *Cvar_VariableString(char *var_name)` | Get cvar as string |
| `Cvar_CompleteVariable` | `char *Cvar_CompleteVariable(char *partial)` | Tab-completion for cvars |
| `Cvar_Command` | `qboolean Cvar_Command(void)` | Handle "cvarname value" console input |
| `Cvar_FindVar` | `cvar_t *Cvar_FindVar(char *var_name)` | Look up cvar by name |
| `Cvar_WriteVariables` | `void Cvar_WriteVariables(int f)` | Write archived cvars to config file |

---

## 10. QuakeC VM Interface (`Quake/WinQuake/progs.h`, `Quake/WinQuake/pr_comp.h`)

QuakeC virtual machine API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `PR_Init` | `void PR_Init(void)` | Initialize VM |
| `PR_ExecuteProgram` | `void PR_ExecuteProgram(func_t fnum)` | Execute QuakeC function |
| `PR_LoadProgs` | `void PR_LoadProgs(void)` | Load `progs.dat` bytecode |
| `ED_Alloc` | `edict_t *ED_Alloc(void)` | Allocate new entity |
| `ED_Free` | `void ED_Free(edict_t *ed)` | Free entity |
| `ED_ClearEdict` | `void ED_ClearEdict(edict_t *e)` | Zero entity fields |
| `ED_FindFunction` | `dfunction_t *ED_FindFunction(char *name)` | Find QuakeC function by name |
| `ED_PrintEdicts` | `void ED_PrintEdicts(void)` | Debug print all entities |
| `PR_Profile_f` | `void PR_Profile_f(void)` | Print VM execution profile |

**Key builtin functions bridge** (`Quake/WinQuake/pr_cmds.c`):

| Builtin # | Function | Engine Function Called |
|-----------|----------|----------------------|
| #1 | `makevectors(angles)` | `AngleVectors()` |
| #2 | `setorigin(entity, origin)` | `SV_SetOrigin()` |
| #3 | `setmodel(entity, model)` | `SV_SetModel()` |
| #8 | `sound(entity, chan, sample, vol, atten)` | `SV_StartSound()` |
| #14 | `spawn()` | `ED_Alloc()` |
| #15 | `remove(entity)` | `ED_Free()` |
| #16 | `traceline(v1, v2, nomonsters, forent)` | `SV_Move()` |
| #19 | `precache_sound(name)` | Load into sound cache |
| #20 | `precache_model(name)` | Load into model cache |
| #40 | `checkbottom(entity)` | `SV_CheckBottom()` |

---

## 11. Model Loading Interface (`Quake/WinQuake/model.h`)

Model and BSP loading API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Mod_Init` | `void Mod_Init(void)` | Initialize model system |
| `Mod_ClearAll` | `void Mod_ClearAll(void)` | Clear model cache |
| `Mod_ForName` | `model_t *Mod_ForName(char *name, qboolean crash)` | Load model by name (cached) |
| `Mod_Extradata` | `void *Mod_Extradata(model_t *mod)` | Get model data from cache |
| `Mod_TouchModel` | `void Mod_TouchModel(char *name)` | Mark model as used |
| `Mod_PointInLeaf` | `mleaf_t *Mod_PointInLeaf(vec3_t p, model_t *model)` | Find BSP leaf at position |
| `Mod_LeafPVS` | `byte *Mod_LeafPVS(mleaf_t *leaf, model_t *model)` | Get PVS for leaf |

---

## 12. Memory Management Interface (`Quake/WinQuake/zone.h`)

Custom memory allocation API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Memory_Init` | `void Memory_Init(void *buf, int size)` | Initialize memory system |
| `Z_Malloc` | `void *Z_Malloc(int size)` | Allocate from zone (general purpose) |
| `Z_Free` | `void Z_Free(void *ptr)` | Free zone memory |
| `Hunk_Alloc` | `void *Hunk_Alloc(int size)` | Allocate from hunk (permanent) |
| `Hunk_AllocName` | `void *Hunk_AllocName(int size, char *name)` | Allocate named hunk block |
| `Hunk_TempAlloc` | `void *Hunk_TempAlloc(int size)` | Temporary allocation (freed next call) |
| `Hunk_HighAllocName` | `void *Hunk_HighAllocName(int size, char *name)` | Allocate from high end of hunk |
| `Cache_Alloc` | `cache_user_t *Cache_Alloc(cache_user_t *c, int size, char *name)` | Allocate evictable cache memory |
| `Cache_Free` | `void Cache_Free(cache_user_t *c)` | Free cache memory |
| `Cache_Check` | `void *Cache_Check(cache_user_t *c)` | Check if cache entry is still valid |

---

## 13. Common Utilities Interface (`Quake/WinQuake/common.h`)

Shared utility functions.

| Function | Purpose |
|----------|---------|
| `COM_Init()` | Initialize common subsystem |
| `COM_LoadFile(char *path, int usehunk)` | Load file from PAK or filesystem |
| `COM_LoadHunkFile(char *path)` | Load file into hunk memory |
| `COM_FileBase(char *in, char *out)` | Extract filename without extension |
| `COM_CheckParm(char *parm)` | Check command-line parameter |
| `MSG_WriteChar/Byte/Short/Long/Float/String/Coord/Angle` | Write typed values to network message buffer |
| `MSG_ReadChar/Byte/Short/Long/Float/String/Coord/Angle` | Read typed values from network message buffer |
| `SZ_Init(sizebuf_t *buf, byte *data, int length)` | Initialize sized buffer |
| `SZ_Write(sizebuf_t *buf, void *data, int length)` | Write to sized buffer |
| `SZ_Print(sizebuf_t *buf, char *data)` | Print string to sized buffer |
| `CRC_Init/ProcessByte/Value` | CRC-16 checksum calculation |
| `va(char *format, ...)` | Printf to static buffer (convenience) |

---

## 14. World / Physics Interface (`Quake/WinQuake/world.h`)

Spatial query and collision detection API.

| Function | Signature | Purpose |
|----------|-----------|---------|
| `SV_InitBoxHull` | `void SV_InitBoxHull(void)` | Initialize bounding box collision hull |
| `SV_HullForEntity` | `hull_t *SV_HullForEntity(edict_t *ent, vec3_t mins, vec3_t maxs, vec3_t offset)` | Get collision hull for entity |
| `SV_AreaEdicts` | — | Find entities in area (QuakeWorld) |
| `SV_Move` | `trace_t SV_Move(vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int type, edict_t *passedict)` | Trace movement through world |
| `SV_RecursiveHullCheck` | `qboolean SV_RecursiveHullCheck(hull_t *hull, int num, float p1f, float p2f, vec3_t p1, vec3_t p2, trace_t *trace)` | Recursive BSP hull collision test |
| `SV_LinkEdict` | `void SV_LinkEdict(edict_t *ent, qboolean touch_triggers)` | Link entity into world area grid |
| `SV_UnlinkEdict` | `void SV_UnlinkEdict(edict_t *ent)` | Remove entity from world area grid |

---

## 15. Math Library Interface (`Quake/WinQuake/mathlib.h`)

Vector and matrix math utilities.

| Function | Purpose |
|----------|---------|
| `VectorNormalize(vec3_t v)` | Normalize vector, return length |
| `CrossProduct(vec3_t v1, vec3_t v2, vec3_t cross)` | Cross product |
| `VectorMA(vec3_t veca, float scale, vec3_t vecb, vec3_t vecc)` | `vecc = veca + scale * vecb` |
| `AngleVectors(vec3_t angles, vec3_t forward, vec3_t right, vec3_t up)` | Convert Euler angles to direction vectors |
| `BoxOnPlaneSide(vec3_t emins, vec3_t emaxs, mplane_t *plane)` | Test bounding box against plane |
| `R_ConcatRotations(float in1[3][3], float in2[3][3], float out[3][3])` | Concatenate 3×3 rotation matrices |
| `R_ConcatTransforms(float in1[3][4], float in2[3][4], float out[3][4])` | Concatenate 3×4 transform matrices |
| `FloorDivMod(double numer, double denom, int *quotient, int *rem)` | Floor division with remainder |
| `GreatestCommonDivisor(int i1, int i2)` | GCD computation |

**Inline macros** (`Quake/WinQuake/mathlib.h`):
```c
#define DotProduct(x,y)    (x[0]*y[0]+x[1]*y[1]+x[2]*y[2])
#define VectorSubtract(a,b,c) {c[0]=a[0]-b[0];c[1]=a[1]-b[1];c[2]=a[2]-b[2];}
#define VectorAdd(a,b,c)      {c[0]=a[0]+b[0];c[1]=a[1]+b[1];c[2]=a[2]+b[2];}
#define VectorCopy(a,b)       {b[0]=a[0];b[1]=a[1];b[2]=a[2];}
```
