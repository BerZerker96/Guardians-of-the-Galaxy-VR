<div align="center">


<img width="2151" height="1165" alt="213g" src="https://github.com/user-attachments/assets/8de20e65-fcd6-4846-a1d1-3e2cd4e43915" />

# 🚀 Marvel's Guardians of the Galaxy VR (GOTGVR)

**Play Marvel's Guardians of the Galaxy (PC, Steam) in VR**

 ![game](https://img.shields.io/badge/game-Guardians%20of%20the%20Galaxy-222) ![xr](https://img.shields.io/badge/OpenXR-SteamVR%20%7C%20Pimax%20%7C%20Quest%20%7C%20VDXR%20%7C%20WMR-555)

</div>



<img width="5639" height="2935" alt="GOTGVR_20261117_235530_181_left" src="https://github.com/user-attachments/assets/66e6f58d-933a-45cd-a8e5-fc9156c6ea97" />

---

## 📦 Install

1. Copy `dxgi.dll` and `Start_GOTG_VR.bat` from the `mod` folder into the game folder with `GOTG.exe`.
   (`alt\winmm.dll` / `alt\GOTGVR.asi` are the same mod for other loaders - use only one.)
2. Start your VR software, then start the game with `Start_GOTG_VR.bat`. VR starts by itself.

Works with the original exe and the FuckTAA-patched exe. To uninstall, delete `dxgi.dll`, the `GOTGVR` folder and `gotgvr_config.txt`.

---

## ✨ Features

- **Stereo 3D with head and position tracking** on the game's own camera - three methods: stereo reconstruction (full frame rate), synchronized (both eyes from the same moment), alternate eye.
- **Any OpenXR headset, set up by itself**: the game renders at your headset's own resolution and shape, with each eye's real field of view - nothing to set.
- **No pop-in at the edges** of your view (wide culling), no extra rendering cost.
- **First person** (`End`) in Star-Lord's head, back to the game camera in cutscenes.
- **HUD scaled into view** (UI scale), kept in the middle of each eye.
- **Comfort**: level horizon, camera shake filter, idle-sway lock (no camera drift when you stand still), near clip so you can lean in close.
- **Flat virtual screen** for pause menus, loading screens and video cutscenes, switched automatically.
- **Smooth frame pacing** at your headset's refresh rate; the headset's smoothing never halves the fps.
- **Fixed foveated rendering** (RTX 20+, RX 6000+, Arc) for more fps.
- **Edge fill**: no black border where the headset sees past the picture.
- **Your game settings stay yours** - the mod changes no game option unless you choose it in the VR panel.
- **In-headset VR panel** with every setting, in tabs: View, Camera, HUD, Graphics, Screen, Advanced.

---

## 🎮 Controls

| Keyboard | Gamepad | Action |
|---|---|---|
| `Insert` | hold `L3 + R3` | VR panel |
| `Space` / `F9` | – | Recentre |
| `End` | – | First person on / off |
| `F8` | – | Stereo method / mono / flat screen |
| `F7` | – | Hide / show the HUD |
| `F6` | – | Flat screen on / off |
| `Delete` | – | Screenshot (PNG next to `GOTG.exe`) |

Settings are saved in `gotgvr_config.txt` next to `GOTG.exe` (change them in the VR panel; delete the file for the defaults).

---

## ❓ Problems?

- **No picture in the headset:** start your VR software first and make it the active OpenXR runtime.
- **Double image in alternate eye / synchronized:** set Game anti-aliasing and Game upscaling to Off (Graphics).
- **HUD not scaling:** HUD tab, step **UI element (picker)** 1, 2, 3 … with the HUD on screen; 0 = automatic.
- **Anything else:** send `GOTGVR\gotgvr.log` and `GOTGVR\xrhost.log`.

Source and build notes are in the `source` folder (`scripts/build_mingw.sh`); history in `CHANGELOG.md`.

---

## 🙏 Credits

| Who | What |
|---|---|
| **mark-mon** | VR framework this mod is built on: XrBridge OpenXR host, vrkit VR panel, proxy loader |
| **Otis_Inf** | IGCS GotG Camera Tools: camera chain, game clock, cutscene black bars and focus-pause sites |
| **GotG OpenTrack head-tracking mod** | Notes on the game's camera |
| **praydog / UEVR** | The synchronized-sequential idea behind Synchronized stereo |
| **adderek / vrdual** | The alternate-eye approach |
| **PureDark (AFW) and the Dark Souls III VR mod** | Showed that depth-based eye reconstruction works. Idea only, no code used |
| **Paul44** | GotG Cheat Engine table: pointers used in research |
| **r/FuckTAA (TL431)** | TAA patch research that led to the game's TAA flag |
| **Tsuda Kageyu** | MinHook (BSD 2-Clause) |
| **Omar Cornut** | Dear ImGui (MIT) |
| **Khronos Group** | OpenXR SDK and loader (Apache 2.0) |

<sub>Marvel's Guardians of the Galaxy is a trademark of Marvel / Square Enix / Eidos-Montréal. This is an unofficial fan mod with no game files included. You need your own copy of the game.</sub>
