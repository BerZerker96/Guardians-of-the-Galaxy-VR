# Changelog

## 0.6.23-alpha (uncapped frame rate toggle)

- **Uncapped frame rate** (Advanced, top, "Frame rate" section; `fps_unlocked`, off by default): no frame cap at
  all, also in VR - the game runs as fast as it can, beyond the headset's refresh rate, and the headset shows the
  newest frame each refresh. Off = the game runs at the headset's own refresh rate (as before).
- The frame rate options (uncapped, FPS limiter, headset refresh rate) are together at the top of Advanced.

## 0.6.22-alpha (audit: any headset / runtime)

Checked how the mod adapts to the headset and the runtime, end to end:

- **Resolution and shape**: the per-eye size is the runtime's recommendation x render_scale (any headset, any
  runtime), rendered 1:1 - the game renders at exactly the size of the eye images now (the two used to be worked out
  separately and could differ by rounding). The larger of the two eyes' recommendations is used (runtimes may report
  them differently). New: the size is kept within the runtime's largest per-eye image (a high render_scale on a
  high-resolution headset could exceed it - its swap chains would fail, black headset); the start-up size from the
  last session is checked again once the runtime answers.
- **Field of view**: each eye's own field of view comes from the runtime every frame (asymmetric and canted displays,
  changes while running - e.g. a Pimax FOV switch or an IPD change), drawn with that eye's own off-centre view;
  culling uses a symmetric view covering it (wide culling).
- **Refresh rate / fps**: the cap follows the headset's refresh rate. New: when the headset is set to a LOWER rate
  (e.g. 90 -> 72 Hz) the cap follows after 20 s steady (it only ever followed a faster rate before, so the game kept
  running at the old faster rate); halved / 2/3 rates (motion smoothing) are still never taken.
- Swap chain formats: sRGB / UNORM, RGBA or BGRA, whichever the runtime offers (shader conversion when needed).

## 0.6.21-alpha (foveated rendering off by default, no green specks)

- **Foveated rendering is off by default** (new installs; turn it on in Advanced).
- **The ring of moving green specks** with foveated rendering on: the lower shading rate was also used for the game's
  full-screen lighting / effect passes, which break up when shaded coarsely. It is now only used for passes that draw
  geometry (a depth buffer bound).

## 0.6.20-alpha (HUD in alternate eye / synchronized; stereo reconstruction depth back to full strength)

From the 0.6.19 log:

- **HUD in alternate eye / synchronized**: the HUD scaled, but with UI placement "flat" each eye got it in the
  middle of its own picture - the Dream Air's eyes are off-centre (off-centre 0.248), so the two HUDs were far apart
  and did not merge. The HUD is now always put on each eye's line of sight, converged at UI depth, in every method
  (the placement option is gone: both values do the same).
