# MetaGlasses — Reference

> **Note (2026-08-24)**: the project pivoted away from the cooking-app concept (see [[Projects/MetaGlasses/Overview|Overview]]). The Hardware and Developer Platform sections below remain valid general reference regardless of what gets built. The Competitive Landscape and Market Sizing sections describe the now-superseded cooking-app plan specifically — kept for history, not current strategy.

## Hardware: Ray-Ban Display ($799)
- 600×600px full-color display in the right lens, 90Hz refresh, 20° field of view, up to 5,000 nits brightness
- Pairs with the **Meta Neural Band** — an sEMG wristband for gesture control, IPX7, up to 18h battery
- 12MP ultra-wide camera, 3K video @30fps, 3x digital zoom; 2 open-ear speakers; 6 mics; 32GB storage; 2GB RAM; Wi-Fi 6; Bluetooth 5.3
- ~6h mixed-use battery (case adds up to 24h); 69g weight
- Other tiers: Ray-Ban Meta Gen 2 ($299, audio/camera only, no display); "Meta Glasses" line starting at $299

## Developer platform: Wearables Device Access Toolkit
Developer preview launched May 2026. Sign up at `developers.meta.com/wearables`.

**Two build paths — this is the key decision point for any feature:**

| | Web Apps | Native SDK (Device Access Toolkit) |
|---|---|---|
| Language | HTML/CSS/JS, deployed via HTTPS URL | Swift/Kotlin, extends an existing iOS/Android app |
| Camera/mic access | ❌ No | ✅ Yes (camera capped at 720p/30fps over Bluetooth, auto-reduces under bandwidth pressure) |
| Other sensors | Neural Band gesture input, phone GPS, motion/orientation, local storage | Same, plus camera/mic |
| App Store publishing | N/A — no store, URL-based distribution only | ❌ Currently blocked — SDK uses `ExternalAccessory` framework, causes App Store rejection under Apple's MFi program (confirmed in Meta's iOS docs as of 2026-08-03). Android also gated to select partners during preview. |
| Custom voice/gesture commands | Not available yet — only standard events (pause/resume/stop) | Same limitation |
| Compute location | Phone or a web server the phone bridges to | Phone |

**Important correction from research**: nothing runs "on" the glasses in either path. The glasses are strictly an I/O peripheral (camera in, mic in, display out, speaker out, EMG gesture in) — all compute happens on the phone or a backend.

Meta's suggested Web App use cases explicitly include cooking guides, transit tools, grocery lists, games, and instrument practice — cooking guides is a named example.

## Testing without owning hardware
- **Meta Ray-Ban Display Web App Simulator** — official Chrome extension: `chromewebstore.google.com/detail/meta-ray-ban-display-web/jpjlmmodokemlepklkdbimceggpbjcll`. Recreates the 600×600 display surface with additive blending, environment backgrounds (including live webcam blending), D-pad input, recording, and an automated QA checklist (viewport meta, favicon, D-pad focusability, horizontal overflow, focus styles). Docs: `wearables.developer.meta.com/docs/develop/webapps/test/`.
- **Mock Device Kit (`MWDATMockDevice`)** — a *different* tool from the Chrome extension above, and easy to conflate with it. It's part of the native Device Access Toolkit, not the Web Apps path, and it's the correct simulator if you're building a native iOS app that connects to and controls the glasses (see "Native iOS Display integration path (DAT)" below). The Chrome extension only previews the Web Apps path (HTML/CSS/JS over HTTPS); it does **not** emulate `iOS App → MWDAT SDK → Bluetooth/Meta connection → glasses`, so you can't point an iOS Simulator at the Chrome extension and have DAT treat it as the glasses.
- What stays hardware-only regardless: real Bluetooth pairing/connection-recovery, true optical display output (brightness/glare/legibility through the waveguide), actual Neural Band EMG gesture recognition (vs. simulated capacitive taps), physical camera sensor quality/latency, donning/doffing sensor edge cases, battery/thermal behavior, end-to-end wireless latency.
- **In-store demo required** before Meta will sell you a Ray-Ban Display unit (Best Buy, LensCrafters, Ray-Ban Stores, Verizon). Demo units won't load custom web apps but answer comfort/legibility questions the simulator can't.

