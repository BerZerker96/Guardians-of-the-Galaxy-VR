# GOTG.exe - full dismantling report (GOTGVR 0.6.0)

Steam build linked **2023-01-26 22:30 UTC**, image base 0x140000000, 518.8 MB image, PDB name `gotg.pdb`.
Everything here was read from the exe itself (static analysis: pefile + capstone, `tools/exe.py`), cross-checked against
the IGCS Camera Tools 1.0.2 DLL and the GuardiansOfTheGalaxy-HeadTracking mod, and - where the mod uses it - verified by
`tools/verify_signatures.py` (byte signatures + RTTI lookups) and the unit tests in `tools/`.

## 1. Layout and protection

```
  section .shared  rva 0x00001000 vsize    37.94 MB raw    37.94 MB
  section .udata   rva 0x0242f000 vsize    11.28 MB raw    11.28 MB
  section .arch    rva 0x02ef2000 vsize    16.41 MB raw     7.99 MB
  section .sxdata  rva 0x03e99000 vsize     2.20 MB raw     0.00 MB
  section .xdata   rva 0x040b3000 vsize     0.14 MB raw     0.14 MB
  section .debug   rva 0x040d5000 vsize   445.97 MB raw   445.97 MB
  section .data1   rva 0x1ea24000 vsize     0.11 MB raw     0.11 MB
  section .impdata rva 0x1ea3f000 vsize     0.01 MB raw     0.01 MB
  section .didata  rva 0x1ea42000 vsize     0.00 MB raw     0.00 MB
  section .sdata   rva 0x1ea43000 vsize     0.00 MB raw     0.00 MB
  section .ecode   rva 0x1ea44000 vsize     0.01 MB raw     0.01 MB
  section .00cfg   rva 0x1ea48000 vsize     0.06 MB raw     0.06 MB
  section .data    rva 0x1ea57000 vsize     2.20 MB raw     2.20 MB
  section .data2   rva 0x1ec71000 vsize     2.47 MB raw     2.47 MB
```
* `.shared` (38 MB, executable) is the game code. `.udata` (read-only) holds vtables, RTTI locators and constants.
  `.arch` (writable) holds globals (TAA flag exe+305A116, near-plane override exe+3D97774, pass enable flags ...) and
  the RTTI type names.
* `.debug` is 446 MB, executable **and** writable, with `.sxdata` / `.ecode` / `.impdata` beside it and one "Denuvo"
  string: the anti-tamper layer. Game code is readable after start; GOTGVR waits for that (`game::Init` retries) and
  never patches inside `.debug`.