- **Stereo reconstruction depth was too weak**: the log shows the depth terms reconstruction read had near 0.30 (your
  near_clip, through the projection getter's near override) while the picture is drawn with the game's own ~0.09 -
  every distance was taken about 3x too far, so the rebuilt eyes had about a third of the real depth. The getter's
  near override is no longer set; reconstruction gets the real depth again (same 3D as alternate eye) and the HUD
  sits where it should. The near clip is set on the camera node only.
- The camera report also shows the node projection's depth terms.
- Screenshots (Delete) save the left eye only.

## 0.6.19-alpha (HUD scale found again; your settings as the defaults; closer near clip; simple README)

From the 0.6.18 log:

- **HUD / UI scale**: 0.6.18 only accepted a UI layer drawn with "UI pipelines" - the log shows the game's HUD layer
  has none, so no layer was found ("UI layer not found" all session) and nothing was scaled. The HUD layer is now the
  full-screen UI-format layer the game binds LAST in the frame (it composes the HUD after the 3D picture and its
  effects; the effect buffers of the same size are bound earlier), one layer only. The log lists those layers in
  bind order ("ui: full-screen layers ...") so the choice can be checked.
- **UI placement "flat" scales too** (your config had flat - UI scale did nothing in that mode): the UI stays in the
  middle of each eye's picture and is scaled by UI scale.
- **Near clip**: since 0.6.14 the picture is drawn from the camera node's own projection, which the old near-clip
  override never reached (log: near_clip 0.30 set, the camera kept the game's 0.05-0.15 m). The node's near plane is
  now set too, only ever closer than the game's. Default 0.05 m: lean in close without things being cut.
- **Your settings are the defaults for new installs** (the config you sent; obsolete keys removed, near clip 0.05).
  Only the resolution and shape come from each headset (render_resolution = auto). Existing configs are not touched.
- README rewritten: short, with the feature list.

## 0.6.18-alpha (the missing shadows / surface effects: effect buffers were taken as the HUD layer)

Traced back to 0.6.14 (the build after the unmodified exe was sent):

- **The cause of the missing shadows and surface effects.** 0.6.14 let the automatic HUD layer be ANY full-screen
  layer of a UI format at any number of draws (to find the HUD again after a reload). The game also renders some of
  its effects into full-screen RGBA8 targets with 1-4 full-screen passes (the 0.6.15 log listed five such layers, the
  HUD was only one of them) - one of those was taken as "the HUD", and its passes were scaled by the UI scale into
  the middle of the screen, so the effect was missing / wrong across the picture. 0.6.16 made it worse (every such
  layer). Now:
  - a layer only counts as UI when most of its draws use UI pipelines (no depth, blended, one target);
  - inside a UI layer only those UI-pipeline draws are scaled (not clears, copies or effect passes);
  - in the final image, a full-screen pass without a UI pipeline after the 3D picture (a post effect / composite)
    is not scaled either.
- Not TAA: the game's own anti-aliasing is left exactly as before (0.6.17's "On" only acts when you choose it).
- Wide culling (0.6.17) stays the fix for the side pop-in at the headset's size; symmetric view stays off.

## 0.6.17-alpha (HUD scale back; headset-size rendering with wide culling; idle-sway lock; TAA on for the FuckTAA exe)

From the 0.6.16 log:

- **HUD / UI scale works again.** 0.6.16 left full-screen 6-vertex draws in a UI layer alone ("composites") - but the
  game draws every UI sprite that way, so nothing was scaled. Every draw into a UI layer is UI again; with the picker
  at 0 all full-screen UI layers are scaled (menus and the gameplay HUD). The picker only lists full-screen layers
  now (the small 349x344 / 1127x349 render-to-texture targets filled the list and moved the numbers); an old picker
  number is reset to 0 once (config version 9).
- **Back to the headset's own size and shape, without the side pop-in.** 0.6.16's symmetric view rendered
  4788x2994 (1.4 x 1.12 the headset) - slow. Now each eye is drawn with its own off-centre view at 3420x2668 again
  (symmetric view off, config version 9), and "Wide culling" (Advanced, on) gives the game's visibility / shadow /
  LOD side (the projection getter, ~30 calls a frame) one symmetric frustum that covers the eye, while the picture
  keeps the eye's own (node projection). The log shows "projections patched .. (wide culling ..)".
- **Idle-sway lock** (Camera, 0.5): standing still, the game camera's idle drift (third person) and the head bone's
  breathing / bob (first person) no longer move the view. The camera follows only when the game's moves more than
  a little (1.0 = 2.5 degrees / 6 cm), then exactly.
- **Missing shadows / surface effects, over-sharp picture**: the log shows the FuckTAA-patched exe (TAA flag 0). The
  game's shadows, hair, reflections and ambient occlusion are dithered and made to be smoothed by TAA; without it they
  break up or vanish and the picture is over-sharp. "Game anti-aliasing: On" now turns TAA back on in that exe too
  (best with stereo reconstruction: the same view every frame, no eye mixing).
- **Reconstruction fill**: six background taps reaching further into the background with the hole's width, each
  checked to be the same background and spread up / down, the edge weighing most - a soft continuation of the
  background instead of a mirrored copy.
- First person: "character found through the controller" was logged on every lookup (5000 lines) - once now.

## 0.6.16-alpha (HUD scale in menus and gameplay; no pop-in at the sides; Advanced tab; better reconstruction fill)

- **HUD scale works in the main menu and in gameplay.** The 0.6.15 log showed five full-screen UI layers of the same
  size and format: the main menu's UI was candidate 3 and the gameplay HUD candidate 5, so one pick could only ever
  scale one of them. With the picker at 0 (automatic), every full-screen UI layer drawn in the last few seconds is
  scaled now ("UI layers: All", Advanced). Full-screen clears / composites drawn into a UI layer are left alone.
- **Objects, characters and shading no longer pop in and out at the sides** (alternate eye / synchronized). The game
  culls and bins lights from a symmetric view; the per-eye off-centre view of 0.6.14 broke that at the outer edges.
  "One symmetric view for both eyes" is on again (existing configs are moved to it once, config version 8). The
  render grows sideways to keep the headset's pixel density; "Pixel density: fixed" keeps the headset's exact size
  (faster, softer). Off = the 0.6.15 behaviour.
- **New Advanced tab** (last): foveated rendering at the top, then picture width / height, symmetric view, cull
  margin, near clip, eye alignment, render size and timing, and fine-tuning options. The other tabs only keep the
  everyday options.
- **Sections in every tab**: labelled divider lines group the options (Stereo, Quick actions, Camera offset, First
  person, Game effects, ...).
- **Stereo reconstruction fill** (ideas from SuperDepth3D's background infill): a hidden area is filled from the
  background beyond the occluder's edge, mirrored across that edge so the background's texture continues instead of
  one stretched column, checked to be the same background depth (else the edge is used), then softened with a
  small blur that grows with the hole's width.

## 0.6.15-alpha (original exe = FuckTAA-patched exe; first person through the game's own hero lookup)

- **Works the same with the original exe and the FuckTAA-patched one** (checked against both: every signature
  matches). The FuckTAA "TAA fix" is the byte the mod already knows as the TAA flag (exe+305A116, 1 -> 0): with the
  original exe, "Game anti-aliasing: Off" does the same in memory; the log says which state the exe is in. The
  FuckTAA "resolution fix" (the internal render size rounded down to 64 px - 3420x2668 became 3392x2624 and was
  scaled up) is now done in memory: "Exact render size" (Graphics, on); an exe that already has it is recognised.
  tools/verify_signatures.py reports both.
- **First person**: exe+17850D0 is the game's own "local hero" lookup - it takes the local player controller (what the
  mod got so far) and returns the character it controls. Used when the first lookup gives the controller.

## 0.6.14-alpha (the game's own projection; headset-size rendering; HUD layer after reloads; edge fill)

From the 0.6.13 log and a look at the exe:

- **The display was wrong because of the game's own projection.** The renderer builds the main view in two places:
  the view update (through the projection getter the mod patched) and the render-view setup (exe+98DC70, called from
  exe+A703BD / A71CAC), which copies the camera node's projection (node+0xC0) directly. That copy still had the
  game's own field of view and a symmetric frustum - why the picture never quite matched the headset, why the
  off-centre terms "were ignored", and why ~10 % larger picture width / height felt right. The node projection itself
  now carries the headset frustum (depth rows and TAA jitter kept, the game's own put back when VR is off); the log
  shows the game's own FOV next to the mod's.
- **Rendered at the headset's own size and shape** (3420x2668 on the Dream Air, x render_scale) in every method.
  With the node projection honoured, each eye gets its own asymmetric frustum, so alternate eye / synchronized need
  nothing wider (13.4-14.3 MP before -> 9.1 MP). Head mode and reconstruction's centre view fit the both-eye view
  into the same size; the rebuilt eyes come out at the headset's size. No resize ever happens on mode switches.
- Picture width / height reset to 1.00 once (they made up for the bug above and would now squeeze the picture).
- **HUD scale after dying / reloading**: the picked UI element is remembered by what it is (size, format, order -
  kept in the config as ui_pick_sig) and found again among the targets the game makes after a reload; stale entries
  in the 16-entry list are replaced. The automatic UI layer accepts a full-screen layer at any number of draws (the
  HUD layer has 4-15 draws - under the old 12 minimum after a reload) and is searched every frame sample.
- **Edge fill** (Screen tab, on): where the headset sees past the game picture, the picture's rim is stretched
  outward with a blur instead of black (as in the AC VR mods).
- **First person**: the character is looked for in more places from the controlled entity; if it is still not found
  the log lists what that entity is made of.
- **Performance**: the memory-readability checks (VirtualQuery, slow as the address space fills - the camera / menu
  time grew to 5.5 ms in the log) are cached; the report splits that time into camera, menu + input and start-up
  checks, and counts the memory checks.

## 0.6.13-alpha (picture fixes in the View tab)

- The Reconstruction tab is gone; its settings are back in View (shown while stereo reconstruction is selected).
- New, to fix the picture by hand while you look at it:
  - **Reconstruction: depth strength** (recon_parallax): how far apart the two rebuilt eyes are, x your eye distance.
  - **Reconstruction: eye align sideways / up-down** (recon_align_h / _v, degrees).
  - **Eye align sideways / up-down** for alternate eye and synchronized (aer_align_h / _v, degrees).
  - **Picture width / height** (view_width / view_height, all 3D methods): how much of the rendered picture the
    headset shows. If the world looks too wide, raise the width above 1.
  The rendered view grows to cover the shifted / larger window, so no edge goes missing.

## 0.6.12-alpha (the real causes: depth direction, stretched pixels, controller entity, UI viewport)

From the 0.6.11 log and screenshots:

- **Reconstruction eyes offset / crossed**: the eyes were not canted (the log measured 0.0 deg). Both rebuilt eyes
  were pushed apart by ~20 deg, near and far alike, with ~10 % "gaps stretched" in every scene: every pixel was
  rebuilt as if it sat at the near plane. The depth buffer runs the other way (reversed / standard Z) than the
  projection the game hands out. The mod now measures which way the depth runs on the depth itself and turns it
  round when needed (log: "recon: depth runs ..."). Reconstruction tab: "Depth direction" (auto / as the game says /
  turned round) to force it.
- **Wide picture in every mode**: the eye frustum was padded to the back buffer's shape; the back buffer grows to the
  wide symmetric view, so the frustum grew with it (1.70 -> 2.05 -> 1.99) and the render size was computed against
  the padded frustum - pixels 1.21x wider than tall. The frustum is now the headset's own field of view, and the
  render size's height is made from the drawn view's exact shape (square pixels). The eye_aspect option is removed.
- **First person**: the game's "local character" is the player controller entity on this build (all 10 of its
  interfaces are ZPlayerController). The character is now taken from what the controller controls.
- **UI scale had no effect**: 1200 UI draws a frame were found but none moved - their viewport was not seen on the
  command list, so the scaling was skipped (and the 3D picture's own pass was never recognised either). A draw
  without a seen viewport now uses the whole target. The report line says how many draws were placed and why not.
- **Reconstruction UI**: the centre view splits the UI from the picture too - the HUD is copied flat into each eye at
  ui_depth instead of being warped with the scene.
- **Performance**: the menu's gamepad poll read all four XInput slots every frame (an empty slot costs ~0.3-1 ms);
  only the connected slot is read now, empty slots once a second.

## 0.6.11-alpha (centre-only reconstruction, one frustum for every method, Reconstruction tab)

- **Stereo reconstruction is centre view only**: the left / alternate drawn views are gone (misaligned on canted
  headsets; alternate's "fill from last image" was the flashing see-through patch). Old recon_eye /
  recon_history / recon_shared_frustum lines in the config are ignored.
- **Wide picture, audited**: every stereo method now draws the SAME symmetric view (the widest of both eyes and the
  head-centre view), and the render size is made from exactly that view (no 2 % rounding). The picture's shape always
  equals the render size's shape, so the 3D cannot be stretched whether the engine takes its x scale from the mod or
  from its window shape; each eye is cut out 1:1 at the headset's size (0.6.9). Unit-tested.
- **UI scale across mode switches**: with one view and one render size for all methods, a switch no longer resizes
  (the game's targets - and the UI picker's pick - stay), and the UI's size in your view is the same in every method.
- **Reconstruction tab** (second tab): gap reach and "show hidden areas".
- **Performance**: the 10 s report now splits the mod's CPU time at Present into camera / menu, eyes (copy,
  reconstruction, blits) and sending, to find the 11 ms seen in the 0.6.8 centre-view log.

## 0.6.10-alpha (stereo reconstruction centre view: canted eyes)

- The 0.6.8 log + screenshots of the centre view showed the cause of the misaligned eyes: the Pimax runtime reports
  the two eyes turned apart (canted), the centre view was drawn facing like the left eye, so the right eye looked past
  the drawn picture's edge - a constant 22 % band of that eye was stretched ("gaps stretched 11.1 %" = the average of
  both eyes, the same in every scene).
- The centre view now faces halfway between the eyes and is widened to cover both turned eyes (unit-tested: 2 x 6 deg
  canting covered exactly). The frusta are recomputed whenever the angle between the eyes changes, from live poses;
  the log prints it ("headset: the eyes are turned N deg apart").
- Alternate (per-eye) reconstruction is unchanged: its flashing see-through patch is the "fill from last image"
  history; use the centre view (default), or turn "Reconstruction: fill from last image" off if you use alternate.

## 0.6.9-alpha (headset-shaped eye images; aligned reconstruction; UI picker keeps its pick; tidier menu)

- **Eye images are the headset's own size and shape** (e.g. 3420x2668 on the Dream Air, x render_scale) in every
  stereo method. The game still renders a slightly wider view; each eye is cut out of it 1:1 and sent at the headset's
  resolution. No more wide 4446x2882-style eye images.
- **No resize when you switch stereo methods.** The render size is set once from the widest view any method needs, so
  switching (F8) no longer recreates every render target (hitch) or loses the UI picker's target.
- **Stereo reconstruction aligned**: the head-centre view is now drawn with the head's orientation halfway between
  the eyes and covers both eyes' real (canted) views; both eyes are rebuilt with their own orientation. Existing
  configs move recon_eye to centre once (left / alternate still selectable). The log now prints the eyes' cant angle.
- **UI picker survives mode switches / resizes**: the picked target is remembered by shape, format and order and is
  found again among the game's recreated targets.
- **VR panel**: the View tab now has the method, quick actions and the stereo picture first; settings of one stereo
  method only show while that method is selected. Flat-screen rules moved to the Screen tab.

## 0.6.8-alpha (performance audit against the VR Performance Manual; reconstruction centre view; UI picker)
Audited against the framework's VR Performance Manual (W2VR). The D3D9 managed-pool sections (6.x) do not apply to
this D3D12 game; everything else was checked.

- **Pacing (manual 9.1 - 9.3)**
  - Refresh reading per runtime. "A rate under 66 Hz is a halved (smoothing) rate" is now applied only on Pimax
    OpenXR, which reports the halved rate from the start when Smart Smoothing is forced on.
    - On Virtual Desktop / Quest / SteamVR / WMR, 60 Hz is a real setting. Before, it was doubled to 120: the cap was
      too high and, in alternate-eye modes, the eyes uneven.
    - Smoothing there is detected as a drop from the full rate seen. One line says which rule is used:
      `pacing: runtime '...' - ...`.
  - A faster rate than the known one is taken only after 30 consistent reports (within 3 %). Reports under 30 Hz or
    over 250 Hz are ignored (runtime noise, stalls).
  - The cap in VR:
    - Alternate eye / synchronized: exactly the headset's refresh, whatever `fps_limit` says.
    - Reconstruction / head: a lower cap is allowed; a higher one or "off" is clamped to the refresh.
    - Flat: the user's value.
    - Logged once when overridden.
  - Never below the full refresh while smoothing (Pimax Smart Smoothing, ASW, SSW, SteamVR Motion Smoothing): kept.
- **Never wait (manual 9.5)**: the game's thread could wait up to 1 s at Present when an interop ring slot was still
  in use on the GPU. Now a free slot is taken (ring 3 -> 4), or the frame is skipped and counted (`ring busy`).
- **Measurements (manual 4)**, one line every 10 s: `perf: mod work at Present X ms avg, Y max | longest game frame Z
  ms | GPU frame interval A ms avg, B max | ring busy N`.
  - The GPU interval comes from timestamps on the mod's own copy, read back only after the GPU has run them.
  - Hitches show up as maxima.
- **Hook cost (manual 12)**: the UI-layer bookkeeping (interlocked counters shared by every recording thread, on every
  draw / bind / ExecuteIndirect) now runs on one frame in eight only.
- **Stereo reconstruction: the head-centre view (`recon_eye = centre`, new default).**
  - The engine always draws a symmetric view.
    - Centred on the left eye, it had to reach the right eye's outer edge: about 1.56x the headset's width in
      0.6.7. That was the "wide" picture, and expensive.
    - Centred between the eyes it needs about 1.3x, the same as alternate eye.
  - Both eyes are rebuilt from that centre view. Each is only half the eye distance away, so the gaps behind close
    objects are half as wide.
  - Each rebuilt eye is made at its own view only, at the drawn picture's pixel density: no pixel is computed that
    the headset does not show.
  - Configs on 0.6.4's "left" are moved to centre once. `left` and `alternate` are still there.
- **Head mode**: its both-eyes view is no longer padded to the eye image's shape (it was 1.34 x 1.34, about 1.8x the
  pixels; now about 1.3 x 1.05). The UI keeps its size in both head mode and centre reconstruction.
