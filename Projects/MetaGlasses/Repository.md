# MetaGlasses — Repository

Documentation of **`Adipat2003/MetaGlassTest`**, the working codebase, as of 2026-08-26.
Built by Adipat. Cloned locally to `/Users/vedeti123/Documents/MetaGlasses/MetaGlassTest/`.

> **Status: builds and runs on Ved's machine** (2026-08-26). iPhone 17 simulator, iOS 26.5. DAT mock initializes, registers, discovers and connects a mock device. Offline LLM reports `MODEL NEEDED` (Gemma not committed — expected). Display send cannot work on simulator — see gap 3.

## Running it locally — what Ved's machine needed
Xcode 26.6 and Node v26.5.0 were already fine (RN 0.87 accepts Node 26+; the `xcode-select --switch` step was already done). Two real blockers:

1. **System Ruby was 2.6.10** (macOS built-in, from 2018) but `Gemfile.lock` was generated on **Ruby 4.0.6 / bundler 4.0.16**. Fixed with `brew install ruby` — Homebrew ships exactly 4.0.6, an exact lockfile match. *Chose this over `brew install cocoapods` (1.17.0) because the Gemfile deliberately pins `xcodeproj < 1.26.0` and `concurrent-ruby < 1.3.4` to dodge known build failures, and standalone CocoaPods would drag in newer ones.*
2. **Homebrew Ruby is keg-only.** Add to `~/.zshrc`, or `bundle` silently falls back to system Ruby and `pod install` breaks again:
   ```sh
   export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
   ```

Full sequence from a fresh clone:
```sh
npm install
npm approve-scripts fsevents        # npm 11+ blocks install scripts; fsevents is Metro's file watcher
gem install bundler:4.0.16
bundle install
bundle exec pod install --project-directory=ios   # ~103s, 88 pods
npm run ios -- --simulator "iPhone 17"
```
Open `ios/MetaGlassTest.xcworkspace` in Xcode — **the workspace, not the `.xcodeproj`** (CocoaPods projects only build from the workspace).

Deployment target is `17.2`, matching DAT 0.9.0's raised minimum. DAT resolves via SPM at build time into `DerivedData/.../SourcePackages/checkouts/meta-wearables-dat-ios` as binary xcframeworks.

> **Access note**: the repo is **private**, so HTTPS cloning prompts for a password and fails — GitHub removed Git password auth in Aug 2021. Clone over SSH instead: `git clone git@github.com:Adipat2003/MetaGlassTest.git`. Ved's SSH key (`~/.ssh/id_ed25519`) is already registered as `VedEti123` and already has access. To make this automatic for all GitHub repos: `git config --global url."git@github.com:".insteadOf "https://github.com/"`.

## What it is
A **bare React Native 0.87** app (React 19.2.3, TypeScript 6, Node ≥22.11) with two hand-written Swift native modules bridging to iOS SDKs. Mock-first: the whole flow runs on the iOS Simulator with no glasses.

**This is the real codebase.** Everything in [[Projects/MetaGlasses/Overview|Overview]] describing planned architecture should be read against what actually exists here.

## Structure
```
App.tsx                              RN screen — bridge status UI, device list,
                                     display text field, LLM chat
src/wearables/MetaWearables.ts       platform-neutral TS API (graceful no-op
                                     when the native module is absent)
src/wearables/types.ts               WearablesStatus, WearableDevice, WearablesApi
src/llm/OfflineLlm.ts                TS API for the on-device model
src/llm/types.ts                     OfflineLlmStatus, MemoryDiagnostics
ios/MetaGlassTest/
  MetaWearablesModule.swift          DAT bridge (+ .m for ObjC export)
  OfflineLlmModule.swift             MediaPipe bridge (+ .m)
  AppDelegate.swift, Info.plist
android/                             generated, empty of custom work —
                                     "should mirror the public TypeScript API
                                     when Android support is added"
__tests__/                           Jest + react-test-renderer
AGENTS.md                            canonical repo instructions
CLAUDE.md                            one line: @AGENTS.md
DEVELOPMENT_SETUP.md                 clean-machine onboarding
```

## Native module 1 — `MetaWearables` (DAT bridge)
Imports `MWDATCore`, `MWDATDisplay`, `MWDATMockDevice`. **DAT pinned to 0.9.0** via Swift Package Manager (`facebook/meta-wearables-dat-ios`, rev `9b1b83d`).