## Distribution (no hardware needed)
- Share a link via the Meta AI app; recipients with Developer Mode enabled can add it in one tap. Coding tools can generate a QR code that launches straight into the phone app's Web Apps section.
- Where to find glasses owners for testing: Herald forum (`herald.ascents.gg`), `glassapps.io`, Meta developer GitHub discussions, smart-glasses subreddits.
- Meta's own storefront doesn't exist yet — general community publishing access is expected "sometime in 2026," with **Meta Connect (Sept 23–24, 2026)** the likely announcement venue. Ved is registered to attend.

## Competitive Landscape
- **Herald** (`herald.ascents.gg`) — an independent, unofficial catalog of ~110 apps across 11 categories for Ray-Ban Display, built by a group called Ascent (not affiliated with Meta). Has a review process, ratings, data-safety notes — but no payment rails; funded via Buy Me a Coffee, apps are free/donation-based only. **Reclassified 2026-08-24**: this is an app-store/platform, not a competitor — its own "AI generation app" isn't meant for complex tasks. The platform-mechanics facts (review process, no payment rails) likely still matter for distribution under the new agent-deployment direction.
- **Sous** (on Herald) — an existing app doing essentially the same core loop as Mise: recipe list, ingredients, pinch-through steps, per-step timers. Live since roughly Q2 2026, covered by BGR (2026-06-01). This is the direct competitor.
- Name collision warning: "Sous" is heavily reused — do not confuse the above with *Sous – AI Recipe Assistant* (iOS, Aurora Technologies), *Sous – Personalised Recipes* (iOS, subscription £4.79/mo), or *Sous Chef Recipes* (older Android/Google Nest Hub app). None are related to each other or to the glasses app.
- **Open wedge identified**: nobody currently combines a personal, imported recipe library with a glasses cooking mode — Sous (Herald) has no personal library; phone recipe apps have no glasses layer.
- Broader phone recipe app market: 91 of the top 100 US apps are free; category leaders include Paprika (15 years of users); adjacent "AI recipe import" competitors already crowding in include Fond, RecipeClip, and Recipe One.

## Market Sizing (from the pitch deck, "Glasses recipe market analysis")
- **TAM**: ~$1.6B global recipe/cooking app spend (2026), reaching $2.63B by 2030 at 13.1% CAGR. Conservative mid-point — estimates range $0.88B–$6.65B depending on whether meal planning/nutrition apps are included.
- **SAM**: ~$650M across English-speaking developed markets (North America alone ≈35% of global revenue, ~$560M).
- **SOM**: ~$4.5M ARR by year three (~0.6% of SAM).
- **Glasses-only addressable spend**: ~$3M. Basis: ~150K Ray-Ban Display units exist, international launch is currently paused, Meta's own two-year target is 500K units. This is why the deck's recommendation is phone-first with glasses as the differentiator, not a glasses-only business.

## Native iOS Display integration path (DAT) — 2026-08-25

The concrete, official way for a native iOS app to connect to and show content on the Ray-Ban Display. Distinct from — and not simulated by — the Chrome Web App extension above.

- **SDK**: open source at `facebook/meta-wearables-dat-ios`, added via Swift Package Manager.
- **Modules**: `MWDATCore` (device discovery, registration, session management), `MWDATDisplay` (send visual content to the lens — added to DAT in v0.7, May 2026), `MWDATMockDevice` (simulate a paired device without hardware).
- **App registration**: requires configuring a Meta app registration / URL callback scheme in `Info.plist` — the SDK uses a URL scheme to talk to the Meta AI app during registration/connection.
- **Reference sample**: Meta's official `DisplayAccess` iOS sample demonstrates the full connect → register → session → send-content flow. Recommended starting point over a blank Xcode project.

Architecture:
```
iPhone
  └─ Your iOS App (Swift/SwiftUI)
       ├─ MWDATCore     (device discovery, session, registration)
       └─ MWDATDisplay  (send visual content to the lens)
            │  Meta Wearables connection
            ▼
     Ray-Ban Display (right-lens display)
```