- **UI**
  - **UI element picker** (HUD tab, `ui_pick`).
    - 0 = automatic. 1, 2, 3 ... = each display-format target the game draws into that is not the scene. Each is
      logged once as found (`ui: picker candidate N: WxH format F`), and the status line shows how many were found.
    - Step through them in game until the HUD shrinks to the UI scale: that one is the HUD.
  - UI scale default 60 % (an untouched old 0.62 moves to 0.60).

## 0.6.7-alpha (eye screenshots: symmetric view; UI; VR panel; launcher .bat)
- **Alternate eye / synchronized: the two pictures now line up.**
  - Cause: the 0.6.6 screenshot pair of alternate-eye mode. Both eyes were centred on the same point, but the Dream
    Air's eyes look about 13 % of the picture to each side (left eye tan -1.67 .. 0.98, right -0.98 .. 1.67).
  - The game renders a symmetric projection whatever the mod puts in the off-centre terms: its projection is
    rebuilt after the mod's patch (depth rows, TAA jitter in the same two terms). In the headset each eye's picture
    sat shifted to its side, so the two never merged.
  - Fix: the engine now always draws a symmetric view that covers the eye's real one (the AC mods' "symmetric
    view"). The mod cuts each eye's own view out of it (0.6.6's `submit_view = headset`). `symmetric_view = false`
    goes back.