Exposed to TypeScript as `WearablesApi`:
| Method | Does |
|---|---|
| `initialize()` | `Wearables.configure()`, enables MockDeviceKit with `initiallyRegistered: true` + `initialPermissionsGranted: true`, pairs `.rayBanMeta`, then `powerOn/unfold/don`. Returns a status dict |
| `register()` | `Wearables.shared.startRegistration()` if not already registered |
| `checkCameraPermission()` / `requestCameraPermission()` | DAT camera permission |
| `getDevices()` | Maps `Wearables.shared.devices` → id, name, connected, displayCapable, deviceType, linkState, compatibility |
| `connect(deviceId)` | `createSession(deviceSelector: SpecificDeviceSelector)` then `session.start()` |
| `disconnect()` | Stops display + session, then `doff/fold/powerOff` |
| `display.showText(text)` | `session.addDisplay()` → `display.start()` → `display.send(FlexBox { Text(text) })` |
| `display.clear()` | `display.clearDisplay()` |

The TS layer degrades gracefully — if `NativeModules.MetaWearables` is missing it returns an `unavailable` status rather than throwing, so Android/web don't crash.

**Camera capture — ✅ built and confirmed working, 2026-09-19** (scoped 2026-09-18 as part of the "glasses photo → chat attachment" feature; turned out buildable directly, same week, no blocker). `capturePhoto()` is now a real `@objc` method exposed as `MetaWearables.capturePhoto(): Promise<CapturedPhoto>`.

**One DAT capability per session, not one per photo.** The first cut started a fresh camera stream (`session.addCamera` → `stream.start()`) per call and tore it down (`stream.stop()`) after every photo. DAT rejects a second `addCamera` on the same session outright — `"A capability of this type is already active on the session"` — because `stream.stop()` stops the *stream*, not the capability registration. Fix: `addCamera` once, on the first `capturePhoto()` call; every call after that reuses the existing stream via `stream.capturePhoto(format: .jpeg)` directly. Only `disconnect()` tears it down.

