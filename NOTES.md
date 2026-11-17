# GOTGVR - research and technical notes

See EXE_REPORT.md for the full exe inventory, the post filter table and the engine family research.

Static analysis of the supplied `GOTG.exe` (PE32+ x64, link date 0x63D20AF5 = 2023-01-26, Steam, 508 MB, image base
0x140000000). Addresses are RVAs (exe+...). Nothing here has been checked in a running game yet.

## The exe

- Sections: `.shared` (0x1000, 36 MB, the real code, readable on disk), `.udata` (read-only data: vtables, RTTI COLs,
  strings), `.arch` (writable data), `.debug` (445 MB, executable - the protector's), `.data*`. The code is not
  encrypted on disk, so signatures are scanned in `.shared` only (sections between 1 and 256 MB).
- Same build as IGCS GotGCameraTools 1.0.2: all of its AOBs match exactly once.
- Imports winmm.dll statically (timeBeginPeriod / timeEndPeriod) -> the mod loads as winmm.dll. dxgi / d3d12 are
  loaded dynamically (`12on7\D3D12.dll` first, then `d3d12.dll`).

## Renderer

- Dawn engine (Glacier 2 lineage), the DXMD engine: RTTI `ZRenderGraphNodeCamera`, `ZRenderCameraEntity`,
  `ZCameraEntity`, `ZRenderDeviceD3D11/D3D12/Vk`, `ZRenderSwapChainD3D11/D3D12/Vk`.
- API choice exe+C9B40: ini options `USEVULKAN` (2) / `USED3D12` (1), else 0 = D3D11 ("base"); exe+99F440 constructs
  ZRenderDeviceD3D12 ("advanced", exe+7C9640), Vk (exe+7ED120) or D3D11 (exe+7BC540). The shipped game requires D3D12;
  the D3D11 path is compiled in but untested (its shaders may not ship). The mod targets D3D12.
- Settings registration exe+1CC1E40 (runtime.pcsettings): VSync, HDR, MultiGPU, Fullscreen, ExclusiveFullscreen,
  Monitor, Adapter, FullscreenWidth/Height, RefreshRate, Brightness, MaxRefreshRate, DynResType, DlssQuality,
  Sharpening, DlssSharpening, ExposureOffset, ResScale, TextureFiltering, GeomQuality, ShadowQuality, TextureQuality,
  RayTracing, RayTracingTransparentReflections, SoftShadows, AmbOcclusion, SSReflections, MotionBlur, RadialBlur,
  ChromaticAberration, Dof, LensFlare, AntiAliasing, LightScattering, ApiLocale, FSRQuality.
  **No Stereoscopic3DMode / StereoSeparation / StereoPopOut** (DXMD had them at settings +0x94..+0xA4).

## Stereo 3D - what is left

- No "side-by-side" strings, no settings, and no eye-index driver: DXMD's stereo driver stores the eye index
  (view+528h = 1, 0, 2) around two scene records; a scan for that store pattern finds nothing comparable here.
- The projection getter (below) only adds TAA jitter (exe+979FE0) - DXMD's StereoProj (P[8] / P[12] from the
  separation / pop-out sliders) is gone.
- What survived: exe+1CC4810 (called from the launcher dialog exe+1CC15AD, `DxmConfigDialog`): creates a D3D11
  device, checks for an AMD GPU (vendor 0x1002 / 0x1022), loads `atidxx64.dll` -> `AmdDxExtCreate11`, queries the
  quad-buffer stereo interface (id 2) and lists stereo display modes ("Could not load AMD stereoscopic extension").
  That is the launcher's HD3D mode list only; nothing in the D3D12 renderer uses it. This is the "hidden 3D mode".
- Hence alternate-eye rendering.

## Render camera (ZRenderGraphNodeCamera, vtable exe+2571C98)