* Entry point exe+1EA43020 (`.sdata`, the protector's stub).

## 2. Imports and middleware

```
IMPORTS:
  advapi32.dll                   13  CryptAcquireContextA, CryptGetProvParam, CryptReleaseContext, GetUserNameA, GetUserNameW, RegCloseKey, RegCreateKeyExA, RegDeleteTreeA ...
  api-ms-win-crt-convert-l1-1-0.dll   13  _itoa, _strtod_l, _strtoi64_l, _strtoui64_l, atof, atoi, atol, mbsrtowcs ...
  api-ms-win-crt-environment-l1-1-0.dll    1  getenv
  api-ms-win-crt-filesystem-l1-1-0.dll    5  _fullpath, _wmkdir, _wremove, _wrename, _wstat64
  api-ms-win-crt-heap-l1-1-0.dll    7  _aligned_free, _aligned_malloc, _set_new_mode, calloc, free, malloc, realloc
  api-ms-win-crt-locale-l1-1-0.dll    4  _configthreadlocale, _create_locale, setlocale, ___lc_codepage_func
  api-ms-win-crt-math-l1-1-0.dll   26  __setusermatherr, _finite, _isnan, acosf, asinf, atan2f, atanf, cos ...
  api-ms-win-crt-runtime-l1-1-0.dll   26  _c_exit, _cexit, _clearfp, _configure_narrow_argv, _control87, _controlfp_s, _crt_atexit, _errno ...
  api-ms-win-crt-stdio-l1-1-0.dll   27  __acrt_iob_func, __p__commode, __stdio_common_vfprintf, __stdio_common_vfprintf_s, __stdio_common_vfscanf, __stdio_common_vsnprintf_s, __stdio_common_vsprintf, __stdio_common_vsprintf_s ...
  api-ms-win-crt-string-l1-1-0.dll   27  _memicmp, _stricmp, _strnicmp, _wcsdup, _wcsicmp, _wcsnicmp, _wcsupr, isalnum ...
  api-ms-win-crt-time-l1-1-0.dll    4  _time64, clock, strftime, _localtime64_s
  api-ms-win-crt-utility-l1-1-0.dll    4  bsearch, qsort, rand, srand
  bcrypt.dll                      1  BCryptGenRandom
  bink2w64.dll                   36  BinkAllocateFrameBuffers, BinkClose, BinkControlBackgroundIO, BinkDoFrame, BinkDoFrameAsync, BinkDoFrameAsyncWait, BinkDoFramePlane, BinkFreeGlobals ...
  dinput8.dll                     1  DirectInput8Create
  gdi32.dll                      17  BitBlt, CreateCompatibleBitmap, CreateCompatibleDC, CreateFontIndirectA, CreateFontW, DeleteDC, DeleteObject, GetDeviceCaps ...
  kernel32.dll                  177  AcquireSRWLockExclusive, AcquireSRWLockShared, AllocConsole, AreFileApisANSI, CancelIo, CloseHandle, ConvertThreadToFiber, CopyFileW ...
  libscepad.dll                   8  scePadClose, scePadGetControllerInformation, scePadInit, scePadOpen, scePadRead, scePadSetTriggerEffect, scePadSetVibration, scePadSetVibrationMode
  msvcp140.dll                   48  ??0?$basic_ios@DU?$char_traits@D@std@@@std@@IEAA@XZ, ??0?$basic_iostream@DU?$char_traits@D@std@@@std@@QEAA@PEAV?$basic_streambuf@DU?$char_traits@D@std@@@1@@Z, ??0?$basic_streambuf@DU?$char_traits@D@std@@@std@@IEAA@XZ, ??0?$codecvt@_SDU_Mbstatet@@@std@@QEAA@_K@Z, ??0?$codecvt@_WDU_Mbstatet@@@std@@QEAA@_K@Z, ??1?$basic_ios@DU?$char_traits@D@std@@@std@@UEAA@XZ, ??1?$basic_iostream@DU?$char_traits@D@std@@@std@@UEAA@XZ, ??1?$basic_streambuf@DU?$char_traits@D@std@@@std@@UEAA@XZ ...
  msvcp140_codecvt_ids.dll        1  ?id@?$codecvt@_SDU_Mbstatet@@@std@@2V0locale@2@A
  msvcrt.dll                      1  memcpy
  nrd.dll                         6  CreateDenoiser, DestroyDenoiser, GetComputeDispatches, GetDenoiserDesc, GetLibraryDesc, SetMethodSettings
  ntdll.dll                       9  NtClose, NtCreateNamedPipeFile, NtCreateThreadEx, NtGetContextThread, NtOpenFile, NtReadFile, NtSuspendThread, NtTerminateThread ...
  ole32.dll                       9  CoCreateGuid, CoCreateInstance, CoInitialize, CoInitializeEx, CoSetProxyBlanket, CoTaskMemAlloc, CoTaskMemFree, CoUninitialize ...
  oleaut32.dll                    3  SysFreeString, VariantClear, SysAllocString
  osdk.dll                      107  osdk_Allocator_Free, osdk_Allocator_Init, osdk_Allocator_Malloc, osdk_Base64_Encode, osdk_Create, osdk_HTTPHeaderNames_Get, osdk_HTTPRequestBuilder_construct, osdk_HTTPRequestBuilder_destruct ...
  secur32.dll                     1  GetUserNameExA
  setupapi.dll                    5  SetupDiDestroyDeviceInfoList, SetupDiEnumDeviceInterfaces, SetupDiGetClassDevsW, SetupDiGetDeviceInstanceIdW, SetupDiGetDeviceInterfaceDetailW
  shell32.dll                     5  CommandLineToArgvW, SHGetFolderPathW, SHGetKnownFolderPath, ShellExecuteA, ShellExecuteW
  shlwapi.dll                     1  PathFileExistsW
  steam_api64.dll                 9  SteamAPI_GetHSteamUser, SteamAPI_Init, SteamAPI_RegisterCallback, SteamAPI_RestartAppIfNecessary, SteamAPI_RunCallbacks, SteamAPI_Shutdown, SteamAPI_UnregisterCallback, SteamInternal_ContextInit ...
  user32.dll                    103  AdjustWindowRect, BeginPaint, CallWindowProcA, CallWindowProcW, CheckDlgButton, ClientToScreen, ClipCursor, CloseClipboard ...
  vcruntime140.dll               23  _CxxThrowException, __C_specific_handler, __CxxFrameHandler3, __current_exception, __current_exception_context, __intrinsic_setjmp, __std_exception_copy, __std_exception_destroy ...
  vcruntime140_1.dll              1  __CxxFrameHandler4
  version.dll                     6  GetFileVersionInfoA, GetFileVersionInfoExW, GetFileVersionInfoSizeA, GetFileVersionInfoSizeExW, VerQueryValueA, VerQueryValueW
  wininet.dll                     9  HttpOpenRequestA, HttpQueryInfoA, HttpSendRequestA, InternetCloseHandle, InternetConnectA, InternetOpenA, InternetQueryOptionA, InternetReadFile ...
  winmm.dll                       2  timeEndPeriod, timeBeginPeriod
  ws2_32.dll                     32  accept, ioctlsocket, WSAGetLastError, WSAStartup, WSACleanup, inet_ntoa, listen, ntohl ...
  xinput9_1_0.dll                 3  XInputGetState, XInputSetState, XInputGetCapabilities
```
* The exe **exports Wwise** (`AK::SoundEngine::...`, `CAkEmitter` ...): the sound engine is linked in statically.
* Middleware by strings / imports: **Havok** physics (`hknp*`, `ZPhysicsObjectHavok` - metres), **Wwise** audio,
  **Bink 2** video (`bink2w64.dll`), **Umbra** occlusion culling, **NavPower** navigation, **NVIDIA NRD** denoisers
  (`nrd.dll`, ray-traced reflections), **DLSS** (`nvngx`), **FSR 1 + CAS** (own passes), **libscepad** (DualSense
  triggers / vibration), **osdk** (Square Enix online SDK), Steamworks. No PhysX / APEX / FMOD / Scaleform UI
  rendering (DXMD's middleware list differed: Dawn swapped PhysX/APEX for Havok and FMOD for Wwise).
* Graphics API: D3D12 (shipped); `USEVULKAN` / `USED3D12` ini switches exist (exe+C9B40), a Vulkan device class is
  compiled in, D3D11 is the "base" tier. `USEPCSETTINGS`, `USECURRENTDIR` are the other launcher switches.

## 3. Classes (RTTI)

RTTI types (classes/structs, unscoped): 6266 (+ namespaced/template ones)
  most common prefixes: ZRender 305, ZCamera 193, ZBase 163, ZGame 118, ZCharacter 90, ZValue 78, ZBehavior 78, SEffect 76, ZDebug 75, ZPlayer 71, ZUIEle 71, ZAudio 62, ZRuntime 60, ZFlow 52, ZGeneric 50, ZEntity 42, ZAnimation 39, ZBone 34, ZMelee 34, ZAction 33, hkImag 30, ZAnim 30, ZInput 30, ZNavigation 30, SRender 30, hknpCo 29, ZOnline 29, ZPostfilter 28, ZStory 28, ListIt 28, IRender 26, ZCombat 25, ZControl 24, hknpVe 23, ZScene 23, ZDamage 23, ZTarget 22, ZFilter 21, hkndDe 20, ZIs 20
  camera (221): ICameraBehaviorDataModifier, ICameraBehaviorRotation, ICameraDataOwner, ICameraEntity, ICameraIntentions, ICameraShakeHandler, ICameraTrack, IDebugCameraModeHandler, ZBaseCameraProjectionOnShapeEntity, ZCameraAction, ZCameraActionData, ZCameraActionData_LookAt_Actor, ZCameraActionEntity, ZCameraActionEntity_LookAt, ZCameraActionEntity_Recenter, ZCameraActionLookAt, ZCameraActionRecenter, ZCameraBehavior, ZCameraBehaviorData, ZCameraBehaviorDataModifier_Collision, ZCameraBehaviorDataModifier_DigitalInput, ZCameraBehaviorDataModifier_Input, ZCameraBehaviorDataModifier_PassThrough, ZCameraBehaviorDataModifier_PassThroughDelta, ZCameraBehaviorDataModifier_SimpleCollision, ZCameraBehaviorData_Action, ZCameraBehaviorData_AdditiveTransform, ZCameraBehaviorData_AimRotation, ZCameraBehaviorData_AimRotation_Offset, ZCameraBehaviorData_CoverAlignment, ZCameraBehaviorData_Damping, ZCameraBehaviorData_DollyZoom, ZCameraBehaviorData_FOV, ZCameraBehaviorData_FirstPerson, ZCameraBehaviorData_FollowSpline, ZCameraBehaviorData_FrameFollow, ZCameraBehaviorData_Free, ZCameraBehaviorData_Generic, ZCameraBehaviorData_GuardianLookAt, ZCameraBehaviorData_Input_Debug, ZCameraBehaviorData_LastTransform, ZCameraBehaviorData_Locator, ZCameraBehaviorData_LocatorSwitcher, ZCameraBehaviorData_LogicSpline, ZCameraBehaviorData_LookAt, ZCameraBehaviorData_MarketingMode, ZCameraBehaviorData_Melee, ZCameraBehaviorData_Offset, ZCameraBehaviorData_Offset_Aligner, ZCameraBehaviorData_PlaneClip, ZCameraBehaviorData_Position, ZCameraBehaviorData_ProximityDithering, ZCameraBehaviorData_Recenter, ZCameraBehaviorData_Rotation, ZCameraBehaviorData_Rotation_Stabilizer, ZCameraBehaviorData_RubberBand, ZCameraBehaviorData_Sequencer, ZCameraBehaviorData_Shake, ZCameraBehaviorData_Slide, ZCameraBehaviorData_Spline ...
  render graph nodes (38): ZRenderGraphNode, ZRenderGraphNodeCamera, ZRenderGraphNodeCanvas, ZRenderGraphNodeCubemapProbe, ZRenderGraphNodeDestination, ZRenderGraphNodeDisplacer, ZRenderGraphNodeFogBox, ZRenderGraphNodeForceAttractor, ZRenderGraphNodeForceBase, ZRenderGraphNodeForceGravity, ZRenderGraphNodeForceOmni, ZRenderGraphNodeForceTurbulence, ZRenderGraphNodeForceVelocityTransfer, ZRenderGraphNodeForceVortex, ZRenderGraphNodeGIVolume, ZRenderGraphNodeGIVolumeModifier, ZRenderGraphNodeGeomCache, ZRenderGraphNodeInstancedPrimitiveContainer, ZRenderGraphNodeLODProxy, ZRenderGraphNodeLight, ZRenderGraphNodeLinked, ZRenderGraphNodeMaterial, ZRenderGraphNodeMultiTransformGeomCache, ZRenderGraphNodeParticles, ZRenderGraphNodePostfilterController, ZRenderGraphNodePrimitiveContainer, ZRenderGraphNodeProjectiveDecal, ZRenderGraphNodeProjectiveDecalMaterial, ZRenderGraphNodeRainAccessibility, ZRenderGraphNodeRainSurfaceSimulator, ZRenderGraphNodeReflectorBase, ZRenderGraphNodeRenderGlobal, ZRenderGraphNodeShadowCaster, ZRenderGraphNodeShadowCasterBase, ZRenderGraphNodeSky, ZRenderGraphNodeSource, ZRenderGraphNodeVideoPlayer, ZRenderGraphNodeVisibilityProxy
  render passes (43): ZRenderPassAlphaMask, ZRenderPassAtmosphere, ZRenderPassBase, ZRenderPassBentCone, ZRenderPassCharacterBoost, ZRenderPassCharacterRig, ZRenderPassClearTarget, ZRenderPassColorOverride, ZRenderPassDebug, ZRenderPassDeferredOpaqueLighting, ZRenderPassDownsample, ZRenderPassEyeGeoModif, ZRenderPassFauveMauve, ZRenderPassFilteredColor, ZRenderPassGBuffer, ZRenderPassGI, ZRenderPassHair, ZRenderPassHairBlur, ZRenderPassInWorldUI, ZRenderPassInWorldUIPostProcessed, ZRenderPassLensFlares, ZRenderPassLightScattering, ZRenderPassLightShaftMask, ZRenderPassOpaqueEmissive, ZRenderPassPostFilters, ZRenderPassProjectiveDecal, ZRenderPassRT, ZRenderPassRaytracingScene, ZRenderPassReflection, ZRenderPassSaturationMask, ZRenderPassScreenSpaceMesh, ZRenderPassScreenSpaceUI, ZRenderPassScreenSpaceUIPostProcessed, ZRenderPassSkin, ZRenderPassSoulGlo, ZRenderPassStagger, ZRenderPassTextureDestinationPostFilters, ZRenderPassTransparent, ZRenderPassTransparentShadows, ZRenderPassUIBase, ZRenderPassVelocity, ZRenderPassVertexPreprocess, ZRenderPassWind
  postfilters (84): IPostfilter, IRenderPostfilterControllerEntity, IRenderPostfilterParametersEntity, SEffectParameterPostfilterCS, SRenderPostfilterParametersBase, SRenderPostfilterParametersCharacterBoost, SRenderPostfilterParametersChromaticAberration, SRenderPostfilterParametersColorCorrection, SRenderPostfilterParametersDepthOfField, SRenderPostfilterParametersExposure, SRenderPostfilterParametersFadeOverlay, SRenderPostfilterParametersFauveMauve, SRenderPostfilterParametersFilmGrain, SRenderPostfilterParametersGaussianBlur, SRenderPostfilterParametersGlare, SRenderPostfilterParametersGlitch, SRenderPostfilterParametersLensFlare, SRenderPostfilterParametersLightShaft, SRenderPostfilterParametersMotionBlur, SRenderPostfilterParametersOutline, SRenderPostfilterParametersRadialBlur, SRenderPostfilterParametersRainHaze, SRenderPostfilterParametersSSAO, SRenderPostfilterParametersSSR, SRenderPostfilterParametersScattering, SRenderPostfilterParametersSoulGlo, SRenderPostfilterParametersStagger, SRenderPostfilterParametersVHS, SRenderPostfilterParametersVignette, SRenderPostfilterParametersVignetteTransition, ZPostfilterAreaBoxEntity, ZPostfilterAreaTransitionBoxEntity, ZPostfilterAspectLightShaftAspect, ZPostfilterCharacterBoostAspect, ZPostfilterChromaticAberrationAspect, ZPostfilterColorCorrectionAspect, ZPostfilterContainerEntity, ZPostfilterDepthOfFieldAspect, ZPostfilterExposureAspect, ZPostfilterFadeOverlayAspect, ZPostfilterFauveMauveAspect, ZPostfilterFilmGrainAspect, ZPostfilterGaussianBlurAspect, ZPostfilterGlareAspect, ZPostfilterGlitchAspect, ZPostfilterLensFlareAspect, ZPostfilterMotionBlurAspect, ZPostfilterOutlineAspect, ZPostfilterRadialBlurAspect, ZPostfilterRainHazeAspect, ZPostfilterSSAOAspect, ZPostfilterSSRAspect, ZPostfilterSoulGloAspect, ZPostfilterStaggerAspect, ZPostfilterVHSAspect, ZPostfilterVignetteAspect, ZPostfilterVignetteTransitionAspect, ZPostfilterVolumetricScatteringAspect, ZRenderGraphNodePostfilterController, ZRenderPostfilterControllerEntity ...
  VR / stereo (0): 
  UI screens (pause) (19): ListItemDataPauseMenuDefault, ZGWUIPauseMenuGameplayCheatScreen, ZPauseMenuAIDebugCheatScreen, ZPauseMenuCheatEntity, ZPauseMenuCheatManager, ZPauseMenuCoverDebugCheatScreen, ZPauseMenuPlayerPerksDebugCheatScreen, ZPauseMenuUIDebugCheatScreen, ZPauseMenuViewportSettingsCategoryScreen, ZPauseMenuViewportSettingsScreen, ZUIPauseMenuAchievementCheatScreen, ZUIPauseMenuCameraDebugScreen, ZUIPauseMenuCameraSettingDebugScreen, ZUIPauseMenuCheatScreen, ZUIPauseMenuGameplayCheatScreen, ZUIPauseMenuGeneralDebugScreen, ZUIPauseMenuItemsCheatScreen, ZUIPauseMenuMain, ZUIPauseMenuMainScreen
  video (29): IBinkDecoder, ILoadingScreenVideo, IRenderVideoPlayerEntity, IVideoPlayer, ZBink, ZBinkAudioConfig, ZBinkDecoder_D3D11, ZBinkDecoder_Software, ZBinkManager, ZLoadingScreenVideo, ZRenderGraphNodeVideoPlayer, ZRenderToTextureVideoPlayer, ZRenderUIVideoPrimitive, ZRenderVideoPlayerBase, ZRenderVideoPlayerEntity, ZRenderVideoRAMAllocator, ZRuntimeVideoModule, ZUIElement_Video, ZUIVideoScreen, ZUIVideoSettingScreen, ZVideoArchivesInterface, ZVideoArchivesItem, ZVideoEntity, ZVideoPlayerUnitTest, ZVideoRecorder, ZVideoSequenceItem, ZVideoSequenceTrack, ZVideoSettings, ZVideoSettingsBank
  photo mode (29): IPhotoModeOptionBehaviorEntity, IPhotoModeOptionEntity, IRenderPhotoModeManager, ListItemDataPhotoModeBase, ListItemDataPhotoModeCheckBox, ListItemDataPhotoModeGauge, ListItemDataPhotoModeOptionStepper, ZBasePhotoModeOptionBehaviorEntity, ZBasePhotoModeOptionEntity, ZNumericPhotoModeOptionBehaviorEntity, ZNumericPhotoModeOptionEntity, ZOptionListPhotoModeOptionBehaviorEntity, ZOptionListPhotoModeOptionEntity, ZOptionListPhotoModeOptionItem, ZOptionListPhotoModeOptionTextureItem, ZPhotoModeFlagEntity, ZPhotoModeManager, ZPhotoModeManagerEntity, ZPhotoModeMetricEntity, ZRenderPhotoModeManager, ZRenderPhotoModeManagerDataEntity, ZTabDataPhotoMode, ZTabItemPhotoMode, ZTogglePhotoModeOptionBehaviorEntity, ZTogglePhotoModeOptionEntity, ZUIMenuPhotoMode, ZUIPhotoModeCharacterScreen, ZUIPhotoModeDOFScreen, ZUIPhotoModeScreen
  input (207): IADEBodyController, IAsyncInputStream, IBaseController, IBaseInputStream, ICharacterController, ICharacterControllerOwner, IController, IControllerSupports, IInputButtonFeedbackEntity, IInputButtonFeedbackParams, IInputDevice, IInputDeviceManager, IInputIntentions, IInputScheme, IInputStream, ILODControllerManager, IMeleeSideStepController, IPCInputRemapping, IProjectiveDecalController, IRenderPostfilterControllerEntity, ISoundAmbienceController, ZADEBaseAnimationController, ZADEBaseBodyControllerImpl, ZADEBaseController, ZADEPlayerController, ZAIBrainController, ZAIController, ZAIControllerFromCharacter, ZAIControllerFromEntityRef, ZAIControllerSafeGetter, ZAIInputSystem, ZActivateInputSchemeEntity, ZAnalogInputAction, ZAnalogInputActionCheckerEntity, ZAnalogInputPairing, ZAnimationControllerAiming, ZAnimationControllerAnimDialogue, ZAnimationControllerEnvironment, ZAnimationControllerGeneric, ZAnimationControllerHitReaction, ZAnimationControllerIncapacitated, ZAnimationControllerInspecting, ZAnimationControllerLocomotion, ZAnimationControllerLookAt, ZAnimationControllerMain, ZAnimationControllerMeleeCombat, ZAnimationControllerMount, ZAnimationControllerNecrofiend, ZAnimationControllerNewCover, ZAnimationControllerOccupation, ZAnimationControllerProtectiveDevice, ZAnimationControllerReaction, ZAnimationControllerRide, ZAnimationControllerSequence, ZAsyncResourceInputStream, ZBaseAnimationController, ZBaseCharacterAnimationController, ZBaseCharacterController, ZBaseController, ZBaseInputAction ...
  time / game speed (14): ZAnimTimeScaleEntity, ZBarkTimerRestriction, ZBaseTimerComponent, ZCinematicCombatTimeScaleEntity, ZFlowTimer_Finite, ZFlowTimer_Fixed, ZFlowTimer_Infinite, ZFlowTimer_Random, ZFlowTimer_Variable, ZPlayerPromptBaseTimerEntity, ZScopedTimer, ZScriptHighPerformanceTimer, ZTimeScaleStatusEffectEntity, ZTimerCondition
  upscalers (10): SEffectParametersCAS, SEffectParametersFSR, SEffectParametersFSRSlow, ZRenderDLSS, ZRenderDLSSImpl, ZRenderPostfilterPassCAS, ZRenderPostfilterPassDLSS, ZRenderPostfilterPassFSR, ZRenderPostfilterPassFSRRCAS, ZRenderPostfilterPassUpscaling
  ray tracing (9): ZRaytracingBLAS, ZRaytracingBLASD3D12, ZRaytracingCommandList, ZRaytracingExcludeAspect, ZRaytracingSceneManager, ZRaytracingTLAS, ZRaytracingTLASD3D12, ZRenderPassRaytracingScene, ZRenderRaytracingPassManager

Points that matter for VR:
* **No VR / stereo class at all** (zero `Stereo*`, `*VR*`, `Hmd`, `Oculus`, `OpenVR` types) - DXMD's stereo driver and its
  2017 "VR Experience" build path did not survive into GOTG. GOTGVR's own stereo (synchronized / reconstruction / AER)
  is the only way.
* `ZCameraBehaviorData_FirstPerson`, `_MarketingMode`, `_Free`, `_DollyZoom` and `ZUIPauseMenuCameraDebugScreen` /
  `ZUIPauseMenuCameraSettingDebugScreen` exist: a first-person camera behaviour and debug camera screens are compiled in
  (not reachable from the shipped menus). Candidates for a future first-person / free-camera VR mode.
* `ZPhotoModeManager`, `ZUIPhotoModeDOFScreen`: photo mode has its own DOF screen (covered by the DOF switch below).
* `ZRenderPassScreenSpaceUI` / `...PostProcessed` / `ZRenderPassInWorldUI`: the UI split GOTGVR uses for UI placement and
  stereo reconstruction (screen-space UI is drawn after the 3D picture).

## 4. Render camera and frame (verified, used by the mod)

| What | Where | Notes |
|---|---|---|
| ZRenderGraphNodeCamera vtable | exe+2571C98 | slot 9 = view update exe+990CB0 (hooked); RTTI lookup confirms |
| View update builds the view | exe+990D40 | rigid inverse of node+0x24 (rows right / up / back, position) |
| Projection getter | exe+972270 | copies node+0xC0, rebuilds depth rows (standard 0..1), jitter exe+979FE0 (hooked) |
| Ortho test | exe+F7B40 | P[15] check, skips the rebuild |
| Near-plane override global | exe+3D97774 | used when > 0 by the getter (exe+9722CD) and exe+9724C0 - GOTGVR `near_clip` |
| Second view x projection | exe+9724C0 | LOD screen-size only (caller exe+9CBD30) |
| Render size | node+0x1A8 / +0x1AC | near / far node+0x1D0 / +0x1D4 |
| View outputs | view+0x10 view, +0x50 proj, +0x90 view x proj, +0xE20 position | all from the hooked getter |
| Camera chain | exe+C997C2 (IGCS copy, record -> r14+0x390) -> exe+C83780 (entity transform, FOV x 57.2958) -> ZCameraReflector (vtable exe+2567778) -> node+0x24 -> scene views exe+97FD30 | gameplay entity rows right / forward / up; render node rows right / up / back |
| Game clock | TimerUpdate exe+42E890 on exe+3C2CDF8 (call exe+D2488) | `paused` argument from the pause manager (exe+D2457..D2486); IGCS timestop = its tail exe+42E963 |
| TAA flag | exe+305A116 | read by the jitter exe+979FE0 |
| Pause on focus loss | exe+CBC2F | IGCS patch site |
| Cutscene black bars | exe+960D7F / exe+960DD5 | IGCS patch sites |
| HUD flag | movzx at exe+BD5619 -> exe+3DCB920 | IGCS HUD toggle |
| Settings table | exe+1CC1E40 (runtime.pcsettings) | VSync, HDR, ... MotionBlur, RadialBlur, ChromaticAberration, Dof, LensFlare, AntiAliasing, DlssQuality, FSRQuality, RayTracing ... (no stereo settings) |
| World | Z-up, metres | gravity 9.81 stored at exe+20E892D, Havok |

## 5. Post filters (all 21 passes)

Each `ZRenderPostfilterPass<X>` has a 3-slot vtable: **[0] IsActive** (bool; the chain skips the pass when false), **[1]
record** the pass into the render graph, [2] destructor. GOTGVR switches effects off at [0] - the game's own "effect off"
path.

| Pass | vtable | IsActive | record | GOTGVR |
|---|---|---|---|---|
| Antialiasing | exe+25752C8 | | exe+8D53B0 | (TAA via the TAA flag) |
| CAS | exe+2575508 | | exe+8D5EA0 | |
| ColorGrading | exe+25752E8 | exe+E59E0 (shared with Exposure) | exe+8D6100 | |
| DLSS | exe+2575308 | | exe+9030F0 | |
| **DOF** | exe+2575348 | **exe+8DA090** | exe+8D6780 | **dof_off (default on)** |
| Exposure | exe+25696A0 | exe+E59E0 | exe+9033B0 | |
| FSR / FSRRCAS / Upscaling | exe+2575528 / 2575548 / 2575328 | | exe+9038C0 / 903B20 / 907210 | |
| FadeOverlay | exe+2575448 | | exe+903D80 | (needed: fades) |
| **FilmGrain** | exe+25754A8 | **exe+90C0F0** | exe+904140 | film_grain_off |
| GaussianBlur | exe+25754E8 | | exe+904480 | (menus) |
| Glare | exe+25753C8 | exe+90C1A0 | exe+904E50 | |
| **HexDOF** (bokeh) | exe+2575368 | **exe+90C210** | exe+905950 | **dof_off (default on)** |
| **MotionBlur** | exe+25753E8 | **exe+90C2A0** | exe+9061D0 | motion_blur_off (default on) |
| Outline | exe+257E0D8 | | exe+C76B0 | |
| **RadialBlur** | exe+25753A8 | **exe+90C2E0** | exe+906910 | radial_blur_off (default on) |
| RainHaze, VHS, Vignette, VignetteTransition | exe+2575388 / 25754C8 / 2575488 / 2575428 | Vignette exe+90C760 | exe+906D60 / 907560 / 9078A0 / 907BE0 | |

* DOF `IsActive` (exe+8DA090) is true only with its enable flag exe+304D15D set, active DOF parameters, strength
  (params+0x2C) > 0 and a perspective camera - gameplay DOF (the setting) and cutscene DOF
  (`ZPostfilterDepthOfFieldAspect`, `ActivateDepthOfField`) both run through DOF / HexDOF, so switching both off removes
  depth of field everywhere, photo mode included.
* Parameter blocks exist for ChromaticAberration, LensFlare, LightShaft, SSAO, SSR, Glitch, Scattering, SoulGlo,
  CharacterBoost, Stagger, FauveMauve (no separate passes for chromatic aberration / lens flare: they live in other
  passes or render passes - `ZRenderPassLensFlares`).

## 6. IGCS Camera Tools 1.0.2 sites, identified

The IGCS DLL's AOBs, matched to the exe and named by RTTI (its labels and patterns are interleaved in its string table;
the pairing below is from the exe, not from the label order):

| IGCS pattern | exe address | What it really is |
|---|---|---|
| camera copy (6 movups) | exe+C999A4 | gameplay camera record copy (head-tracking mod hook) |
| `48 89 5C 24 10 ... 4D 8B E0` | exe+8D6780 | ZRenderPostfilterPassDOF::record |
| `48 89 5C 24 10 ... 4D 8B F8` | exe+905950 | ZRenderPostfilterPassHexDOF::record |
| `... E0 FE FF FF 48 81 EC 20 02 00 00` | exe+904E50 | ZRenderPostfilterPassGlare::record |
| `... 6C 24 90 48 81 EC 70 01 00 00 49` | exe+906910 | ZRenderPostfilterPassRadialBlur::record |
| `F3 0F 10 05 \| ...` | exe+929E21 | a per-quality float getter exe+929E00 (7 callers, generic - not used by GOTGVR) |
| `FF 50 18 EB 7B 48 8B 07 48 89 41 08 EB 72 E8` | exe+12305D | LOD scale (IGCS "higher LODs") |
| `48 C1 F9 14 49 89 48 30 ...` | exe+42E963 | the game clock update's tail (IGCS timestop) |
| black bars, pause on focus loss, HUD flag | exe+960D7F / 960DD5, CBC2F, BD5619 | used by GOTGVR |

## 7. The engine and its family (research)

* **Dawn Engine** (Eidos-Montréal) = a heavily modified **Glacier 2** (IO Interactive, Hitman: Absolution era). Games:
  Deus Ex: Mankind Divided (2016), Deus Ex: Breach (2017, inside DXMD), the free **DXMD VR Experience** (2017, Steam -
  Dawn once rendered VR: Vive / Rift / OSVR), and Marvel's Guardians of the Galaxy (2021; PC port co-developed with d3t,
  ray tracing + DLSS added). Eidos-Montréal moved to Unreal Engine 5 afterwards; GOTG is the last Dawn game.