- **Stereo reconstruction looked too wide**: same cause. Its shared view was off-centre and drawn centred. It is
  symmetric now too, and both eyes are cut to the headset's own view.
- **The semi-transparent flat copy flashing in front (reconstruction)**:
  - The UI-layer search had picked an HDR scene target (format 26, R11G11B10). Its draws were scaled like UI, which
    put a smaller copy of the scene in front.
  - The UI layer must now be an 8 / 10-bit display format; HDR targets are never taken.
- **UI scale**:
  - The 0.6.6 log showed 34 2D pipelines but 0 captured draws. Under 4 % of the game's target binds were known,
    because the mod only hooked pipeline and view creation when the swap chain came. The pipelines and views the game
    made before that (its UI canvas, its cached pipelines) were never seen.
  - The device and command-list hooks are now installed at the game's first DXGI factory, with a short-lived device
    of the mod's own, so they are in place before the game makes anything.
  - A 2D draw into a view the mod does not know uses the viewport as its size.
  - The UI canvas may be a fixed size such as 1920x1080, smaller than half the 4514-pixel eye image: any target from
    1280x720 up now counts.
- **VR panel**:
  - Five tabs instead of eight: View (view, stereo, reconstruction), Camera (camera, first person), HUD, Graphics
    (graphics, speed, foveated), Screen (screen, start-up).
  - The build number is gone, the text is smaller (26 px), rows are tighter, and the panel is shorter.
- **Start_GOTG_VR.bat** (next to dxgi.dll): starts GOTG.exe with `-nolauncher` (through Steam if it is not in the bin
  folder).

## 0.6.6-alpha (one dxgi.dll, no host exe; reconstruction, first person, UI, field of view, from the 0.6.5 Pimax log)
- **The mod is now `dxgi.dll`, and there is no XrBridgeHost.exe any more.**
  - The OpenXR host runs as a thread inside the game: the same host code, compiled into the DLL with the static
    OpenXR loader.
  - It no longer sets the process priority or the process GPU scheduling class, because that would now hit the game.
    Its own thread priority and its D3D11 device's GPU priority stay.
  - Its log is still `GOTGVR\xrhost.log`.
  - Loaded as dxgi.dll, the mod takes the game's own CreateDXGIFactory calls and installs its swap-chain hooks before
    the first factory is made. Every other export is forwarded to the system dxgi.dll.
  - `alt\winmm.dll` (the old install name) and `alt\GOTGVR.asi` are the same mod.
- **Stereo reconstruction did nothing in 0.6.5.**
  - Cause: the shared depth texture was refused, `recon: shared texture ... fmt 27 failed 80070057` in every log
    line. The right eye kept a frozen image.
  - Fix: NVIDIA also refuses RGBA8 *typeless* shared textures. Depth now travels as RGBA8 *UNORM* (the format the
    eye ring already shares fine), and the shader rebuilds each byte exactly. The picture copy no longer uses a
    typeless format either.
- **"Everything looks wider"**: each eye image is now cut to the headset's own field of view before it is sent, and
  sent with exactly that field of view (`submit_view = headset`, default).
  - The game draws wider than the headset shows: reconstruction's shared view is 1.3 x 1.08 on the Dream Air, plus the
    cull margin. That wider field of view went to the runtime with the image.
  - A runtime that does not honour a layer's own field of view squeezes such an image into its own: everything looks
    smaller and wider. Pimax OpenXR already ignores quad layers.
  - `submit_view = rendered` sends the whole image as before.
- **UI / HUD: 2D pipelines, as the AC mods do with 2D shaders.**
  - The log showed why nothing was found before: the scene depth got only ~21 draw calls a frame. GOTG renders
    GPU-driven, so no UI layer reached the old 12-draw threshold.
  - Now every pipeline the game creates is classified when it is made: plain descriptions, pipeline streams, and
    pipeline libraries. A pipeline with no depth test, blending on target 0 and one display-format target is 2D (UI /
    HUD).
  - Every draw with a 2D pipeline into a screen-size target is scaled and placed with the one UI scale / position
    (`ui_2d`, default on). Full-screen passes are left alone.
  - ExecuteIndirect is counted as drawing too.
  - Log line: `ui: 2D pipelines N of M ... | 2D draws into screen-size targets ..., moved ... | indirect draws ...`.
- **First person**: the 0.6.5 log found the player entity but `no ZCharacter among its 10 interfaces`. Interfaces
  are now matched through the MSVC RTTI class hierarchy: a class *derived from* ZCharacter, or a ZAnimatedEntity
  directly. If it still fails, the log names the class of every interface of the player entity.
- **Camera offsets**: new **Offset step (cm per press)** slider (Camera page, `offset_step_cm`, default 3 cm =
  3x the old 1 cm). It sets the step of all six camera and first-person offset sliders. Their ranges are 3x wider
  (camera X / Y +-9 m, Z +-15 m; first person +-1.5 m).
- **Alternate eye / synchronized "two pictures in each eye"**: not reproduced from the log, so 0.6.6 collects the
  proof.
  - `Delete` now saves both eyes as sent to the headset (`..._left.png`, `..._right.png`).
  - The log lists the order in which eyes were drawn (`xr: eyes drawn ...: LRLR...`).
  - One AER screenshot pair shows whether the double image is already in the eye image (the game's temporal
    passes) or comes later.

## 0.6.5-alpha (you choose: the mod changes no game option on its own)
- **The mod no longer switches anti-aliasing, upscaling or resolution scale by itself.** 0.6.4 turned them off
  automatically, depending on the stereo method. Now there are two explicit choices on the VR panel's Stereo page:
  - `game_aa = game | off | on`: the game's anti-aliasing option.
  - `game_upscaling = game | off`: off = upscaling / dynamic resolution (DLSS, FSR) off and resolution scale 100 %.
  - Both default to `game`: the mod never touches the option, whatever the stereo method. The choice applies in
    every mode until you change it.
  - The renderer's TAA flag is now held at 0 only with `game_aa = off`.
  - `taa_off` and `vr_native_res` are gone.
- **Motion blur and radial blur** stay as the game has them by default (`motion_blur_off` / `radial_blur_off` =
  false). Existing configs are moved to that once (`config_version = 4`). Depth of field stays off by default, as
  you asked earlier.
- `recon_eye` is no longer rewritten by an update. 0.6.4 had set it to `left` once; set it back on the Stereo page
  if you want `alternate`.