- slot 9 = view update exe+990CB0 (rcx node, rdx context, r8 view, r9d n) - the analogue of DXMD exe+5A5FC0:
  - node+0x24 SMatrix43 (movups xmm4/xmm3/xmm10 from [rsi+24h/34h/44h], psrldq 4 for the translation): rows right /
    up / back + translation - identical to DXMD's node+0x24.
  - first call exe+972270 = projection getter (node, float out[16], bool keep): copies node+0xC0..0xFF, unless the
    camera is orthographic or `keep` is set rebuilds rows 2/3 from near / far (node+0x1D0 / +0x1D4, global override
    exe+3D97774) - row 2 = (0, 0, -f/(f-n), -1), row 3 = (0, 0, -fn/(f-n), 0): this ZEROES the off-axis terms - then
    exe+979FE0 adds the TAA jitter to [8] / [9] (jitter / node+0x1A8, node+0x1AC = render width / height).
    [11] = -1: row-vector, right-handed, D3D depth - the same form as DXMD (+0x220 there).
  - builds view+0x10.. (rows), view+0xE20 (position), previous-frame copies view+0x100.., view+0x140.. (for motion
    vectors).
- Callers of the getter: exe+8C6491, 8CA1DE, 990D32 (view update), 9B31B6.
- The mod hooks the view update (pose into node+0x24 for the main node; "clean" base kept, see camera.cpp) and the
  getter (x / y terms = the eye's frustum, the game's depth rows and jitter kept).
- Main node = the node whose render size equals the back buffer, seen most per frame.

## Gameplay camera (IGCS)

- `AOB_CAMERA_ADDRESS_INTERCEPT` exe+C999A4 in exe+C99950: copies a 0x74-byte camera record from a behaviour's vfunc
  (+0x10) into r14+0x390: +0x20..+0x5F 4x4 (rows, position in row 3), +0x60 float (FOV), +0x64..+0x70 more; then
  exe+C83780(r14, record). This is the logic camera (what IGCS and the head-tracking mod move); the VR mod does not
  touch it.

## Patches (IGCS GotGCameraTools 1.0.2 sites)

- `AOB_NO_BLACKBARS_CUTSCENES1/2` exe+960D7F / exe+960DD5 inside exe+960C8C: the cutscene viewport is letterboxed
  when w/h > 16:9 (constant exe+243E2B0 = 1.7778) or pillarboxed when h/w > 9:16 (exe+253C940 = 0.5625). `jbe` ->
  `jmp` (0x76 -> 0xEB) skips both. Needed: a 1:1 / 5:4 eye would be letterboxed in every cutscene.
- `AOB_NO_PAUSE_ON_FOCUS_LOSS` exe+CBC2F: `je` over "Win32 Gameloop Pause" / "OnGameLostFocus" -> `90 E9 73 02 00 00`.
- `AOB_HUD_DISABLE_FLAG` exe+BD5619: `movzx eax, byte [exe+3DCB920]`, compared with [rbx+421h], copied there with a
  dirty flag [rbx+420h]: the HUD on / off state. The mod flips that byte (F7).
- Not used: DOF / hex DOF / glare / radial blur pass function starts (exe+8D6780 / 905950 / 904E50 / 906910 /
  929E21), high-LOD scale (exe+12305D), game state manager (exe+CBC35), timer struct (exe+42E963: [r8+68h] x
  [r8+6Ch] scale the frame delta - game speed).

## D3D12 output path

- IDXGIFactory(2)::CreateSwapChain(ForHwnd) hooked through a factory of our own; the "device" argument is the game's
  direct command queue. Back buffers via GetBuffer, device via queue->GetDevice.
- The host needs D3D11 keyed-mutex textures: the mod runs its own D3D11.4 device on the game's adapter (LUID), shares
  a ring of 3 textures (D3D12 shared heap -> OpenSharedResource1) and two shared fences (D3D12 -> D3D11 "copied",
  D3D11 -> D3D12 "read"). The copy is recorded on the game's queue right before its Present.
- HDR: SetColorSpace1 is hooked; PQ (BT.2020) and scRGB inputs are tone mapped to SDR in the copy shader.

## Performance audit (0.6.8, against the framework's VR Performance Manual)