**Mock Device Kit needs a feed, not just a result image.** `camera.setCapturedImage(fileURL:)` only sets what a capture *returns once the stream is already active* — with no feed configured, the stream has nothing to stream and goes straight to `.stopped` before ever reaching `.streaming`. Needs `camera.setCameraFeed(cameraFacing: .front)` too (the Mac's own camera, standing in for the glasses' — the argument label is `cameraFacing:`, confirmed by an actual compiler error, not the docs, which don't show the full signature).

**`Stream` needed disambiguating** — `MWDATCamera.Stream` collides with Foundation's own `Stream` (the Swift-native name for `NSStream`); both are in scope once you `import Foundation`.

**`MWDATCamera` wasn't linked as an SPM product at all** — added by hand-editing `project.pbxproj` (mirroring the existing `MWDATCore`/`MWDATDisplay` entries), since there's no Xcode GUI available in this environment to add it the normal way.

Full detail in [[Projects/MetaGlasses/Todo|Todo]]. Glance's own half of the feature (phone camera/library, already shipped) is documented in [[Projects/MetaGlasses/Glance|Glance]].

## Native module 3 — `AudioModule` (voice loop) — added 2026-08-28, branch `appDev`
Imports `AVFoundation` + `Speech`. **No DAT** — this is the publishable half. Promise-based `NSObject` matching the other two bridges; **holds no conversation state**, the JS layer drives the cycle.

| Method | Does |
|---|---|
| `requestPermissions()` | Mic (`AVAudioApplication`) + speech (`SFSpeechRecognizer`) — both platform dialogs, no Meta AI deeplink |
| `getStatus()` | Permissions, recognizer availability, on-device support, locale, **current audio route incl. `hasBluetoothHFPInput`** |
| `startListening(preferGlasses)` | `AVAudioEngine` tap → `SFSpeechAudioBufferRecognitionRequest`. Resolves with `{transcript, onDevice, heardSpeech}`. **Two timeouts** (see below); prefers on-device recognition when supported |
| `stopListening()` | Forces `endAudio()` so the pending listen resolves |
| `speak(text, preferGlasses)` | **Tears down capture first — this is the mic gate.** `AVSpeechSynthesizer`, resolves on `didFinish` |
| `stopSpeaking()` | Immediate stop |

**Endpointing needs two timeouts, not one** — learned the hard way 2026-08-28. First version armed a single 1.5s silence timer the moment capture started, so the wearer had 1.5s to *begin* speaking before it decided they'd stopped, resolved with an empty transcript, and returned to idle **with no error** (an empty result is a successful resolve). Looked exactly like a broken microphone.
- `startTimeout` **8s** — waiting for speech to begin; the tap-to-speak gap is human
- `silenceTimeout` **1.5s** — armed only *after* the first non-empty partial result, re-armed on each subsequent one; this is the latency felt every turn
- `heardSpeech` in the result distinguishes "no audio ever arrived" from "audio arrived, no words recognised" — different causes, and without it the failure is invisible

**Simulator microphone**: the iOS Simulator takes audio from the Mac, but only once enabled via Simulator menu → **I/O › Audio Input**. Until then it never requests macOS mic access, so it never appears in System Settings → Privacy & Security → Microphone. There is **no `simctl` equivalent** (`simctl privacy` only controls permissions *inside* the simulated iOS). A **"Test speak"** button in the Voice loop card runs TTS with no mic involved, to separate an input problem from a loop problem.

**AirPods are a faithful stand-in for the glasses on the audio path.** For audio purposes they are the same class of device — both present as `portType == .bluetoothHFP`. Testing with AirPods on a real iPhone exercises the exact code path the glasses will use: `.allowBluetoothHFP`, `setPreferredInput` on the HFP port, the A2DP→HFP switch when the mic opens, and 8 kHz mono reaching the recognizer. This means **the 8 kHz ASR-quality question can get an early answer without owning glasses**.

*Does not work on the Simulator* — no Bluetooth stack there; host audio is presented to iOS as `MicrophoneBuiltIn`, so no HFP port ever appears and `preferBluetooth` finds nothing.

The flag is named `preferBluetooth`, **not** `preferGlasses` (renamed 2026-08-28) — it selects any Bluetooth HFP input, and the honest name makes the AirPods path first-class. **Defaults false (phone mic)**. Step 4 of the build plan is flipping this toggle, not writing code.
- **`.allowBluetoothHFP` compiles on the iOS 26 SDK** — Meta's docs hedged ("if your SDK only exposes the older `.allowBluetooth`"); confirmed the modern symbol exists.
- TS API at `src/audio/Audio.ts`, types at `src/audio/types.ts`, degrades gracefully when the native module is absent (same pattern as `MetaWearables`).
- `Info.plist` gained `NSMicrophoneUsageDescription` + `NSSpeechRecognitionUsageDescription`.
- **Voice loop card** in `App.tsx` runs one full turn: listen → transcribe → (echo placeholder) → speak. Step 3 replaces the echo with `POST /v1/chat`.

## Native module 2 — `OfflineLlm` (on-device inference)
Imports `MediaPipeTasksGenAI`. Runs **Gemma 2 2B instruct, CPU int8** (`gemma2-2b-it-cpu-int8.bin`).

- Model is **not committed** — must be added to the Xcode target manually; the module reports `modelFound: false` with instructions otherwise. This is why CI passes without it.
- `maxTokens = 512`. Session opts: `topk 40`, `temperature 0.8`, `randomSeed 101`.
- CPU backend forced on simulator; `.defaultBackend` on device.
- Streams via `generateResponseAsync()`, accumulates, strips `<end_of_turn>`.
- Prompt template: `<start_of_turn>user\n{prompt}<end_of_turn>\n<start_of_turn>model\n`
- Reports `MemoryDiagnostics` — `physicalMemoryBytes`, `appResidentMemoryBytes`, `lowPowerModeEnabled`, `isSimulator`.

## Bundle identifier — changed 2026-08-29 (branch `appDev`)
`org.reactjs.native.example.MetaGlassTest` → **`vedeti123.metaglassesapp`** (both Debug and Release).

The old value was the stock React Native template ID, shared by thousands of tutorial projects. Free provisioning registers bundle IDs with Apple, so the template value would likely be rejected as already claimed — this blocks deploying to a physical iPhone.

- **DAT callback is unaffected** — the URL scheme is a hardcoded `metaglasstest` literal; only `CFBundleURLName` (a display label) derives from the bundle ID.
- ⚠️ **Check with Adipat**: if the Wearables Developer Center integration was registered against the old identifier, that registration now points at a different app. Would surface as registration failure on real hardware, not against the mock.
- Not reverse-DNS (`com.vedeti123.…` would be conventional) but legal — Apple allows alphanumerics, periods, hyphens.

## Info.plist
- `NSBluetoothAlwaysUsageDescription` ✅
- `CFBundleURLTypes` + `LSApplicationQueriesSchemes` — the Meta AI callback scheme for DAT registration ✅
- `UIBackgroundModes`: `bluetooth-peripheral`, **`external-accessory`**
- `NSLocationWhenInUseUsageDescription` present but **empty string**
- **No `NSMicrophoneUsageDescription`**

## Tooling
- CI: GitHub Actions on push/PR to `main` — format, tsc, ESLint, Jest. No glasses, model files, or secrets required.
- `bundle exec pod install --project-directory=ios` — **CocoaPods is here doing its actual job**: React Native's own iOS dependencies. It is *not* "translating React Native to Swift/Java" (see [[Projects/MetaGlasses/Overview|Overview]] — that claim was wrong, but the tool legitimately belongs in an RN project).
- Style: 2 spaces, LF, single quotes, trailing commas, 80 cols. `PascalCase` components/Swift types, `camelCase` functions.
- Known env constraints: RN 0.87 rejects Node 25 (use 22.13+/24.3+/26+); `sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer` required before native builds.

## Commit history
```
fc7b87a Add free GitHub Actions CI
b831260 Fix Gemma chat prompt formatting
1c6fb75 Add offline MediaPipe LLM chat
a9ec77d Build mock first wearable workflow
dc201f3 Add developer setup guide
92f17c5 Add shared Claude and Codex context
863ed66 Update .gitignore
4fd486e Add Meta Wearables bridge and mock device
6898863 Initialize React Native app and formatting
3fe1989 Initial commit
```

## How this differs from the planned architecture
| | Plan ([[Projects/MetaGlasses/Overview|Overview]] Rev 2) | This repo |
|---|---|---|
| Client | Swift/SwiftUI | **React Native + Swift bridge** |
| Interaction | Voice in, speech out | **Display + typed text** |
| Inference | Cloud (Claude, cascaded) | **On-device Gemma 2 2B** |
| DAT | Behind a build flag | Linked unconditionally |
| Audio | The core requirement | **None — zero AVFoundation anywhere** |

### Where the plan was wrong
1. **"DAT has no React Native support"** — Adipat wrote a clean working bridge. The plan's real concern was bridge-maintenance cost, not impossibility; that cost is now already paid, which changes the calculus.
2. **CocoaPods "cut"** — the *claim* about it was false, but the *tool* legitimately belongs here for RN's iOS deps. Cut the explanation, not the tool.
3. **MediaPipe/Gemma "not for v1"** — it's built, so there's real evidence to evaluate instead of speculation.

### Gaps to close
1. **Voice is not started**, and it's the part that needs no DAT — pure AVFoundation, slots in *alongside* the bridge, not into it. This is the core product requirement.
2. **App-Store-blocked as built** — `external-accessory` in `UIBackgroundModes` + DAT linked unconditionally. The build-flag split is far cheaper now (~10 files) than after the app grows around it.
3. **CONFIRMED 2026-08-26 — the display path cannot be tested on a simulator.** Verified two ways: by reading DAT 0.9.0's binary `.swiftinterface` files, and empirically by running the app.

   `MWDATCore.DeviceType` includes **`metaRayBanDisplay`**. `MWDATMockDevice.GlassesModel` does **not** — its only cases are `rayBanMeta`, `oakleyMetaHSTN`, `oakleyMetaVanguard`, `rayBanMetaOptics`, `metaGlasses`, `notEnabled`. The 0.9.0 CHANGELOG enumerates the same list. The string "display" appears nowhere in the MockDevice interface.

   Running the app confirms it: the mock device connects successfully and the Devices card reports **`connected · displayless`** — `device.supportsDisplay()` is false.

   **Not a bug in Adipat's code** — a gap in Meta's mock kit. Every mockable model is a non-display product. Consequence: **the display half of the app is hardware-only** until Meta ships a display-capable mock. This is a strong independent argument for voice-first ordering — the voice loop can be developed entirely on a simulator with the phone mic; the display half cannot be iterated on at all without a Ray-Ban Display.
4. **Gemma 2 2B int8 is ~2.5GB resident.** The memory diagnostics and `maxTokens: 512` read like someone already fighting memory pressure. For the voice loop specifically this is likely the wrong trade — cloud TTFT is ~400ms and ASR/TTS will be cloud anyway, weakening the offline/privacy argument for a local LLM.
5. `NSLocationWhenInUseUsageDescription` is an empty string — App Store review rejects empty purpose strings. Fill it or remove the key.