- README rewritten: shorter, with a credits table for everything used.

## 0.6.4-alpha (double image in AER / synchronized, native resolution, cheaper reconstruction, UI layer)
- **AER and synchronized showed two pictures side by side in each eye.**
  - Cause: the game's temporal anti-aliasing / upscaler. In those modes every frame is the other eye, and TAA, DLSS
    and FSR 2 blend the previous frame into the new one, so each eye carried a shifted copy of the other.
  - The old `taa_off` only held the renderer's jitter byte (exe+305A116), which turned out to be the TAA flag only.
    DLSS / FSR / dynamic resolution were untouched. Reconstruction drawing the same eye every frame did not show it.
  - Fix: the mod now changes the game's own options, through the setters the game's options menu uses (the option
    table exe+3DE1C70; see NOTES "Render options"):
    - Anti-aliasing (option 0x4BE) goes off while the eyes alternate: AER, synchronized, and reconstruction with
      `recon_eye = alternate`.
    - Upscaling / dynamic resolution (option 0x51B, the game's "DynResType") goes to 0 = off, and resolution scale
      (0x51A) to 100 %, in every VR mode (`vr_native_res`, on by default).
  - Your own settings are only read, never written, and come back in screen mode. If you change graphics options
    in the game while in VR, the mod applies its values again.
  - The log shows `render options: the game's settings - anti-aliasing .., upscaling .., resolution scale ..` and
    each change it makes.
- **Native resolution**: with upscaling off and resolution scale at 100 %, the game draws every pixel of the eye
  image itself. Before, DLSS / FSR / dynamic resolution could render at a fraction and upscale, which also explains
  "the resolution looks lower than it should".
- **Stereo reconstruction: native resolution, less work.**
  - `recon_eye = left` is now the default, and existing configs are moved to it once (`config_version = 3`). The
    left eye is drawn every frame, so the game's TAA keeps a valid history: full image quality, no ghosting, no
    aliasing. The right eye is rebuilt from it.
  - The shared view only grows where the other eye needs it. The two eyes differ mostly sideways, so it is now about
    1.3x wider but barely taller.
    - Before, it kept the eye image's shape, so it was also 1.3x taller: about 1.7x the headset's pixels to draw.
      Now it is about 1.3x, at the same pixel density.
    - The eye image follows per axis (`resolution: rendered view 1.30 x 1.03 the headset's`), so pixels stay
      square.
- **Why hide HUD worked and UI scale did not**: hide HUD flips the game's own HUD flag. UI scale has to find the
  UI's draw calls and move them, and GOTG never draws its UI into the final image. It draws the UI into a layer of
  its own, which a compute pass blends into the picture before it is copied to the screen (`3D 0, UI 0`).
  - 0.6.3's search for that layer was fragile. It required "no depth buffer bound" (UI masks can bind a stencil
    buffer) and "bound last" (the game records command lists on several threads, so bind order says nothing).
  - Now the layer is the colour target with the most draws that is never drawn with the scene's depth buffer, with
    8-bit targets preferred.
  - Tested with a D3D12 test app that draws its UI into its own layer with its own stencil buffer: the layer is
    found and all its draws are taken as UI.
  - New log line: `ui: UI layer in use | not found | last frame: N targets, scene depth .. draws, best UI candidate
    .. draws (format ..)`.
- **Compared with DXMDVR** (the Deus Ex mod):
  - DXMD draws its UI straight into the back buffer with known Scaleform shaders, so DXMDVR could split it there.
    GOTG needs the layer route above.
  - DXMDVR switched TAA and motion blur off through the game's registry settings at start. GOTGVR does it live,
    through the option table, and only in the modes that need it.
- New settings: `vr_native_res` (default true). `taa_off` now means the game's anti-aliasing option, and only while
  the eyes alternate.
- `scripts/build_tests.sh` builds the camera, flat screen and window unit tests.

## 0.6.3-alpha (second Pimax log: stereo reconstruction repaired and the default, UI layer, density, hitches)
- **Stereo reconstruction was not working, and is now the default stereo method** (`vr_mode = recon`).
  - Cause: the shared depth copy could not be created on NVIDIA. The log showed `recon: shared texture 3420x2668
    fmt 42 failed 80070057` and `reconstructed 0`. D3D11 does not open a D3D12 R32_UINT texture as a shared
    resource there.
  - Effect: the "other" eye kept an old image every frame, so each eye only updated at half the rate. That looked
    like head-tracking hitches.
  - Fix: the depth now travels as the same bytes in an RGBA8 (RG8 for 16-bit depth) texture, which every driver
    shares, and the shader rebuilds the value. The shaders compile with the real d3dcompiler_47.
  - Existing configs are moved to `vr_mode = recon` once (`config_version = 2`).
- **Head tracking hitches**: besides the reconstruction fix, the bridge no longer waits for the game inside a
  headset frame unless the game is clearly faster than the display, and then at most 30 % of a frame.
  - The xrhost log showed late frames (27 ms intervals, up to 17 per 10 s) whenever the game ran at 60-69 fps on
    the 72 Hz headset. Those frames are now shown at once with the newest image.
- **Resolution / FOV**: the eye image now keeps the headset's own pixel density over its field of view
  (`render_density = headset`).
  - Before, stereo reconstruction drew each eye wide enough to hold both eyes' views (about 1.3x the headset's
    FOV), and a cull margin widened it too, but the image stayed 3420x2668. The part you see had about 25 % fewer
    pixels per degree in each direction.
  - Now the eye image grows with the rendered view (e.g. 3420x2668 -> about 4440x3460 in reconstruction), so the
    headset gets its full resolution. Use `render_scale` if that costs too much, or set
    `render_density = fixed` for the old behaviour.
  - `cull_margin` stays 1.0 by default: the rendered frustum is exactly the headset's FOV.
- **UI / HUD scale and position**: the game draws its UI into a layer of its own and blends that into the picture
  later. The log showed `3D 0, UI 0`: no UI draw ever reached the final image, so UI scale did nothing.
  - The mod now finds that layer: a target with no depth and many draws, bound last, while the final image gets no
    draws. It then moves and scales the UI draws there (`ui_target = auto | off`).
  - Views copied between descriptor heaps (CopyDescriptors / CopyDescriptorsSimple) and D3D12 render passes
    (BeginRenderPass) are now followed too.
  - A new log line, `ui: target binds ... (known, final image, render passes), view copies`, shows what the game
    does.
- **First person**:
  - The player search logs the step where it stops (`firstperson: player search stops at step N: ...`), so the next
    log shows what is missing.
  - New fallback: the local player controller (exe+1785560) and its controlled character.
  - New fallback: any interface whose vtable has ZCharacter's GetAnimatedEntity, for derived character types.
- **Cheat Engine speed hack**:
  - Every export stub of the winmm proxy now has 16 bytes of its own. The speed hack patches timeGetTime with a
    long jump; before, that overwrote the next export's stub.
  - The mod's own timers (resource retirement, menu input, menu animation, foveation rebuilds) now use the
    interrupt-time clock, which the speed hack does not change. GPU resources are no longer freed early at high
    speeds, and the FPS limiter and pacing keep real time. The game itself speeds up or slows down as intended.