**Simulating without hardware** — `MWDATMockDevice`:
```swift
import MWDATMockDevice

let mockDeviceKit = MockDeviceKit.shared
mockDeviceKit.enable()
let mockDevice = try mockDeviceKit.pairGlasses(model: .rayBanMeta)

mockDevice.powerOn()
mockDevice.unfold()
mockDevice.don()
// ...
mockDevice.doff()
mockDevice.fold()
mockDevice.powerOff()
```

**Recommended build sequence:**
1. **Stage 1 — connection only.** Minimal SwiftUI app (`RayBanDemoApp.swift`, `ContentView.swift`, `WearablesManager.swift`). Wire up `MWDATCore` against a `MWDATMockDevice`-simulated device. Milestone: Connect → device discovered → registered → session established. No app logic yet.
2. **Stage 2 — add Display.** Bring in `MWDATDisplay`, open a display session, send content to the lens. Use the `DisplayAccess` sample as reference.

**Dev setup for this path** (two independent simulators, for two independent paths — don't conflate them):
```
Mac
├── Xcode → iOS Simulator → Your iOS app → MWDATMockDevice      (native/DAT path)
└── Chrome → Meta Ray-Ban Display Web App Simulator              (Web Apps path)
```

**Explicitly advised against**: reverse-engineering the Chrome extension or building a custom Bluetooth protocol — Meta officially supports Display through DAT, so there's no need to.

**Open question, not addressed by this research**: whether DAT-built apps can currently publish to the App Store. Earlier cooking-app-phase research (see Developer Platform table above) found DAT's use of `ExternalAccessory` currently blocks App Store publishing under Apple's MFi program — this 2026-08-25 note doesn't revisit that constraint, so it should be reconfirmed before committing deeply to the native path.

## Agent-platform stack (candidates, unconfirmed — 2026-08-24)
Raised in the pivot meeting as directions to evaluate, none decided yet:
- **React Native** — share code across iOS/Android/web frontends
- **Expo** — UI rendering layer within React Native
- **CocoaPods** — noted as bridging/converting React Native code to native Swift/Java; this is rough shorthand from the meeting (CocoaPods is actually iOS's native dependency manager) and should be verified before being treated as architecture
- **iOS Simulator**, **Android Studio** — standard native dev tooling, explicitly listed alongside the above
- **MediaPipe** (Google) — taking an LLM call from a React app and running it on-device on the phone
- **Gemma** (Google) — candidate on-device small LLM
- Phone-to-glasses bridge/transport — **resolved for the native iOS path** as of 2026-08-25: Meta's own DAT SDK (`MWDATCore`/`MWDATDisplay`), see above. Still open whether/how a React Native (cross-platform) app would use the same SDK, since DAT as described here is iOS-native (Swift/SwiftUI).

Secondhand research cited in the meeting (unverified): running LLMs locally on iPhone is inefficient; vision tasks need hardware acceleration even on-phone.

## Verified capability matrix (2026-08-25, from Meta's docs via Wearables MCP)

| Capability | Web Apps | Device Access Toolkit |
|---|---|---|
| Display | 600×600 additive waveguide, no scrolling | SDK components; full-view replace per `send()` |
| Input | Arrows + Enter only; every element must be focusable | Captouch + EMG gesture events |
| Microphone | ❌ | ✅ HFP, 8kHz mono, beamformed to wearer |
| Camera | ❌ | ✅ streams + single-frame capture |
| Speaker | ❌ | ✅ A2DP |
| IMU / compass | ✅ `DeviceMotionEvent`, `DeviceOrientationEvent` | ✅ |
| Location | ✅ `navigator.geolocation` (from phone) | ✅ |
| Storage | `localStorage`, lightweight only | Phone-side, unrestricted |
| Text input | Docs say ❌ — **contradicted by bundled skill, verify** | Phone-side |
| Offline / notifications / back-nav | ❌ | Phone-side |
| Distribution | Any HTTPS URL, no store, no review | Invite-only release channels; **App Store blocked** |
| Language | HTML/CSS/JS | Swift or Kotlin — **no React Native support** |

Also verified:
- Display **dims at 20s inactivity, sleeps at 25s**; sleep does not end a DAT session.
- Additive display: **pure black = fully transparent**. Dark grounds + bright high-contrast type is a requirement. Body text min 16px, primary content 20–24px.
- DAT `send()`/`sendContent()` **replaces the entire display** — no partial updates. Glasses retain no app state; the phone/backend is always the source of truth.
- DAT permissions requiring Meta review: camera, microphone, voice invocation.
- Required Web App `<head>` metadata: `<meta name="mrbd-web-app-capable" content="yes">` plus a description.
- App icons: PNG ≥52×52 or Unicode. **SVG not supported.**

## Glasses audio — the voice loop (2026-08-25)

**Uses no DAT APIs.** All plain AVFoundation; for audio the glasses are an ordinary Bluetooth headset. This is what makes a *publishable* voice agent possible.

| Profile | Direction | Quality | Use |
|---|---|---|---|
| **A2DP** | Output only | 44.1/48 kHz stereo | Music, TTS playback |
| **HFP** | Bidirectional | 8 kHz mono | Mic capture from the wearer |

- **Mutually exclusive**: *"Activating HFP switches the glasses away from A2DP, and audio output quality drops to 8 kHz mono for the duration of the session."* Route takes ~2s to settle after switching.
- Mic **beamforms onto the wearer's voice**, heavily suppressing ambient sound and other speakers. Expected behavior, not a bug — and it means the agent can't hear the room.
- Mic permission is a **standard iOS/Android platform dialog** (`AVAudioApplication.requestRecordPermission`), *not* a DAT permission. Camera is the DAT-gated one.
- DAT sessions *share* mic/speaker access with the system Bluetooth stack.

```swift
// Mic in — HFP
let s = AVAudioSession.sharedInstance()
try s.setCategory(.playAndRecord, mode: .default, options: [.allowBluetoothHFP])
try s.setActive(true, options: .notifyOthersOnDeactivation)
if let hfp = s.availableInputs?.first(where: { $0.portType == .bluetoothHFP }) {
    try s.setPreferredInput(hfp)
}
audioEngine.inputNode.installTap(onBus: 0, bufferSize: 1024, format: fmt) { buf, _ in
    handleAudioBuffer(buf)   // → your pipeline (Meta's docs name LiveKit as an example)
}
// Verify the route actually landed on the glasses before trusting it:
// s.currentRoute.inputs.contains { $0.portType == .bluetoothHFP }
```

- **Ordering constraint** if combining HFP with a DAT camera stream: add the camera stream → configure and start HFP → wait for route to settle → *then* start the stream. Out of order, audio fails **silently**.

## On-device speech recognition — endpointing (2026-09-04)
Learned the hard way while building Glance's voice loop. Applies to any `SFSpeechRecognizer` client, not just this app.

- ⚠️ **A transcript that keeps changing does not mean someone is still talking.** Recognition continues emitting *changed* partial results long after the speaker stops, refining and rewriting what it already has. Any end-of-turn countdown re-armed on "the transcript changed" therefore **never expires** in a room that is not silent — which is every room, and doubly so on an 8 kHz HFP mic that picks up its own noise floor. The failure looks exactly like a broken timer: manual stop works, automatic stop never fires.
- **Endpoint on microphone level instead.** RMS per capture buffer (`vDSP_measqv`) against an adaptive noise floor that drops to any new quiet instantly and climbs back very slowly. Steady room noise settles *into* the floor and stops counting as voice; speech, being loud and intermittent, never does. Voice = level above `max(floor × 3.5, 0.004)` — the absolute term stops a near-silent room treating its own hiss as speech once the floor converges toward zero.
- A transcript that **gains a word** is still worth keeping as a secondary signal, for quiet speech the level detector under-reads. Merely *changing* is not.
- **`SFSpeechRecognitionTask` expires at roughly 60 seconds.** Continuous listening must restart the task and stitch transcripts across the seam; a single task cannot run indefinitely.
- **An empty result can arrive last.** Cancelling a task is the normal end of every capture and may deliver one final result with nothing in it — never let it overwrite what was already heard.
- **Restarting capture is not synchronous.** Cancelling a task and switching the audio route both take time; starting a new capture immediately (exactly what a wake word handing over to dictation does) races the teardown and yields silence. Bluetooth needs ~1.1s to select the HFP input, versus ~0.35s for the built-in mic. Anything guarding on "is a task running" is **blind during that window** — guard on an explicit starting flag too.

## Voice agent stack (2026-08-25)

**Cascaded pipeline (STT → LLM → TTS), not end-to-end speech-to-speech.** Reason: this is an agent platform, so tool calling is central, and S2S suits shallow tool surfaces. Cascaded also gives per-stage debugging, vendor swapping, predictable cost, and lets you pick an ASR that handles 8 kHz telephony audio. Industry consensus for 2026 is that cascaded dominates production.

| Slot | Pick | Notes |
|---|---|---|
| Backend language | **Python** | Both voice frameworks are Python-first |
| Voice framework | **LiveKit Agents** | WebRTC transport, barge-in/reconnect from the RTC layer, Swift SDK. Meta's DAT docs name LiveKit as an example buffer destination. *Alt: Pipecat* if you want transport-agnostic swappable stages |
| ASR | **Benchmark two** | AssemblyAI Universal-3.5 Pro Realtime (6.99% pooled WER) vs Deepgram Nova-3 (trained on telephony). Our 8 kHz beamformed input is unusual — published rankings don't settle it |
| LLM | ~~**`claude-opus-5`**~~ | 1M ctx, $5/$25 per MTok. Tune latency with `effort`, not by downgrading. ⚠️ **Not what got built** — the backend calls OpenAI `gpt-5.6-sol`; see "The backend as built" below. Glance's BYO-key path still offers Anthropic/OpenAI/Google, so this row describes the client tier only |
| TTS | **Cartesia Sonic-3** | 188ms P50 TTFA. *Optimize latency + price only* — see below |
| Tools | **MCP** | Standard protocol + existing server ecosystem; authoring = picking servers. Claude connects natively: pass `mcp_servers` alongside `tools: [{type: "mcp_toolset", mcp_server_name: ...}]`. Third parties can extend the platform without you shipping code. |
| Agent store | **Postgres** | Definitions, versions, owners |
| Console | **Next.js** | Plain web, built last |

**Key insight — don't pay for premium TTS.** While the mic is open, HFP caps *output* at 8 kHz mono too. The wearer hears everything through a telephone-grade codec, so voice fidelity differences are inaudible on this device.

**Claude latency levers**, in order: `output_config: {effort: "low"}` (primary; better than disabling thinking — on Opus 5 that can make it write tool calls into visible text); **prompt caching** (agent instructions + tool defs are byte-identical every turn — order is `tools`→`system`→`messages`, verify via `usage.cache_read_input_tokens`); **streaming** (non-negotiable — start TTS on sentence one); model tier per-role (`claude-haiku-4-5` for cheap routing/classification only). Use the SDK **tool runner** for the loop.

**Two agent modes** worth deciding between early — they share the definition but nothing else:
- *Synchronous voice turn* — Messages API, low effort, cached, latency-critical
- *Asynchronous background task* — Managed Agents (persisted versioned configs, hosted sandbox, scheduled deployments). Latency-irrelevant. Arguably the more wearable-native interaction: ask, walk away, get told when done.

**Infrastructure constraint**: the agent runtime **cannot be serverless** — a voice session is a long-lived stateful process holding an audio stream; cold starts alone blow the budget. Needs persistent workers (Fly.io / Railway / Render / ECS) + LiveKit Cloud for the media plane. Deploy console separately from voice workers.

### Latency budget (estimates, ~1,340ms naive)
Natural conversation wants <800ms end-of-speech → first syllable.

| Stage | ms | Tunable? |
|---|---|---|
| LLM first token | 400 | ✅ effort, caching, streaming |
| Endpoint detection | 250 | ✅ **cheapest win** — pure waiting |
| TTS first audio | 190 | ✅ vendor |
| ASR finalize | 150 | ✅ vendor |
| Bluetooth → ear | 150 | ❌ fixed |
| Mic → Bluetooth | 100 | ❌ fixed |
| Network (both legs) | 100 | ❌ fixed |

**350ms is untouchable** (Bluetooth both ways + network) — the glasses tax a phone-only agent never pays. **990ms is yours**; ~550ms has to come out of it to feel natural.

## The codebase
`Adipat2003/MetaGlassTest` — React Native + Swift bridges to DAT 0.9.0 and MediaPipe. Full documentation, structure, API surface, and known gaps: [[Projects/MetaGlasses/Repository|Repository]].

Clone over **SSH** (`git@github.com:Adipat2003/MetaGlassTest.git`) — the repo is private and HTTPS password auth was removed by GitHub in 2021.

## API contract — draft v0.1 (2026-08-28)
Interface between phone app (Ved), web app (Milos), backend (Adi). **Adi owns it**; this is a starting draft to accept/amend:
https://claude.ai/code/artifact/fb920b8c-48c9-41db-a0e6-352a3f565f79

Encodes Ved's 2026-08-28 decisions: **backend is stateless** (no history), **phone owns all chats locally** (no cloud storage cost), **lens shows latest response only**, **STT/TTS on-device**, **audio is phone↔glasses Bluetooth only**.

**No audio crosses this API — every field is text.** STT and TTS both run on the phone; the backend never sees audio and never stores a transcript.

Three endpoints: `POST /v1/chat` (full transcript in → `{text}` out), `GET /v1/display?token=` (web app polls), `POST /v1/state` (phone drives lens status).

**Single `text` field** (v0.2, Ved's call): the same string is spoken by the phone and shown on the lens. Keeping it short is the *system prompt's* job — one step, one short sentence — which is right for a step-by-step assistant anyway. Safety net: web app truncates client-side. Adding a separate short `display` field later is additive, not breaking.

**Pairing**: phone mints `https://<host>/w/<pairingToken>`; user adds that URL once as a Web App. Web app reads the token from its own path — no typing needed, which matters because Web Apps have no text input. Backend keeps one tiny record per token (latest response + state, idle-expired) — the only state it holds.

**Big consequence**: with STT/TTS on-device and no server state, the backend is a thin wrapper around one Claude call. The **LiveKit/Pipecat decision is now moot** — cross it off the stack list.

### Decisions
1. ~~Separate `display` field~~ — **resolved: one `text` field.** See above.
2. ~~Barge-in in v1?~~ — **resolved 2026-08-28: no barge-in, mic gated during TTS.** Speakers and mics are centimetres apart, so an open mic during playback transcribes the agent's own voice. Gating is the simpler build — no echo cancellation, no `setVoiceProcessingEnabled`, no half-duplex tuning. Cost: the wearer can't interrupt, which is acceptable when every answer is one short sentence.

**Audio state machine** (mirrors the API's `state` field exactly):
```
idle → listening → thinking → speaking → idle
       mic OPEN    mic OPEN   mic GATED
```
The lens can show "Speaking…" so the wearer knows talking now won't be heard.

### Pairing
Phone mints `pairingToken` (32-hex, local-only) → builds `https://<host>/w/<token>` → user adds it once in Meta AI app (App Connections → Web Apps) → web app reads the token from its own URL path. No typing, which matters because **Web Apps have no text input**. Both clients then present the same token and the backend correlates them. Diagrammed in the system map.

**Watch, don't decide yet**: phone sends full transcript every turn + stores chats forever = unbounded growth, paid per turn. Agree that trimming happens *on the phone* (last N turns / token budget), not silently server-side.

## The backend as built — v0.1, 2026-09-04
`metaglasses-backend/` (Adi), FastAPI + Python 3.12, `uv` for deps, ruff + pytest, CI publishing containers to `ghcr.io/adipat2003/metaglasses-backend`. Reconciliation plan against Glance — D1–D15 discrepancies, A1–A6 asks:
https://claude.ai/code/artifact/a58cd0ce-75ab-41a4-9d61-9631bf2e4b56

**Routes.** `POST /v1/chat` · `GET /v1/display?token=` · `POST /v1/state` · `GET /healthz`. The two phone routes need a Supabase bearer token; `/v1/display` needs none — the pairing token *is* the credential.

**Wire shapes**, which happen to match Glance's existing types exactly:
```
POST /v1/chat  { pairingToken: <32 hex>, messages: [{role: "user"|"assistant", content: str}] }
            →  { responseId, text, createdAt }
POST /v1/state { pairingToken, state: "idle"|"listening"|"thinking"|"speaking" }  → 204
GET  /v1/display?token=  → { responseId, text, state, createdAt }   (nulls before first response)
```

**Auth is Supabase, not ours.** The phone signs in directly with Supabase; FastAPI is a pure resource server that verifies the JWT against the project's **public JWKS**, checking signature, algorithm (ES256/RS256 — legacy HS256 deliberately rejected), issuer, audience, expiry, subject and the `authenticated` role. It never needs and must never receive a service-role key. **The password must never reach FastAPI.**

**Four environments**, because the Supabase free tier allows two hosted projects: `local` (no auth, tests), `dev` (Supabase CLI on :54321), `trial` (hosted, TestFlight), `prod` (hosted). Auth **cannot** be disabled outside `local`, and trial/prod reject a wildcard CORS origin — both enforced at startup, not by convention. Each environment needs its own project URL + publishable key in the matching build; publishable keys are public by design and belong in the app.

**Limits and failure modes worth knowing before designing against it:**
- **256,000 UTF-8 bytes** of serialized transcript, else `413`. A turn *count* is not a budget against this.
- Errors: `401` bad/absent token · `403` pairing owned by another user · `413` transcript · `422` any malformed field · `429` provider rate limit · `503` provider unavailable *or unconfigured*.
- **Pairing records are process-local and ephemeral** — an in-memory dict, 1hr TTL measured from the last *phone* request (lens polling does not refresh it), wiped by any redeploy. This is why the deployment is pinned to **one replica**; a second needs a shared TTL store first.
- `set_state` **recreates a missing record** rather than failing, so a phone posting state on foreground silently re-pairs itself. That is what makes the ephemerality survivable.
- **The pairing token itself must be permanent** — it lives in the Web App URL the user added by hand, and Web Apps have no text input to re-enter one.
- `/v1/chat` writes lens state as a side effect (`thinking`, then `speaking`, or `idle` on error). ⚠️ It flips to `speaking` when the *text is saved*, which is before the phone has synthesised anything. **Still unresolved (A3)**: Glance now also reports state itself via `POST /v1/state` (see [[Projects/MetaGlasses/Glance|Glance]]), so both sides write it — whether the server should stop is an open ask for Adi, not yet decided.

**Model**: OpenAI via the Responses API — `gpt-5.6-sol` by default (`OPENAI_MODEL` overrides), `reasoning: {effort: "low"}`, `max_output_tokens: 160`, `store: false`, prompt-cache key set, `safety_identifier` derived as a SHA-256 of the pairing token so no user identifier leaves the service. ⚠️ **This silently reverses the `claude-opus-5` pick in the stack table above** — Adi's key, Adi's call, but the stack note now describes something nobody is building. ⚠️ **Still unverified as of 2026-09-04**: no OpenAI key has been available in testing, so whether `gpt-5.6-sol` actually resolves remains unconfirmed — every backend-tier chat currently 503s for this reason alone.

**✅ Fixed 2026-09-04** (commit `e5fe88c`, branch `integration`): the system prompt was hardcoded server-side and knew nothing about Glance's agents — any agent's own `instructions` were silently dropped on this path. `ChatRequest` now takes an optional `system` field, appended to (not replacing) the server's own lens-brevity prompt, and Glance sends it on both the BYO-key and backend tiers. Also reworded the `/v1/display` 401 copy, which previously read as "re-pair" — misleading, since the pairing *token* must stay permanent (see the pairing section above) and only the *record* is meant to expire.

## System map (2026-08-26)
Every path between backend, phone, glasses and lens — solid = built, dotted = alternative, dashed = upcoming:
https://claude.ai/code/artifact/5add558c-c301-4948-a350-6dd42575e3ee

**The asymmetry that drives everything**: audio has exactly one route to the glasses and it needs **no DAT** (standard Bluetooth HFP/A2DP). The lens has **two** routes — DAT from a native app, or a Web App the glasses browser fetches directly — and the native one requires DAT, which forfeits App Store eligibility. The build-flag seam isn't arbitrary; it's the line the platform itself draws.

Also settled 2026-08-26:
- **Display requires DAT for any native app.** No standard Bluetooth profile exists for it — the SDK serializes declarative layouts (`FlexBox`, `Text`, `Image`, `Button`, `Icon`, `Video`/MP4-from-URL) and transmits them proprietarily. It is *not* a pixel stream; `send()` replaces the whole view, no partial updates, vertical scroll only.
- **A Web App is fetched by the glasses' own browser**, not rendered or proxied by the phone app. It works whether or not your app is installed. It can call your backend directly (REST + WebSockets both supported) — budgets: <10 network requests on load, <500KB JS gzipped, <128MB memory, and design for intermittent Wi-Fi.
- **Mic permission is a platform dialog, not a DAT permission** — Meta's integration-lifecycle docs draw the contrast explicitly against camera, which does deeplink to the Meta AI app. Caveat: the developer center still lists microphone among reviewable DAT permissions, so confirm on hardware. *Cheap test: pair the glasses and record in Voice Memos — if a stock Apple app captures glasses audio, any app can.*
- **API keys belong on the backend**, not the phone app.

## Architecture references (2026-08-25)
- **Rev 2 — three routes to the wearer**, the DAT build-flag split, layered system, voice loop:
  https://claude.ai/code/artifact/f2eb5bfa-dadf-4ba4-98af-d896df6505df
- **Voice stack decisions** — pipeline shape, vendor picks, latency budget:
  https://claude.ai/code/artifact/eb97e9f2-2333-4cfb-910c-8ce2a9a02b1a

Sources cited in the voice-stack artifact's footer (latency/vendor benchmarks — engineering estimates, not measurements of this system):
[Pipecat vs LiveKit](https://www.evalgent.com/blog/pipecat-vs-livekit) · [Voice agent frameworks](https://soniox.com/wiki/voice-agent-frameworks) · [STT benchmarks 2026](https://futureagi.com/blog/speech-to-text-apis-in-2026-benchmarks-pricing-developer-s-decision-guide/) · [STT for voice agents](https://www.autointerviewai.com/blog/best-stt-speech-to-text-models-voice-agents-2026) · [TTS latency benchmark](https://gradium.ai/content/tts-latency-benchmark-2026) · [Cascaded vs speech-to-speech](https://gradium.ai/content/cascaded-voice-agent-vs-speech-to-speech-2026) · [Speech-to-speech for voice agents](https://www.assemblyai.com/blog/speech-to-speech-for-voice-agents)

## iOS/RN gotchas worth not re-deriving

- **React Navigation nested params must sit under `params`.** `navigate('Tabs', { screen: 'Notes', autoRecord: true })` puts `autoRecord` on the *Tabs* route; the child screen never sees it. Correct form is `navigate('Tabs', { screen: 'Notes', params: { autoRecord: true } })`. Cost a whole debugging round where "navigates but does nothing" looked like two separate bugs.
- **A one-shot flag in a `useRef` never re-fires on a tab screen.** Tabs stay mounted for the session, so "handled" is permanent. Pass a token (timestamp) and compare against the last one handled.
- **`DEVELOPMENT_TEAM` is not a secret.** An Apple Team ID is public — it is embedded in the code signature of every shipped app and readable with `codesign -dv`. Signing requires team membership plus a certificate and private key; the ID alone grants nothing. Keeping it out of a shared project file is about **per-developer churn**, not secrecy.
- **Only an Organization Apple Developer membership can have team members.** Individual is one person, permanently; upgrading needs a D-U-N-S number. A collaborator outside the team can still build free on their own device with a Personal Team (7-day provisioning, no push, 3 apps).
- **`.gitignore` does nothing to an already-tracked file**, and git ignores files rather than lines — so it can never exclude one setting from a file the project needs.
- **`xcodebuild error 65` with no compiler error usually means the disk is full** (`lipo: No space left on device` appears far up the log). Hit twice.
- **Open `.xcworkspace`, never `.xcodeproj`, in a CocoaPods project** — the `.xcodeproj` builds into a different DerivedData directory with no Pods, and fails with missing modulemaps.

## Useful links
- Meta Wearables MCP endpoint: `https://mcp.developer.meta.com/wearables` — **wired up and working in Claude Code** as of 2026-08-25. Exposes `search_dat_docs` and `search_webapps_docs` (use the latter for the current Web Apps path). Not usable from claude.ai chat.
- Dev signup: `developers.meta.com/wearables`
- Web Apps AI coding plugin: `github.com/facebookincubator/meta-wearables-webapp`
- Getting a Web App onto glasses: Meta AI app → App Settings → App Connections → Web Apps → Add a Web App (requires Developer Mode)
- DAT toolkit capabilities overview: `wearables.developer.meta.com/docs/develop/dat/getting-started-toolkit`