* Marvel's Avengers is **not** Dawn (Crystal Dynamics' Foundation engine) - nothing from Avengers mods applies.
* IO Interactive later added native VR to **Glacier** (Hitman 3: PSVR 2021, PC VR January 2022, PSVR 2) - after the
  Dawn fork, so none of that code is in GOTG.
* DXMD (same engine) is what DXMDVR runs: the render camera node, getter, view update, units (metres) and coordinate
  conventions all carried over; DXMD's stereo driver (Stereoscopic3DMode, eye index at view+0x528) and its PhysX /
  FMOD did not.
* Other GOTG mods worth knowing: IGCS Camera Tools (Otis_Inf), the OpenTrack head-tracking ASI, Paul44's Cheat Engine
  table (FearLess), "Simple Realistic 3D" (ReShade Depth3D), vorpX (unofficial).

Sources: Wikipedia / HandWiki "Dawn Engine"; TheGamer and Screen Rant (Eidos: GOTG uses Dawn, not Foundation);
Steam app 526180 (DXMD VR Experience); Road to VR / UploadVR (Hitman 3 PC VR); Autodesk "Building the world of DXMD"
(metres); dsogaming (Dawn = modified Glacier 2).


## 8. Player, skeleton and first person (0.6.0)

| What | Where | Notes |
|---|---|---|
| ZPlayerManager instance | exe+3E41B58 (vtable exe+27C9A70) | built at exe+1877EED; ZCharacterManager next to it at exe+3E41BE0 (+0x28: character list, 16-byte {ref, obj}) |
| local player character | exe+1784E90 (this = ZPlayerManager, out = {entity, ZPreparableCharacterAspect*}) | fallback exe+1785560 walks the controller list of *(exe+3C339E0) |
| entity interfaces | *(entity) = type; type+0x18 -> {array, count} of {type info*, offset} | the game's own QueryInterface walks it from the end |
| ZCharacter type info | exe+3D99E18 | from ZCharacter vtable exe+27C0430 slot 4 (exe+1756650: lea rax, [table]; lea r8, [type info]) |
| GetAnimatedEntity | ZCharacter slot 80 = exe+17425C0 | ref at ZCharacter+0x2308 / provider at +0x2310; out = {ref, ZAnimatedEntity*} |
| ZAnimatedEntity | vtable exe+2547ED8 | +0x340 pose (+8: 32-byte model-space QV per bone), +0x348 rig (+0x88/+0x90/+0x98 bone hash table), +0x2C m_bVisible |
| SetVisible | ZAnimatedEntity slot 76 = exe+593760 | what ZCameraBehavior_FirstPerson (vtable exe+276C018, slots 9/10 -> exe+1569400) calls on the followed character: hide on activate, show on deactivate |
| bone world transform | exe+81E740 GetBoneWorldTransformQV(this, out QV, id) | script native (registered exe+8255F6); -> exe+816270 world = root x pose[bone]; unknown id -> the entity's own transform |
| bone by name (entity) | ZAnimatedEntity slot 105 exe+814FE0 -> slot 106 exe+815040 | creates a ZBoneEntity ("ZAnimatedEntity::GetBoneEntity") - not used (allocates) |
| bone id | CRC-32 (IEEE, table exe+243D800) of the lower-case name, exe+1093B0; registry exe+3C226C8 | jntbd_head 0x2A4C5889, jntbd_neck 0xADAE7846 |
| body joints | names exe+2445998.. (jntbd_pelvis, _spine_00.._03, _neck, _head, arms, legs), CRCs exe+2EFD5D0.. | the engine's 21-joint body map (exe+14A470) |
| ZBoneSpatialFromCharacterEntity | exe+1700200 | character -> slot 80 -> slot 105 (flags 5): the same chain the mod uses |
| other first-person markers | "IsInFirstPerson", Locomotion.Movement.FirstPerson, FirstPersonStateEntered/ExitedEvent, eFirstPersonRequest | the game's scripted first-person locomotion state; hknpFirstPersonCharacter is Havok sample code, unused |