| Manual | GOTGVR (D3D12) |
|---|---|
| 3 threads | in-process host thread HIGHEST; the game's process priority untouched; limiter: high-resolution waitable timer + short spin |
| 4 instrumentation | `perf:` line (Present work avg / max, longest frame, GPU frame interval avg / max, ring busy); recon GPU ms; host stats |
| 6 managed pool | not applicable (D3D12) |
| 7.1 render size | recommended x render_scale, widened per axis by the symmetric view; no automatic lowering |
| 7.3 atlas re-publish | happens on a stereo-method change (render size follows the view) - not during play |
| 8 stereo methods | recon centre view = one view per frame like AER; synchronized = same-tick |
| 9.1 / 9.2 refresh | never below the full rate; halved-rate rule only on Pimax; 30 consistent reports for a faster rate |
| 9.3 cap | AER / synced exactly the refresh; recon / head <= refresh; flat: user's |
| 9.5 never wait | free ring slot or skip (counted); keyed-mutex slots acquired with timeout 0 |
| 10 runtimes / layers | API layers logged by the host; runtime name drives 9.2 |
| 11 atlas | 4 slots, LUID logged, direct-copy format |
| 12 hook cost | per-draw UI bookkeeping sampled 1 frame in 8; PSO flags looked up once per SetPipelineState |
| 15 logging | counters only in hot paths, one report per 10 s |

## Render options (0.6.4)

- PC settings block exe+3E603A0 (the manager is exe+3E60398; values at +8; registered in exe+1CC1E70; defaults in
  exe+1CC4A60). Fields, as {type, offset}:
  - VSync +0, HDR +1, Fullscreen +2, ExclusiveFullscreen +3, MultiGPU +4
  - FullscreenWidth / Height +8 / +C, RefreshRate +10, MaxRefreshRate +14 (float), Monitor +18, Adapter +1C,
    Brightness +20 (float)
  - DynResType +24 (int, default 0), DlssQuality +28, Sharpening +2C, DlssSharpening +30, ExposureOffset +34,
    ResScale +38 (float, %, default 100)
  - TextureFiltering +40, GeomQuality +44, ShadowQuality +48, TextureQuality +4C, RayTracing +50, RT reflections +54
  - bytes: SoftShadows +58, AO +59, SSR +5A, MotionBlur +5B, RadialBlur +5C, ChromaticAberration +5D, Dof +5E,
    LensFlare +5F, AntiAliasing +60 (default 1), LightScattering +61
  - ApiLocale +68 (int), FSRQuality +64
- Apply exe+CDBF1 pushes them into the option table exe+3DE1C70. The table's setters are thunks into protected code:
  - set bool exe+DC7540, set int exe+DC7700, set float exe+DC76C0 (rcx = table, edx = id, r8 / xmm2 = value)
  - get bool exe+D4010: an FNV-1a hash map (+0x78 bucket count, +0x80 buckets, +0x88 24-byte entries
    {next, id, object}).
  - The options menu's apply (exe+1B6C910) uses the same setters from its own copy (exe+3D86A00).
- Option ids:
  - 0x4BE AntiAliasing, 0x4B9 MotionBlur, 0x4BC Dof, 0x4BA ChromaticAberration, 0x4BD RadialBlur, 0x4BB LensFlare,
    0x4B8 SSR, 0x4B7 AO
  - 0x51B DynResType, 0x51A ResScale, 0x51C Sharpening, 0x51D DlssQuality, 0x51E DlssSharpening, 0x520 FSRQuality
  - 0x4B0 graphics preset
- The renderer copies DynResType..FSRQuality into its own struct at start (exe+9A4050: +0x8B0..+0x8F0).
- AA per view (exe+A71B80): view+0x384 = 0 none, 1 AA without TAA, 2 TAA. TAA needs exe+977B70 = the byte
  exe+305A116 and the count exe+305A118 > 0, plus view flags. The AA pass (exe+8D53B0) picks "[AA] ApplyTAA" or
  "[AA] ApplyNoTAA" from it.
- GOTGVR: game.cpp FindRenderOptions finds the AntiAliasing sequence of the apply (setting address, table, bool
  setter). It then picks the int / float setter sequences whose setting address and table match. Applied on the
  game thread (TimerUpdate), after a mode has stayed for 1 s, only once the option exists in the table.

## UI

- GOTG's UI is its own `ui::` system (ZUIManager@ui, ZLayoutManager@ui, ZCanvasEntity, ZRenderGraphNodeCanvas), not
  DXMD's Scaleform HUD box (ZPlatformManagerWindows slots 7 / 8 do not exist here). There is no HUD scale option in
  the game.
- 0.6.4: no UI draw reaches the final image. The UI goes into a layer of its own, blended in by a compute pass.
  The layer is taken as the colour target with the most draws that is never drawn with the scene depth (the depth
  buffer with the most draws), 8-bit formats first, 30 frames in a row. Hide HUD works through the game's HUD flag
  (exe+BD5619), whatever the UI path is.