- Foveated rendering was running in that log (`passes with reduced shading 5865`). It only affects draw passes
  into eye-size targets. Lighting and post effects that the game runs as compute shaders are not affected, so the
  gain depends on the scene. `fov_debug = true` shows the outer zone.

## 0.6.2-alpha (fixes from the first Pimax Dream Air log)
- **Fixed: the picture filled only the top 80 % of each eye, then colour streaks and a crash.**
  - The game's window was kept inside the 2160-line monitor, so the game rendered 3420x2153 into 3420x2668 back
    buffers. The log showed `3420 x 2153 requested`.
  - Its final whole-frame copy into the back buffer (CopyResource) then had mismatched sizes. That is invalid in
    D3D12: the top part arrives, and the rest is GPU memory garbage that ends in a device removal.
  - Now the game is told the eye size every way it can ask: GetClientRect is hooked for its window, WM_SIZE carries
    the eye size, and the window size is put back after the game's own WM_WINDOWPOSCHANGING handler too.
  - The copy into the back buffer is cut to the common area if the sizes ever differ again. That case is logged as
    `swapchain: the game copies a ... frame into the ... back buffer`.
- **Fixed: the VR menu did not show, and loading / menu screens were black (Pimax OpenXR).**
  - The menu and flat screen were OpenXR quad layers, and that runtime does not show them. A frame with no projection
    layer showed black.
  - The host now always submits a projection layer, also for menus and loading screens.
  - New `panel_layers` setting (*Screen* page):
    - *composite*: the VR menu, HUD and flat screen are drawn into the eye images by the host. This works on every
      runtime.
    - *quad*: separate layers, slightly sharper where the runtime supports them.
    - *auto* (default): composite on Pimax, quad elsewhere.
- XrBridgeHost.exe rebuilt (OpenXR loader 1.1.43). Shared-memory protocol unchanged (v6; `panelMode` uses a reserved
  field).
- Verified:
  - `tools/window_test.cpp` under Wine on a 3840x2160 screen, with a game window that clamps itself to the monitor:
    the window, GetClientRect and WM_SIZE are all 3420x2668.
  - `tools/host_panel_test.cpp` runs the host's panel drawing on a real D3D11 device: edges land on the exact
    projected pixels, the transparent part blends, and a panel 30 degrees to the left lands left.

## 0.6.1-alpha (no launcher, always windowed)
- **The launcher is skipped** (`skip_launcher`, default on): the game goes straight in. Its argument parser
  (exe+101E50) turns `UseLauncher` off for `-nolauncher`, and the mod adds that switch before the game reads its
  command line, through every route the game can read it:
  - kernelbase GetCommandLineA / W are hooked;
  - ucrtbase's `_acmdln` / `_wcmdln` and its already-built argv are updated;
  - the process parameters are updated too, which covers Wine / Proton.
- **Always windowed** (`force_windowed`, default on), so the window and the rendering can be the headset's per-eye
  resolution and shape, larger than the monitor:
  - The game's `Fullscreen` / `ExclusiveFullscreen` settings (HKCU\Software\Eidos Montreal\Guardians of the Galaxy,
    DWORDs) read as 0 through RegQueryValueExA / W and RegGetValueA / W hooks. This only applies to keys under
    "Eidos Montreal", and nothing is written.
  - The swap chain is windowed from the very first one, even before the headset size is known; before, this only
    happened once it was known. Exclusive fullscreen is refused, Alt+Enter is off (DXGI_MWA_NO_ALT_ENTER), and a swap
    chain that was already fullscreen is switched back to windowed.
- Both are on the VR menu's *View* page and take effect at the next start; `vr_mode = screen` keeps the game's own
  display mode.
- Verified under Wine:
  - `tools/startup_test.cpp`: GetCommandLineA / W, the CRT command line and argv all end in `-nolauncher`.
    Fullscreen = 1 reads as 0 (A, W, RegGetValue); a missing ExclusiveFullscreen reads as 0; other values and keys
    outside "Eidos Montreal" are untouched; the stored value is still 1 afterwards.
  - The built winmm.dll in a test `gotg.exe` that starts its CRT the way GOTG does (`tools/startup_ucrt_app.cpp`):
    ucrt argv = [exe] [-foo] [-nolauncher], WinMain line "-foo -nolauncher", and ExclusiveFullscreen stored as 1
    reads as 0.

## 0.6.0-alpha (first person, PNG screenshots)
- **First person (`End`)**: the VR camera in Star-Lord's head. Found in the exe (EXE_REPORT.md section 8):
  - the local player's character: exe+1784E90 on the ZPlayerManager instance exe+3E41B58;
  - its ZCharacter interface, by the type info read from ZCharacter's own QueryInterface (exe+1756650);
  - its animated entity: ZCharacter vtable slot 80 (exe+17425C0);
  - the head bone: exe+81E740, the native behind the `GetBoneWorldTransformQV` script function. The bone id is
    CRC-32 of the lower-case joint name (`jntbd_head` = 0x2A4C5889, exe+1093B0). An unknown id returns the character
    root, which the head bob filter uses.
  - The body is hidden with ZAnimatedEntity slot 76 (SetVisible, exe+593760), as the game's own
    ZCameraBehavior_FirstPerson does (exe+1569400).
  - Everything runs on the game thread inside the game clock hook, where the game itself calls these functions. The
    render thread only reads the result.
- **First-person settings of its own** (*First person* page): eye offsets `fp_offset_x / _y / _z`, head bob filter
  `fp_smooth`, `fp_hide_body`, and `fp_max_distance` for cutscene detection. `camera_offset_*` are now third person
  only.
  - The bob filter is two-stage and works relative to the character root: running is never delayed (measured 0.0 mm
    lag at 5 m/s), and a 4 cm 2 Hz bob is cut to 1.2 cm.
  - The camera shake filter leaves the head position alone in first person.
- **Cutscenes / scripted cameras**: first person waits when the game camera does not follow the player, the body is
  shown again, and first person resumes on its own.
- **Screenshots are PNG** (`Delete`): the left eye cut to the headset's own field of view, 1:1 pixels, saved next
  to `GOTG.exe` (`GOTGVR_<date>_<time>_left.png`). `screenshot_view = full` saves the whole rendered eye.
  - The staging copy is read one frame later (no GPU stall), and a worker thread encodes the PNG (WIC), so a shot
    does not cost the headset a frame.
- **Keys**: `Space` + `F9` recentre, `Delete` screenshot and `End` first person (`fp_key`), all checked for
  duplicates. There is also a `first_person.trigger` file and a menu action.
- Verified:
  - verify_signatures.py and the C++ fp::Install, both run on the real exe image, find every first-person piece:
    the sites listed above, the ZCharacter type info exe+3D99E18, and the slot 80 and slot 76 bodies.
  - `tools/firstperson_unittest.cpp` covers the eye maths, the offsets, cutscene detection, the bob filter, the
    screenshot crop and the PNG writer.
  - `tools/screenshot_test.cpp` runs on a real D3D11 device: eye texture -> staging -> crop -> PNG. Pixels are exact
    at 950x1150 and the write takes 24 ms off-thread.
  - `tools/camera_unittest.cpp` drives the real camera hooks in first person: eye = head + offsets, no lag from the
    stabilizer, suspended for a far camera. All checks pass.