## 9. Render options and anti-aliasing (0.6.4)

- PC settings block exe+3E603A0, settings apply exe+CDBF1, option table exe+3DE1C70 with setters exe+DC7540 (bool),
  exe+DC7700 (int), exe+DC76C0 (float) and getter exe+D4010 (bool). Option ids: 0x4BE AntiAliasing, 0x51B
  DynResType, 0x51A ResScale, 0x51D DlssQuality, 0x520 FSRQuality, 0x4B9 MotionBlur, 0x4BC Dof (full map in
  NOTES.md "Render options").
- AA mode per view: exe+A71B80 writes view+0x384 (0 / 1 non-temporal / 2 TAA), using exe+977B70 (TAA byte
  exe+305A116 and count exe+305A118). The AA pass is exe+8D53B0 ("[AA] ApplyTAA" / "[AA] ApplyNoTAA").
  The upscaling passes are exe+9030F0 ("Upscaling - DLSS") and exe+9038C0 ("[Postfilter] FSR").
- UI: ZUIManager (ctor exe+BD1ED0, profiler scopes FinalFrameUpdate / UpdateCanvasBegin / UpdateCanvasEnd),
  ZRenderGraphNodeCanvas (vtable exe+25B7DE0), ZCanvasEntity (vtable exe+25B6380). The UI primitives (sprite, text,
  mask, blur, video, particles, subscene) are drawn into a UI layer and blended into the picture by a compute pass.