- The mod classifies draws into the final image (back buffer, or a texture the game copies into it) on the D3D12
  command lists: the first full-target full-screen triangle / quad is the 3D image, every later draw is UI; UI draws
  get their viewport and scissor scaled about each eye's optical axis + the convergence shift for ui_depth.

## Camera chain, from the IGCS / head-tracking hook to the render node (backtrace)

1. Camera behaviours (ZCameraBehavior_* - follow, aim, cover, spline, look-at ...) produce a camera record each frame.
2. exe+C997C2 (a vfunc of the camera manager, referenced from exe+911472): asks the active behaviour (vfunc +0x10 of
   r14+0x530) for its record and copies it to r14+0x390. **This copy is the IGCS `AOB_CAMERA_ADDRESS_INTERCEPT`
   (exe+C999A4) that the OpenTrack head-tracking mod hooks**: +0x20..+0x5F 4x4 (rows, position in row 3), +0x60 FOV.
3. exe+C83780(manager, record): gets the ZCameraEntity, turns record+0x20 into a transform (exe+CA65D0) and sets it on
   the entity (exe+578DC0 / exe+588540 = the spatial entity's world transform); FOV: record+0x60 x 57.29578
   (exe+2441AD4) -> the camera's +0x130 interface vfunc +0x228 (set FOV).
4. ZCameraReflector (RTTI, vtable exe+2567778; TRenderGraphNodeReflectorCommon<ZRenderCameraEntity,
   ZRenderGraphNodeCamera> exe+2567680) copies the entity's transform into its render node at the game / render sync.
5. ZRenderGraphNodeCamera node+0x24 -> the per-frame view update exe+990CB0, called from the scene-view builder
   exe+97FD30 (node vfunc +0x40 = slot 8, then +0x48 = slot 9 with the view at rbp+0xA8) - **GOTGVR's hook**, and
   the projection getter exe+972270.
Hooking at 2 (head-tracking mod) moves the logic camera: aiming, the HUD reticle and the audio listener follow the
head, and everything downstream (including culling) sees it one frame later. Hooking at 5 (GOTGVR) changes only what
is drawn, in the same frame, and keeps per-eye poses out of game logic - the right place for VR (it is where the
official DXMD VR Experience applies the headset pose, DXVR exe+59D510).

## Game clock (synchronized stereo)

- exe+D23C0 (main loop step, also exe+D8C00 while loading): exe+42E980 BeginFrame(timer) ... exe+42E890
  TimerUpdate(timer, paused) for the game timer exe+3C2CDF8 (lea rcx at exe+D2488). TimerUpdate: guard +0x78,
  +0x28 = old time, +0x38 = old delta, delta +0x30 = (+0x6C x +0x68 x 2^20) x raw +0x50 >> 20 (clamped by +0xC0),
  time +0x20 += delta; paused -> delta 0. IGCS's timestop / game speed act on this struct (AOB_TIMER_STRUCT at the
  `sar rcx,14h` exe+42E963).
- GOTGVR holds the delta on every second tick (time unchanged) and adds it to the next tick. UI timers are separate
  and keep running.

## TAA flag

- exe+305A116 (writable data): checked by the TAA jitter exe+979FE0 and in exe+97262B; 0 in the file, set at run time
  through the settings (no rip-relative writer). The r/FuckTAA hex patch for the 2021 builds edited this byte's
  initial value; its patterns no longer match this build (nor its 64-pixel resolution truncation patch). GOTGVR
  keeps the byte at 0 at run time instead.

## True single-frame stereo - assessment

- No stereo renderer left (above). Two eyes inside one frame would need a second scene view of the camera node each
  frame: exe+97FD30 builds the frame's list of (node, view) and runs each node's view update; the frame renderer
  then draws each view. Adding a view (or running the frame renderer twice with per-eye views) means duplicating
  per-view GPU state, history buffers and fences of the renderer - possible in principle (render-to-texture cameras
  already add views), but not something to write blind. Synchronized sequential gives the same "both eyes from one
  game state" result through the clock.

## StereoShutter

- murdavs/StereoShutter drives 3D Vision glasses; game views come from geo-11 fixes (DX11) or an unreleased DX12
  layer. Nothing GOTG-specific.

## Everything that reads the camera node (0.3.0)