## 0.5.0-alpha (exe dismantled, depth of field always off)
- **Depth of field off everywhere - gameplay and cutscenes** (`dof_off`, default on). The game's two DOF passes
  (ZRenderPostfilterPassDOF and ZRenderPostfilterPassHexDOF, the bokeh DOF) report "not active" to the post filter
  chain (their IsActive, vtable slot 0), the engine's own "effect off" path - nothing half-drawn, no stale buffers.
  Cutscene DOF (ZPostfilterDepthOfFieldAspect) and photo mode DOF end in the same two passes.
- **Motion blur off** (default on), **radial blur off** (sprint / dash / impact zoom blur, default on), **film grain
  off** (default off) - the same way, live switches on the Display page.
- The passes are found by their RTTI class names (type descriptor -> object locator -> vtable), not code bytes, so
  game patches do not break them; a 7-instruction stub per pass (rax only) returns false or jumps to the game's own.
- **EXE_REPORT.md**: the whole exe dismantled - sections / protection (Denuvo-style .debug layer), all imports and
  middleware (Havok, Wwise exported by the exe, Bink 2, Umbra, NavPower, NRD, DLSS, FSR / CAS, libscepad, osdk),
  6266 RTTI classes by subsystem (221 camera, 38 render graph nodes, 43 render passes, 21 post filter passes with
  vtables / IsActive / record addresses), every IGCS Camera Tools site identified by RTTI (its labels are interleaved -
  the "DOF" patterns are DOF / HexDOF, its "hex DOF" / "glare" ones are Glare / RadialBlur), the camera / clock / UI
  address table, and research on the Dawn engine family (Glacier 2 fork; DXMD, Breach, the 2017 DXMD VR Experience;
  Hitman's later native VR is not in Dawn; Avengers is Foundation).
- Verified: the C++ RTTI lookup run against the real exe image resolves all five passes to the hand-found functions
  (and the camera node to exe+2571C98); the stub returns false when off, passes all arguments through when on, and
  switches live; verify_signatures.py checks the RTTI lookups too.

## 0.4.4-alpha (headset smoothing / ASW can never slow the game)
- **Fixed: a headset whose motion smoothing was on from the first frame halved the game.** The FPS limiter followed
  OpenXR's predictedDisplayPeriod, which Meta ASW, Virtual Desktop SSW, Pimax Smart Smoothing, SteamVR motion
  smoothing (forced) and WMR reprojection double while they run the app at a divided rate; the limiter kept the
  shortest period seen, so a session that never saw the full rate stayed at half (measured: 45 fps on a 90 Hz headset).
  Now the cap is the headset's real refresh rate:
  - a divided report is multiplied back (under 66 Hz on any headset - 60 / 45 / 40 / 36 / 30 -> 120 / 90 / 80 / 72 /
    90; the Valve Index under 79 Hz, so 72 -> 144; Reverb G2 / Vive Pro 2 / Varjo / Beyond / PSVR2 under 74 Hz),
  - the full rate is remembered per headset (GOTGVR\refresh.txt),
  - `headset_refresh_hz` (Display page) overrides it; in automatic mode the cap is never under 66 fps.
- **All three headset_smoothing modes keep the game at full rate.** The game never waits for the host or the runtime
  (slots are skipped, not waited for); the modes only decide what the headset shows between game frames. "runtime" is
  now "Headset smoothing fills in" (the headset's own smoothing synthesises frames when the game is slower) and runs
  as "full" with synchronized pairs (a pair every second game frame would otherwise keep the headset's smoothing on).
  The mode follows stereo method changes at once.
- Verified end to end under Wine with the real mod DLL, the D3D12 test app and a fake headset host that reports a
  doubled period the way smoothing runtimes do: Quest 3 at 90 Hz with smoothing forced from the first frame -> 89-91
  fps in AER / full, synchronized / runtime, reconstruction / repeat (0.4.3: 45 fps); 90 -> 45 -> 90 Hz mid-session ->
  90 throughout; 120 Hz headset -> 120; Valve Index 144 Hz reporting 72 -> 144; Quest 72 Hz -> 72.
  `tools/refresh_unittest.cpp`: 18 cases pass.

## 0.4.3-alpha (3D / head-tracking audit, more exe research)
- **ipd_scale no longer lies to the headset:** the images are drawn with the scaled eye distance, but the pose sent with
  them is the real eye pose (before, the runtime got eye positions that were not yours). Also for stereo
  reconstruction's rebuilt eye.
- **The UI's depth uses your real IPD** (ipd_scale changed where the UI appeared).
- **Camera offsets are metres of the game world** at any world_scale (world_scale 2 doubled them); the shake filter's
  cut threshold too.
- **Near clip** (Camera page, `near_clip`, metres, 0 = the game's): lowers the near plane while VR frames are made so
  leaning in does not cut walls / Star-Lord open. Uses the game's own near-plane override global exe+3D97774, read by
  the projection getter (exe+9722CD) and exe+9724C0; restored when VR stops.
- Log: the main camera's near / far planes every 10 s (units check).
- Research (NOTES.md): render node rows are right / up / back (exe+990D40: the view update builds the rigid inverse of
  node+0x24 with the right-handed projection) - the default camera_axes +0+1+2 is right; the head-tracking mod's
  right / forward / up layout is the gameplay camera entity (before the reflector), not the render node. World is
  Z-up and in metres (gravity 9.81 stored by game code at exe+20E892D, Havok physics, DXMDVR's 1.0 on the same
  engine). The projection getter builds standard (not reversed) depth. exe+9724C0 is a second view x projection builder
  (LOD screen size only, never shown).
- Verified by `tools/camera_unittest.cpp` (all checks pass): world_scale 0.5 / 1 / 2 -> eye separation 3.2 / 6.4 / 12.8
  cm, ipd_scale 1.5 -> 9.6 cm drawn and the real pose sent, camera offset 0.5 m at world scale 1 and 2, decoupled pitch
  on (horizon level) / off (follows the game's 30 degree tilt), swap eyes, cull margin (frustum and projection), near
  clip in units, shake filter (+-1 -> +-0.1, a 3 m cut at once).

## 0.4.2-alpha
- **Flat virtual screen for menus and video cutscenes**, each a toggle on the View page:
  - `flat_pause_menus` (default on): while the game is paused - pause menu, settings, map, gear, codex - the game is
    shown on the flat screen. Uses the game's own pause flag (the dl argument of the clock update exe+42E890, computed
    at exe+D2470 from the pause manager) plus a still game camera; photo mode (paused, camera moving) stays 3D.
  - `flat_videos` (default on): pre-rendered Bink 2 cutscenes. bink2w64.dll's BinkDoFrame / BinkDoFrameAsync /
    BinkDoFramePlane / BinkClose hooked; a video at least `flat_video_min_width` (1280) wide decoding while the game
    camera stands still counts. In-world screens and real-time cutscenes stay 3D.
  - `F6` / "Flat screen now": hold the flat screen by hand.
  - While flat, the game renders its own camera (no eye pose, no UI move, no clock hold, no reconstruction) and the
    mod's VR panel shows FLAT (paused / video / F6). Switching waits 0.25 s either way, so it never flickers.
- Verified: `tools/flatscreen_unittest.cpp` drives the real camera hooks in real time - pause -> flat in ~0.5 s with the
  game camera untouched, unpause -> VR, photo mode stays VR, video + still camera -> flat, video while moving -> VR, F6,
  each toggle off / on. `tools/camera_unittest.cpp` (all modes) still passes; Wine / vkd3d run clean.

## 0.4.1-alpha (audit)
- Stereo reconstruction fixes found in a full audit:
  - a depth copy could be recorded with a depth of another size than the copy targets (depth replaced while the copy
    set was rebuilt, or the rebuild failed) - an out-of-bounds GPU copy / device removed. Each copy set now knows its
    depth, every copy checks the real size first, and the mode only runs when set and depth match.
  - after a resolution change the old (freed) depth could be touched by the fallback copy: the fallback now needs the
    depth to be used this frame and its state known from the game's own barriers (never a guessed state); a new depth
    is taken over at once.
  - the UI split no longer turns itself off for good when a full-screen menu fills the picture: off for 20 s, then
    tried again.
  - the game queue's wait for the next copy slot is no longer skipped on frames that leave the compositor early.
  - picture-without-UI copies work with an sRGB final target (typeless copy), and a final target that changes from
    frame to frame no longer rebuilds the copies every frame.
- Verified, all modes: `tools/camera_unittest.cpp` drives the real camera hooks (view update -> projection getter ->
  post filter, game clock) with a fake renderer and headset: screen (camera untouched), head (mono centre), AER
  (alternation, 3.2 cm per eye, per-eye frustum on every getter call), synchronized (clock held for every second eye,
  pairs from one camera although the game camera moved, pair flags), reconstruction (exact projection capture, 6.4 cm
  eye separation, shared view covers both eyes at the image shape, left-only with jitter) - all checks pass.
  `tools/d3d12_testapp.cpp` under Wine / vkd3d: 900 frames + a resize in each mode, depth copies only in
  reconstruction. Both compute shaders compile with Microsoft's d3dcompiler_47 without warnings.

## 0.4.0-alpha
- **New 3D method: stereo reconstruction** (`vr_mode = recon`, VR panel / F8). One eye is drawn per game frame and the
  other eye is reconstructed from that frame's picture and depth, so both eyes are new every frame, at the game's full
  frame rate, of the same moment (no AER tick mismatch, no held clock). Written from scratch for GOTGVR; the idea is
  the same family as PureDark's AFW (UEVR / REFramework) and the Dark Souls III native VR mod, no code taken from them.
  - Scene depth found automatically (the depth target bound with colour targets that gets the most draws; D32S8 in
    GOTG) and copied on the game's own command list right where the game finishes it (its DEPTH_WRITE barrier).
  - The picture without UI is copied before the first UI draw; UI pixels are moved to the other eye's UI position
    instead of being warped by depth (switches itself off if a later full-screen pass makes that impossible).
  - Shared view: the drawn eye is rendered wide enough to also hold the other eye's view (canted displays included),
    so the rebuilt eye has no missing strip. Hidden areas: the deeper surface beside them, or the rebuilt eye's own
    last real image reprojected (`recon_history`).
  - `recon_eye = left` draws only the left eye: TAA / DLSS history stays valid, `taa_off` is not applied.
  - Debug views (`recon_debug`): hidden areas tinted / picture grey with the UI in colour. Log line `recon:` every 10 s.
  - Checked: both compute shaders compile with Microsoft's d3dcompiler_47; a numerical model of the maths
    (`tools/recon_sim.py`) against ground truth: 99.9 % of the rebuilt eye has a direct source with the shared view
    (93 % without), 95 % of pixels within 0.7 px, also with reverse / standard Z, TAA jitter and 10 degree canting;
    the D3D12 capture side runs under Wine / vkd3d (depth found, copies every frame, survives a resize).

## 0.3.0-alpha
- **Every pass gets the eye's frustum.** Three more passes read the camera node with the projection getter besides the
  view update (skin blur post filter exe+8C6350, a post filter exe+8CA170, exe+9B3137). They got the game's own
  narrower FOV with the eye's pose - screen-space effects did not match the picture. They now get the same per-eye
  frustum (only while the mod's pose is in the node).
- **FPS limited to the headset's highest refresh rate whenever the headset is connected** (menus, loading screens and
  flat screen too, not only 3D frames); no monitor vsync then. A runtime that drops to half rate for its smoothing
  does not pull the cap down.
- **Responsiveness:** `max_frame_latency` (default 2) on the swap chain when the game uses a frame-latency waitable
  object (logged), so the head pose in a frame is as fresh as possible.
- **Camera shake filter** (`camera_stabilize`, default 0.35): a damped follower on the game camera's position and
  heading removes shake and bob (jumps, landings, hits); cuts are taken at once; the head pose is never filtered;
  the second eye of a synchronized tick uses the same filtered camera.
- **Fixed foveated rendering on D3D12** (Foveated page; variable rate shading tier 2: RTX 20+, RX 6000+, Arc): a
  shading-rate image per eye centred on its lens centre, on every eye-size pass, never on the final image / UI.
  New image sets are swapped in after the mod's copy list ran, old ones released seconds later.
- Research: Paul44's Cheat Engine table (FearLess), the Nexus "Simple Realistic 3D" ReShade / Depth3D preset
  (depth-based fake 3D), vorpX (unofficial). No public GOTG VR / camera source on GitHub. The settings table
  (exe+1CC1E40) maps names to setting ids, not struct offsets.

## 0.2.0-alpha
- **Synchronized (same tick) stereo** - UEVR's synchronized sequential for GOTG: the game clock (exe+42E890 on the game
  timer exe+3C2CDF8) is held on every second tick and the held time added to the next one; the second eye reuses the
  first eye's camera; the eyes are paired with ticks automatically (clock offset found from when the game camera
  stands still); both eyes are sent to the headset as a pair (or as drawn). Default method; alternate-eye stays.
- **Temporal AA off** (`taa_off`): the renderer's TAA / jitter flag exe+305A116 (read by the jitter exe+979FE0) kept
  at 0 while 3D frames are made, put back when off.
- **VR panel restyled** for Guardians: semi-transparent indigo / violet, neon magenta / cyan, starfield, synthwave
  grid, spinning cassette, pill tabs and switches. The DXMD title / colours are gone. Only options that work in GOTG.
- Research: camera chain traced from the IGCS / head-tracking hook to the render node; true single-frame stereo
  assessed (needs a second scene view per frame, exe+97FD30); StereoShutter checked (DX11 geo-11 only).

## 0.1.0-alpha (first build, untested in game)
- Port of DXMDVR 0.5.12 to Marvel's Guardians of the Galaxy (Steam exe 2023-01-26, D3D12).
- Stereo: alternate-eye rendering on the render camera node (view update + projection getter hooks), per-eye pose and
  asymmetric headset frustum, eye image shape headset / 1:1 / 5:4 / 4:3 at the headset's recommended resolution.
- D3D12 -> D3D11 interop (shared textures + fences) into DXMDVR's XrBridge host (any OpenXR runtime / layer).
- Headset smoothing defaults to "full rate": the headset's motion smoothing never halves the frame rate (DXMDVR 0.5.8's
  "always = half rate" mode is gone); FPS limiter at the headset's highest refresh rate.
- UI moved onto each eye's centre, scaled (ui_scale) and converged at ui_depth (D3D12 command list hooks).
- IGCS patch sites: no cutscene black bars, no pause on focus loss, game HUD toggle (F7).
- VR menu pages View / Stereo / Camera / HUD / Display / Screen; monitor mirror of the left eye; HDR -> SDR.