- View update exe+990CB0 (getter with keepDepth 0) - the frame's view, culling and matrices.
- exe+8C6350 (called from exe+9EF5E6, "Skin Blur" / "Skin Blur DT" pass): getter keepDepth 1 + node+0x24.
- exe+8CA170 (called from exe+8CAC21, the post filter temp-target setup, "PostfilterTemp_{0}x{1}x{2}"): getter
  keepDepth 1 + node+0x24, scaled by the global exe+304D294.
- exe+9B3137: getter keepDepth 1 + node+0x24 (inline view build).
- With keepDepth the getter returns node+0xC0 + jitter, and nothing writes the getter's result back into node+0xC0
  (the only 4x4 store at +0xC0 in the renderer, exe+9AC6CD, is into a view record) - so adding the eye's off-axis
  terms in every call is safe (no double offset).

## Research (0.3.0)

- Paul44's CE table v.30072022 (FearLess / gamepressure mirror): cheats and pointers, 2022 build.
- r/FuckTAA (TL431): TAA and 64-pixel resolution truncation hex patches for the 2021 builds - none of the patterns
  exist in this build (the developers changed both; steamdb notes for a 2022 patch list "fix issues relating to
  square aspect window play").
- Nexus "Simple Realistic 3D for Guardian of the Galaxy": ReShade + Depth3D (depth-buffer 3D, top-bottom / SBS).
- vorpX: unofficial (forum reports). Luke Ross: no GOTG mod. No GOTG camera / VR source code on GitHub.

## Stereo reconstruction (0.4.0)

Goal: both eyes new every frame at full game rate, from the same moment. One eye is rendered (AER camera path), the
other is synthesised from that frame's colour + depth. Same family of technique as PureDark's AFW (UEVR / REFramework
"AFW" branches: the warp itself is a closed plugin, `PDAFWPlugin.dll`; the public branches only carry its header and a
dummy) and the Dark Souls III native VR mod's AFW (D3D11, no licence of its own). GOTGVR's code is its own.

- Depth: `ID3D12Device::CreateDepthStencilView` (vtable 21) maps DSV handles to textures; at `OMSetRenderTargets`
  a depth bound with >= 1 colour target is a candidate; draws are tallied per frame; the busiest one with the back
  buffer's shape is the scene depth (switches at once if it is not drawn into any more - resolution change).
  The exe has the format string `D32F_S8_X24` and passes `ZRenderPassGBuffer`, `GBuffer_Depth`, `ZRenderPassVelocity`,
  `ZRenderPassScreenSpaceUI`; it uses no D3D12 render passes and no enhanced barriers (no List7 / Barrier()).
- Copy point: `ResourceBarrier` (list vtable 26): a transition of the scene depth out of DEPTH_WRITE (not to COMMON)
  appends depth -> COPY_SOURCE, plane 0 -> buffer (placed footprint from GetCopyableFootprints), buffer -> shared
  R32_UINT / R16_UINT texture, back to the game's state. Fallback at the first UI draw with the last state seen, only
  if the depth was used this frame. Direct lists only. Slot = g_presents % 3, tagged with g_presents; the compositor
  makes the game queue wait for fence B of the next slot.
- Picture without UI: at the first UI draw (d3d12hooks' scene/UI split) the bound final target is copied.
- Camera: the view update records the drawn eye's node matrix, the projection the getter returned to the view update
  (keepDepth = 0: the depth rows + jitter the depth was drawn with), and the other eye's node matrix / frustum from
  the same head pose. Matrices are built in double with the world origin at the drawn eye.
- Shared frustum: per drawn eye, the union of its frustum and the other eye's (corners rotated by Re^T Ro for canted
  displays), +3 %, image aspect kept. UI scaled by headset width / drawn width so it keeps its angular size.
- Passes (D3D11 cs_5_0): forward (render size): clip -> world (homogeneous, sky = w 0) -> other clip, atomic min of
  the distance over a 2 x 2 footprint. Resolve (eye size): distance -> ndc z (z = -P10 + P14 / w) -> drawn eye uv;
  empty: march left / right; both sides close and within 3 % = crack (nearer), else hidden (deeper side); hidden or
  outside -> history (that eye's own last real image, reprojected with the drawn depth) when on. UI: pixels where
  final != scene (> 1.5 / 255) copied through the UI centre / scale mapping.
- Accuracy model (tools/recon_sim.py, numpy transcription of the passes): median reprojection error 0.26 px (own
  view) / 0.44 px (shared view, coarser source), p95 0.49 / 0.66 px; direct coverage 93 % -> 99.9 % with the shared
  view; 10 degree canting: 76 % -> 98.6 %.

## Flat screen detection (0.4.2)

- Pause: exe+D2457..D2488 - `call 429780(pauseMgr, id)`; if true and `call 427510(&timer2)` is false: dl = 1, else 0;
  then `TimerUpdate(&gameTimer exe+3C2CDF8, dl)`. With dl = 1 the clock step is 0 (exe+42E974). The camera hook keeps
  the last value; flat after 0.3 s paused with the game camera still (pause menu screens), never while the camera moved
  after the pause began (photo mode) until the game runs again.
- Videos: GOTG imports bink2w64.dll (BinkOpen / BinkDoFrame / BinkDoFrameAsync / BinkDoFramePlane / BinkClose ...);
  `ZBink`, `ZRenderGraphNodeVideoPlayer`, `ZRenderToTextureVideoPlayer` use it. BINK starts with U32 Width, Height.
  Cutscene = >= flat_video_min_width wide, decoded within 400 ms, game camera still 0.5 s (or paused).
- No camera update at all for 3 Presents (title, loading) was already flat (CameraFrameIsVR).

## Camera / world conventions - verified (0.4.3)

- Render node (ZRenderGraphNodeCamera) node+0x24 = SMatrix43, rows right, up, back, position. exe+990D40..990E06 (view
  update) loads the three rows and the position and builds the rigid inverse (transpose + -R^T t); with the getter's
  right-handed projection ([11] = -1, w = -z) view x = row 0, y = row 1, the camera looks along -row 2.
- The gameplay camera entity (IGCS AOB exe+C999A4 copy, the head-tracking mod's hook) is a different matrix: rows
  right, forward, up (its ini: RightRow 0, FwdRow 1, UpRow 2). ZCameraReflector converts it for the render node.
- World: Z-up (the render node's up row = world Z on a level camera; the entity's up row is Z), metres: game code stores
  9.81 as a gravity magnitude (exe+20E892D `mov [rcx+0xE8], 9.81`), Havok physics (`ZPhysicsObjectHavok`), and DXMDVR
  runs DXMD - same engine - at units_per_metre 1.0. The head-tracking mod's "10 units / m" was a feel calibration of
  its own lean gain, not an engine value. A (0, -9.81, 0) constant at exe+299F9A0 belongs to Y-up middleware code.
- Projection getter exe+972270: copies node+0xC0; unless the matrix is orthographic (exe+F7B40: P[15] test) or
  keepDepth, rebuilds rows 2 / 3 from near (node+0x1D0, or the global override exe+3D97774 when > 0) and far
  (node+0x1D4): P[10] = -f / (f - n), P[14] = -f n / (f - n) - standard 0..1 depth, not reversed. Then jitter
  (exe+979FE0).
- exe+9724C0: a second view x projection (same rebuild, inline jitter) from node+0x24 / +0xC0 without the getter;
  its only caller (exe+9CBD30) computes a per-object screen-size value (LOD bucket). It gets the game's FOV with the
  eye's position - LOD choice only.
- View output of the view update: view+0x10.. view rows, +0x50.. projection, +0x90.. view x projection, +0xE20
  camera position (all from the patched getter).
- Pause: see "Flat screen detection". Near override: data in .arch, 0.0 by default.

## Headset smoothing and the game's frame rate (0.4.4)

- Couplings between the headset and the game, all checked: (1) the FPS limiter (the only one that could slow it);
  (2) atlas slots - keyed mutex AcquireSync(.., 0) / slot states, never blocking, a busy slot is skipped; (3) fences A
  / B are GPU-side waits on the mod's own copies (3-deep ring); (4) the host's xrWaitFrame / pacing wait runs in the
  host process; (5) the game's Present runs without vsync when the headset is connected (flip model, sync 0: newer
  frames replace queued ones, no monitor-rate throttle).
- predictedDisplayPeriod doubles on runtimes that throttle the app for their smoothing (Meta ASW, VD SSW, Pimax Smart
  Smoothing, SteamVR when forced). refresh.h undivides it; d3d12hooks keeps the shortest full-rate period per OpenXR
  system name in GOTGVR\refresh.txt.
- Host pacing (third_party/xrbridge host main.cpp RunFrame): mode 1 never waits past the frame budget (period - 4 ms)
  and never when the game is slower; mode 2 waits up to paced_timeout_ms when the game is slower (judged from the
  submit interval - a synchronized pair is every second game frame, so the mod uses mode 1 there); mode 0 never waits.

## Start-up (0.6.1)

- Launcher: built into the exe (ZPcLauncher / ZSettingsDialog@pclauncher, dialog "Guardians of the Galaxy Launcher").
  exe+101E50 parses argc / argv (from WinMain): `-NoLauncher` / `-nolauncher` -> UseLauncher false. Other switches
  there: `-sharedvram`, `-nort` (no ray tracing), `-storewrite`, `-ao CLEANUP` / `-ao DisableOSCheck`.
- The exe imports no `__p___argv`: it builds its argv itself from the WinMain line (`_get_narrow_winmain_command_line`
  = ucrt `_acmdln`) or CommandLineToArgvW(GetCommandLineW()). Imports are through the protection layer (no direct
  `call [IAT]` in .text), so API hooks have to be in kernelbase itself.
- Display settings: the pcsettings registered at exe+1CC1E40 (Fullscreen, ExclusiveFullscreen, FullscreenWidth /
  Height ...), stored in HKCU\Software\Eidos Montreal\Guardians of the Galaxy (PCGamingWiki, Steam forum). The
  "registry_path" default is "Software\Eidos Montreal\Default" (exe+CC9D0).

## First person (0.6.0)

- The chain: ZPlayerManager (exe+3E41B58) -> local character (exe+1784E90) -> ZCharacter interface (type info from
  ZCharacter's QueryInterface) -> slot 80 animated entity -> exe+81E740 bone world QV (id = crc32 of the lower-case
  name). This is the same path ZBoneSpatialFromCharacterEntity (exe+1700200) and the game's own first-person camera
  behaviour (exe+1569400) take. All the calls happen on the game thread, in the clock hook (H_TimerUpdate).
- Every pointer is checked before use (Readable, vtables in code, slot bodies compared with the known bytes), and the
  player manager's vtable is compared before the first call. A build that does not match turns first person off.
- Body hiding: SetVisible(false) only if the body is visible, and SetVisible(true) only if it is still the mod's own
  hide. A level change (new animated entity) forgets the old one.
- Live checks to do: the `firstperson:` log lines (player found, in use / suspended), the eye height (fp_offset_y),
  whether SetVisible also removes the shadow, and whether the game's scripted first-person moments conflict.

## Ideas for later

0. Second scene view per frame for single-frame stereo (exe+97FD30), once the game can be debugged live.
1. Find the canvas root's viewport (ZUIManager::UpdateCanvasBegin, OnViewportResizeEvent) for a layout-level UI box
   instead of the viewport transform, and a HUD-on-a-panel mode (redirect UI draws into a texture of the same format).
2. Turn off TAA / DLSS / FSR / motion blur automatically: hook the pcsettings getters registered at exe+1CC1E40.
3. D3D12 variable rate shading (tier 2) for fixed foveated rendering on any GPU (DXMDVR's was NVAPI / D3D11 only).
4. Synchronous stereo instead of AER: run the frame function twice per game frame (find the render thread's frame
   entry; DXMD exe+58FA50 analogue) - both eyes from the same game state, no AER timing artefacts.
5. Per-eye TAA history (two history sets) so TAA can stay on.

## Next steps (needs the game running)

- Stereo reconstruction: the `recon:` lines - `scene depth` chosen (format 19/20 = D32S8), `depth copies` = frames,
  `reconstructed` > 0, the share of gaps / UI. `recon_debug = 2` shows whether the UI split is right.

0. Check the log's `foveated:` and `swapchain: maximum frame latency` lines.

1. Log: `sig` lines all OK, `camera: main render camera node`, `poses applied` and `projections patched` > 0.
2. Lean test for `camera_axes` and `units_per_metre` (DXMD was metres; the head-tracking mod guessed 10 / m).
3. Check the eye order (stereo depth) and `aer_frame_delay`.
4. Check the `ui:` line (3D vs UI draws) in gameplay, menus and conversations.
