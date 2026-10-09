# MetaGlasses — Log

## 2026-08-08 — Initial platform exploration
Learned the Ray-Ban Display capabilities and developer platform state; corrected a misconception that apps run "on" the glasses (they don't — glasses are strictly an I/O peripheral, compute happens on phone/backend). Signed up for Meta Connect (Sept 23–24, 2026). Created a Meta developer account. Clarified Web Apps (HTML/CSS/JS, display-only, no camera/mic) vs. native Device Access Toolkit (camera/mic, extends existing mobile app) as the two build paths. Learned video-to-display became supported as of the May 2026 SDK update. Brainstormed four app ideas — real-time AI fashion, voice note-taking, camera+audio conference summarization, running telemetry — and scored them against sensor requirements. Concluded running telemetry was the best MVP: Web App path only, no camera/mic needed, matches Meta's own suggested use cases, fastest to a working prototype.

## 2026-08-17 — Pivot to cooking recipe concept, first prototype
Conversation opened with "help me get started developing for the glasses display" and went straight to a **cooking recipe app** idea instead of the running-telemetry MVP concluded on 08-08 (reason for the pivot not recorded). Confirmed Web Apps is the right path (no camera needed, Meta explicitly names cooking guides as a use case). Built a first prototype ("Index"): ingredient list + one-step-at-a-time navigation, using arrow-key/Enter as a stand-in for the real Neural Band input events. Explored Meta's Mock Device Kit / Wearables MCP endpoint (`search_dat_docs`) — determined both are irrelevant to this prototype since it's Web Apps-only with no camera/DAT dependency; MCP endpoint noted as usable only via Claude Code, not claude.ai chat.

## 2026-08-17 to 2026-08-18 — Market analysis, "Mise" prototype, and go-to-market decision
Ran a full market analysis for the cooking recipe glasses app concept and built a 13-slide, 5-minute pitch deck ("Glasses recipe market analysis," PPTX) with TAM $1.6B / SAM $650M / SOM $4.5M ARR by year 3, and a glasses-only addressable market of just ~$3M given ~150K existing Ray-Ban Display units. Identified a direct competitor, **Sous** (on the unofficial Herald app catalog), already shipping the same core loop — and the open wedge of combining a personal recipe library with a glasses cooking mode, which nobody has done yet. Worked through the App Store publishing question: native SDK + glasses currently can't ship to the App Store (MFi/ExternalAccessory conflict), so decided on a split architecture — plain phone app (App Store, no Meta SDK, the actual business) plus a separate lightweight glasses Web App ("Mise") as a free differentiator/demo layer, built first since it's cheap (~2–3 weeks) and produces a demo asset for press/investors before the phone app (months of work) gets built. Built the Mise v1 prototype: single-file HTML, webcam-based simulator with screen-blend compositing to preview the in-lens look without hardware, 3 recipes, fraction-aware serving scaler, per-step timers, peripherally-legible typography. Found Meta's official Web App Simulator Chrome extension for proper QA/testing without owning hardware, and the distribution path (Meta AI app link-share + QR code) for getting it onto a real device via someone else's glasses.

## 2026-08-24 — First local build of the web app
Built a basic v1 of Mise locally at `/Users/vedeti123/Documents/MetaGlasses/index.html` — this is a fresh implementation of the concept described in the 08-17/08-18 chats (that code was never saved to disk), scoped down per Ved's request to "basic, simple features": 3 prepopulated recipes, step-by-step navigation, no serving scaler. Kept the glasses-display simulator (webcam composited via CSS screen-blend under a 600×600 panel) and the corner-timer-persists-across-steps behavior from the original concept.

Tested end-to-end in a real browser via Claude in Chrome, driving it with actual keyboard events (arrow keys + Enter) to match how Neural Band input arrives on a real Web App: recipe select → ingredient scroll → step-by-step with live countdown timer → done screen → back to recipes, all confirmed working. Found and fixed one real bug during testing: the simulator's size/position sliders set inline styles on the lens wrapper, which silently beat the plain CSS rule meant to reset position/size in full-viewport "device mode" (`?display=1`) — fixed by adding `!important` to that override. Added a README with run instructions (needs a local HTTP server, not `file://`, for webcam access) and known limitations.

(Separately, after this, the local `index.html`/`README.md` were refined further outside this conversation — fixed 600×600 layout, custom webcam simulator dropped in favor of Meta's official simulator extension, click-based nav buttons added. Not clear whether that happened before or after the pivot below was decided.)

## 2026-08-24 — Pivot: from cooking app to an agent-deployment platform
Ved added meeting notes to the vault Inbox; processed into the project notes here. Key outcomes:

- **Herald corrected**: it's an app-store/platform for the glasses, not a competing product — its own AI-generation app isn't built for complex tasks. Updates the earlier competitive-analysis framing in [[Projects/MetaGlasses/Reference|Reference]].
- Open question raised: what sensor/capability access is actually available on the glasses (photos, video, microphone) — flagged developer forums as worth searching.
- TODO raised: check whether there's a phone connection that lets agent compute run on-phone rather than needing everything server-side.
- **Major pivot decided**: dropping the cooking app entirely. New direction is a platform that lets users create their own AI agents on phone or web, then deploy those agents to run on the Meta glasses. Full scope of what an "agent" means here wasn't spelled out in the notes.
- Secondhand technical research noted (via Reddit threads, not independently verified): running LLMs locally on iPhone is inefficient; vision tasks need hardware acceleration even on-phone.
- Candidate tools discussed, explicitly **not confirmed**: MediaPipe (Google) for bridging an LLM call from a React app to run on-phone; Gemma models as the candidate on-device LLM; React Native to share code across iOS/Android/web; Expo for UI rendering; CocoaPods described (loosely) as translating React Native code down to native Swift/Java.
- Notes included blank placeholders for Frontend Mobile (React Native iOS), Frontend Web, and Backend ("substitute for Bluetooth") — none filled in.
- Explicit next tasks called out: make a flowchart connecting the components, install platforms, create agents. Timeline and level-of-effort sections were left blank.
- Tool list was later filled in (still marked "not 100% confirmed"): React Native, Expo, CocoaPods, iOS Simulator, Android Studio.

## 2026-08-25 — DAT connectivity path identified (follow-up research from the 2026-08-24 meeting)
Ved appended further notes to the same inbox note, from a separate GPT session researching how to actually connect a native app to the Ray-Ban Display. Key outcome: the "phone connection" question raised on 2026-08-24 now has a concrete answer for a native iOS build.

- **Two dev paths reaffirmed and sharpened**: Web Apps (HTML/CSS/JS, previewed by the Chrome "Web App Simulator" extension) vs. Meta Wearables **Device Access Toolkit (DAT)** — a native iOS/Android app that connects to the glasses and can access Display (DAT gained Display support in v0.7, May 2026). For an iOS app meant to connect to and control the Ray-Ban Display directly, DAT is the correct path, not the Chrome extension.
- **Important correction**: the Chrome Web App Simulator extension does *not* simulate the DAT/native connection path at all (it only renders the Web Apps path) — easy to conflate since both are called "simulators." The actual simulator for DAT development without hardware is `MWDATMockDevice` (Mock Device Kit), which can pair/power/don/doff a simulated glasses device.
- Concrete SDK path: `facebook/meta-wearables-dat-ios` via Swift Package Manager, `MWDATCore` + `MWDATDisplay` modules, `Info.plist` app-registration/URL-scheme setup, and Meta's official `DisplayAccess` sample as the recommended starting point.
- Concrete two-stage build plan proposed: Stage 1, get a bare connection working (SwiftUI app + `MWDATCore` + `MWDATMockDevice`, milestone = session established, no app logic); Stage 2, add `MWDATDisplay` and start sending content to the lens.
- Explicitly advised against reverse-engineering the Chrome extension or building a custom Bluetooth protocol, since Meta officially supports this through DAT.
- Not addressed by this note, and worth reconfirming: whether DAT-built apps can currently publish to the App Store — earlier cooking-app-era research found this blocked, and that constraint would directly affect how an agent-deployment platform built this way ever reaches real users.

## 2026-08-25 — Architecture decided; cooking app archived
Ved asked to archive the cooking app and settle the architecture, saying the tooling had become confusing — unclear which tools belonged to which platform and where each piece fit. Facts were verified directly against Meta's docs via the Wearables MCP server (`search_dat_docs` / `search_webapps_docs`) rather than relying on the secondhand meeting notes.

- **Mise archived**: moved to `MetaGlasses/archive/mise-cooking-app/` with an `ARCHIVED.md` recording what's reusable (five-input nav model, 600×600 no-scroll layout, additive-display typography, persistent corner-badge pattern) vs. what isn't. Working directory top level cleared for the new project.
- **The App Store question got a definitive answer** — still blocked, quoted verbatim from Meta's docs. DAT's only distribution is invite-only release channels (email invites, Meta accounts required, one registered dev-mode app at a time). This single fact decided the architecture.
- **Decision: Web Apps, not DAT.** Distribution beats capability for a platform. An agent = a URL, served by one generic shell parameterized by agent ID (`/a/<agentId>`), so deploying an agent costs a DB row rather than a new codebase.
- **Accepted cost, stated plainly**: no mic, no camera. On-glasses interaction is display-out, D-pad-in. An agent here is a short sequence of choice screens driven by a cloud backend — not a voice assistant. This constrains the product definition and should be settled before building a builder.
- **Confusion resolved by disposition**: Keep — Web Apps, React Native/Expo (console *only*), Chrome simulator, Wearables MCP. Shelve — all DAT modules, `DisplayAccess`, mock device, iOS Simulator/Android Studio. Cut — MediaPipe and on-device Gemma (a web app can't reach phone-native inference), and CocoaPods (the "translates RN to native" claim was simply wrong).
- **New blocking unknown surfaced**: Meta's docs list Web App text input as unsupported, but the bundled `add-text-input` skill claims HTML inputs open an on-glasses composer with handwriting + voice. Contradiction must be resolved on hardware — it determines whether agents can take free-form input.
- Also captured: display dims at 20s / sleeps at 25s; black pixels are fully transparent on the additive waveguide.
- Output: a published architecture reference with three diagrams (the two-path fork, the four-layer system, the runtime request loop) — link in [[Projects/MetaGlasses/Reference|Reference]].

## 2026-08-25 — Rev 2: Ved reverses the iOS decision; voice is core
Ved pushed back on shelving iOS: *"end of the day we want the user to interact with the agent through the glasses as well, such as speaking to the glasses."* He was right, and the correction produced a better answer than the original.

- **Requirement clarified**: the wearer talks to their agent through the glasses. Voice is the point, not a later phase. This rules out Web Apps as the primary experience outright (no microphone).
- **Key discovery, from re-reading Meta's audio docs**: the voice loop uses **zero DAT APIs**. Mic in is `AVAudioSession` with `.allowBluetoothHFP` + an `AVAudioEngine` tap; audio out is `AVSpeechSynthesizer` over A2DP. Mic permission is a standard iOS dialog, not a DAT permission. Docs: *"Use mobile platform functions to access the device over Bluetooth."* For audio the glasses are just a Bluetooth headset.
- **Consequence**: a plain iOS app with **no DAT SDK linked** can do the full voice loop → no `ExternalAccessory` → **App Store publishable**. This is a third route that Rev 1 missed entirely, and it dissolves the distribution-vs-capability tradeoff for voice specifically.
- **Structural recommendation**: linking the DAT SDK is what breaks publishability — so isolate all DAT imports behind one module and one build flag. One codebase → a public voice-only build and a pilot voice+lens build. Flip the flag if Connect opens publishing.
- **Constraint surfaced**: HFP and A2DP are mutually exclusive — while the mic is open, output drops to 8 kHz mono. Either stay in HFP (phone quality, instant turn-taking) or toggle (hi-fi, ~2s settle per switch). Decides the conversational feel; worth settling before writing code.
- **Dispositions revised**: DAT modules promoted shelved → Phase 2; Web Apps demoted core → optional secondary client; MediaPipe/Gemma reopened (the reason for cutting them was web-app-specific); CocoaPods still cut.
- **Sequencing**: backend first (identical across all three routes, unblocked by every hardware question), then a phone-mic-only voice app, then point it at the glasses to verify the no-DAT claim, then turn-taking polish, then the lens behind the flag, console last.
- Artifact updated in place to Rev 2 with three new diagrams (three routes, the build-flag split, the voice loop).

**Lesson worth keeping**: Rev 1 optimized hard for distribution and reached a defensible but wrong conclusion, because it took "what an agent is" as undefined rather than asking. Pinning the core interaction first would have found the third route immediately.

## 2026-08-26 — Adipat's codebase surfaces; the interaction gets settled
Ved hit a `git clone` failure on `Adipat2003/MetaGlassTest` and, once it was cloned, asked for a full read-through. Two outcomes: the project turns out to already have a real codebase, and the product definition finally got pinned.

### The clone failure (two stacked causes)
- GitHub removed Git password authentication in Aug 2021 — the account password can never work.
- More importantly, the repo is **private**, which is *why* it prompted at all; public repos clone with no credentials. Unauthenticated API probe returned 404 (GitHub hides private repos rather than 403-ing).
- Fix: Ved's existing SSH key (`~/.ssh/id_ed25519`, registered as `VedEti123`) already had access. `git clone git@github.com:Adipat2003/MetaGlassTest.git` worked immediately. Global fix available via `url."git@github.com:".insteadOf`.

### The codebase
Bare React Native 0.87 app with two hand-written Swift bridges: `MetaWearables` (DAT 0.9.0 — Core + Display + MockDeviceKit, full session lifecycle, `display.send(FlexBox { Text })`) and `OfflineLlm` (MediaPipe running Gemma 2 2B int8 on-device). Mock-first, green CI, dev-setup docs, `AGENTS.md`. Fully documented in [[Projects/MetaGlasses/Repository|Repository]].

**Three corrections to the Rev 2 architecture:**
1. *"DAT has no React Native support"* — Adipat wrote a clean working bridge. The real concern was bridge-maintenance cost, not impossibility, and that cost is now already paid.
2. *CocoaPods "cut"* — the claim about it (translating RN to Swift/Java) was false, but the tool legitimately belongs here for RN's iOS deps. The explanation was wrong, not the tool.
3. *MediaPipe/Gemma "not for v1"* — it exists, so there's evidence to evaluate rather than speculation.

**The core gap**: zero audio code anywhere — no AVFoundation, no `NSMicrophoneUsageDescription`. The repo builds the *display* half (which Rev 2 said to defer) and skips the *voice* half (which Rev 2 said to ship). Not wrong, just solving a different problem — and the voice half needs no DAT, so it slots in alongside rather than into what exists.

### The interaction, settled
Ved: *"I think there is a spot of the display in the final app, I think we realistically only need voice input as well as speech output that outlines steps that we need to take."*

**This closes the "what is an agent" question open since the 2026-08-24 pivot.** An agent is: you speak a request, it talks you through what to do step by step, with the lens optionally mirroring the current step.

Good fit for the hardware — audio is transient and hands-free but unre-readable; the display persists but can't be spoken to. Speaking the step while showing it makes the lens a *reinforcement surface for the current step* rather than a UI to navigate, which also neutralizes the D-pad limitation entirely. Priority explicit: **voice is the product, display supports it.**

### Consequences
- Todo rebuilt as **Rev 3**, sequenced against the existing repo instead of greenfield: agent runtime returning structured steps → audio native module (mirroring the existing bridge pattern) → wire the loop → point audio at the glasses → attach display to step state (small, because Adipat already built it) → turn-taking → build-flag split → console.
- New issue list for Adipat: possible non-display-capable mock device (`.rayBanMeta` + `displayShowText`), App-Store-blocking `external-accessory`, empty location purpose string, and whether on-device Gemma survives contact with the voice loop.

## 2026-08-28 — Team task breakdown
Ved added a new round of notes to the inbox: the build is now split three ways. Ved owns the Phone App (voice loop, STT/TTS, on-phone chat context), Milos owns the Web app (displays backend LLM output, connects to the phone app over Bluetooth), Adi (Adipat) owns the Backend (cloud LLM calls, relays to the web app, receives glasses sensor data). Full breakdown moved to [[Projects/MetaGlasses/Todo|Todo]].

Two things flagged rather than resolved: the phone-stores-context task sits in tension with the existing "backend owns step state" design note, and Milos's phone↔web-app Bluetooth connection looks like a real attempt at the previously-open "can voice app and Web App run simultaneously" question — worth linking the two once there's an answer.

Also captured: camera-feed latency flagged as a future concern (not urgent, camera is DAT-gated); audio-via-phone-Bluetooth reconfirmed as "not part of Meta DAT yet" (phrasing suggests this could change); a reference link to Meta's DAT toolkit capabilities overview was added to [[Projects/MetaGlasses/Reference|Reference]].

## 2026-08-28 — Design questions answered; API contract drafted
Ved answered the drilling questions raised against the team task breakdown. Five decisions settled, and a first API contract drafted for Adi to own (link in [[Projects/MetaGlasses/Reference|Reference]]).

**Confirmed by Ved:**
- Audio must go phone↔glasses over Bluetooth — the web app can't handle it. (Matches the finding that Web Apps have no Bluetooth in their capability list, which also means **Milos's "connect to phone app (Bluetooth)" task as written isn't buildable** — the two clients must coordinate through the backend instead.)
- Adi owns the API contract; none existed yet, so a draft was produced rather than leaving a blank page.
- **Phone sends the full conversation** and stores all chats locally — explicitly to avoid cloud storage cost. Backend holds no history.
- **Lens shows only the most recent response.** Full history lives on the phone. UI deliberately simple.
- **STT and TTS on the phone.**

**Biggest consequence**: with on-device STT/TTS and a stateless backend, there is no audio pipeline to orchestrate — **the LiveKit vs Pipecat decision is moot**, and the backend collapses to a thin wrapper around one Claude call. That unblocks both other workstreams almost immediately.

**Gap the task list didn't cover — pairing.** The phone app and web app are separate clients and the backend must know they belong together, but Web Apps have no text input so the user can't type a code. Proposal: the phone mints `https://<host>/w/<pairingToken>`, the user adds that URL once, and the web app reads the token from its own path. The backend then keeps one tiny record per token (latest response + state) — the only state it holds, which preserves the no-storage-cost decision.

**Two refinements raised against Ved's answer to "prose or steps":**
1. *One text field isn't enough.* The lens is 600×600 with no scrolling at 20–24px type — about one sentence. A full spoken answer overflows with no way to read the rest. Proposed `speak` + `display` in a single structured-output call, which is also the cheap answer to Ved's own "maybe summarize before speaking" note (a second summarization call would add a round trip to an already tight latency budget).
2. *The agent will hear itself.* Speakers and mics sit centimetres apart; with HFP open during TTS the mic transcribes the agent's own voice. Fix is `setVoiceProcessingEnabled(true)` on the `AVAudioEngine` input node. The alternative — gating the mic while speaking — kills barge-in, so whether barge-in is in v1 scope is now a real decision for Ved's module.

**Also flagged, not urgent**: full transcript per turn + forever-stored chats grows unbounded and is paid for every turn. Trimming should happen on the phone, not silently server-side.

**Still unowned**: nobody has the agent itself — the prompt, what a "step" is, how a user would author one. Every current task is plumbing.

## 2026-09-02 → 09-04 — Hands-free capture works; the endpointer took four passes
Build sessions, not a meeting. The wake word, earcons and note capture landed in [[Projects/MetaGlasses/Glance|Glance]], and Ved confirmed the flow works end to end on his phone.

**What shipped:** "Hey Glance" / "Hey Jerry" opens a chat, "Hey Glance, start a note" opens Notes and starts recording, each with its own earcon. Toggle in Settings.

**The part worth remembering** is how long it took to end a turn automatically. Four separate bugs, each only visible once the one in front of it was fixed, all presenting as the *same* symptom — "the note never stops unless I press the button." Detail in [[Projects/MetaGlasses/Glance|Glance]]; the durable lesson is in [[Projects/MetaGlasses/Reference|Reference]]:

> **A transcript that keeps changing does not mean someone is still talking.** Recognition rewrites what it already has long after speech stops. Endpointing has to read microphone level, not transcript churn.

Three of the four were **race conditions between the wake loop and the capture it had just started** — a wake session seizing the mic during the capture's 1.1s Bluetooth settle window, a shared transcript buffer, and a wake loop that died permanently the first time a capture preempted it. Worth noting for the road ahead: **the settle window is a real structural hazard**, not a one-off. Anything that guards on "is a capture running" is blind during it.

**Process note.** Twice during this stretch a fix was described as "very likely the real cause" and was not. The pattern that actually worked was Ved's own observation — *manual stop works, automatic stop doesn't* — which ruled out `endAudio()` immediately, since both paths call it. Symptom asymmetry beat code reading.

**Also settled:** `DEVELOPMENT_TEAM` is now committed, reversing the 2026-08-30 stripping policy. Prompted by Ved asking the right question — *is pushing this actually harmful?* It is not; a Team ID is public, and the stripping had been cargo-culted from an early instruction into a permanent phantom diff.

**Named as next:** integrating with Adi's backend, and real context management for local chats and notes. Both now in [[Projects/MetaGlasses/Todo|Todo]]. The second carries a question worth putting to the team early — **with BYO-key sending chat straight from the phone to the providers, what is the backend still for?** The honest answer is the pairing/lens state, keyless users, and the agent runtime. Adi may still be building to the older "backend proxies the LLM" assumption.

## 2026-09-04 — Adi's backend arrives; fifteen discrepancies mapped
Not a meeting. Adi dropped `metaglasses-backend/` into the working directory; this was a read of it against [[Projects/MetaGlasses/Glance|Glance]] as built. **Nothing was integrated and no code was changed** — deliberately, at Ved's instruction, so the disagreements get settled before either side moves.

Full plan, every item numbered so it can be cited in a message to Adi (D1–D15 discrepancies, A1–A6 asks):
https://claude.ai/code/artifact/a58cd0ce-75ab-41a4-9d61-9631bf2e4b56

**Adi built to the contract, closely.** FastAPI: `POST /v1/chat`, `GET /v1/display`, `POST /v1/state`, `/healthz`. No audio anywhere, no history stored, the single `text` field as decided, pairing token as the lens capability. More than that, the shapes match Glance's existing types *exactly* — `{role, content}` in, `{responseId, text, createdAt}` out — and the state vocabulary is identical on both sides (`idle | listening | thinking | speaking`), which was luck as much as design since neither was written against the other.

**The disagreements fall in three groups.** *Identity*: Glance signs in against `/v1/auth/login` and `/v1/auth/signup`, which it invented and which do not exist — Adi chose **Supabase Auth**, so the phone authenticates directly and presents a JWKS-verified token. That closes the "add auth to the contract" item, but by moving it out of the contract entirely. *Pairing*: the backend requires a 32-hex pairing token on every phone request and Glance has no such concept anywhere in `src/`. *The voice turn*: no request deadline, no byte budget against the 256 KB cap, two unmapped error codes (403, 422), and the agent's own instructions dropped on the way to the server.

**The one that matters most is D7.** Glance agents carry custom `instructions`; `sendChat()` sends only the transcript, so the backend applies its own hardcoded system prompt. The same agent therefore answers differently depending on whether an API key happens to be saved, and every custom agent silently stops being custom the moment it falls back. On a platform whose premise is custom agents that is a correctness bug, not a nicety. Fix is one optional `system` field — additive, breaks no caller.

**Three smaller traps worth remembering:**
- **The model id is unverified.** The backend defaults to `gpt-5.6-sol`, which does not appear in Glance's own model list. If it does not resolve on Adi's key, *every* backend-tier call returns an unexplained `503`. Curl it before building anything on top — that is the whole reason Phase 0 exists.
- **The pairing token has to be permanent.** It is embedded in the Web App URL the user adds by hand, and Web Apps have no text input, so rotating it means re-adding that URL. Adi's `401` copy says "Re-pair on the phone," which reads as *mint a new one*. The record is ephemeral; the token must not be.
- **Both sides write lens state.** `/v1/chat` flips to `speaking` when it saves the text — but the phone has not started synthesising yet, and on a provider error the server writes `idle` over an error the phone is about to speak. The phone is the only party that knows when audio actually starts, so it should win.

**Two silent reversals of recorded decisions**, neither wrong, both worth having said out loud. The LLM slot in [[Projects/MetaGlasses/Reference|Reference]]'s stack table is `claude-opus-5`; the backend calls OpenAI. Adi's key, Adi's call — but the stack note now describes something nobody is building. And **Rev 3 step 1 is not closed by this.** It asked for an ordered list of steps with a current-step cursor held per session; what exists returns one sentence and holds no step state. That follows correctly from the 2026-08-28 single-`text` decision, which made brevity the system prompt's job — so the *runtime* in the Rev 3 sense still does not exist, and nothing currently needs it to.

**Answered by reading the code**: what the backend is for, at least as built — pairing and lens state, keyless users, and one OpenAI call. Not a proxy for chat, which BYO-key already took away from it.

## 2026-09-04 (continued) — the plan gets built, verified on real hardware, and pushed
Same day. Ved said to start reconciling rather than stop at the plan. Branch `integration` created in both repos; seven commits landed (one backend, six Glance), pushed with no PR opened. Full commit-by-commit writeup, built for sharing with Adi and Milos:
https://claude.ai/code/artifact/25195d44-06b9-44de-9466-4e38ab2772fe

**Backend** (`e5fe88c`): the two safe, additive fixes from the plan's A-list — an optional `system` field on `ChatRequest` so an agent's own instructions survive a fallback to the backend tier, and a reworded `/v1/display` 401 that no longer implies the pairing token itself should be rotated. The debatable design changes (A2, A3, A5, A6) were deliberately left as asks for Adi rather than decided unilaterally in his repo.

**Glance**, Phases 0–3 of the plan, in order:
- **Phase 0.** Docker was down; the user started it, then a full local Supabase CLI stack came up (following the backend's own `docs/supabase.md` workflow) rather than settling for `AUTH_MODE=disabled`. A live test user's Supabase JWT was confirmed round-tripping through the backend's real JWKS check across `/v1/state` (204), the ownership-conflict path (403), and `/v1/chat` (503 with no OpenAI key — proving auth passed before the model call failed, not a false negative). ⚠️ **D13 still not settled**: no OpenAI key was available, so `gpt-5.6-sol` resolving remains unconfirmed.
- **Phase 2 — the auth rewrite** (`f119239`, `30cc5a3`). `@supabase/supabase-js` in, `AuthContext` rebuilt around it with its public surface untouched, session in the Keychain via a custom storage adapter, `ApiConfig.token` replaced by an async `getToken()` re-resolved per request.
- **Phase 3 — pairing and state** (`139284b`). The 32-hex token, permanent by design; `/v1/state` posted fire-and-forget on every voice transition; the same call site also finally threads the agent's own instructions to the backend tier, closing D7 client-side to match the backend fix.

**Then run on a physical iPhone**, not just the simulator — the first time the reconciled app has run on real iOS hardware. Two device-specific snags, both fixed: `127.0.0.1` only resolves on the Mac itself, so `dev` was pointed at the Mac's mDNS hostname instead of a raw LAN IP (`a0d55a3`); and the phone was critically low on storage (under 1MB short) for the install, which needed the user to free space by hand before the build would land. Confirmed no DAT/Bluetooth claim was tested here — only the phone's own mic, a separate still-open item.

**Then a real request arrived mid-test.** The user hit "no model provider" on a plain new chat — correctly guessed it was the missing OpenAI key, but the fuller answer surfaced something worse: **no chat had ever been reachable via BYO-key**, because `createChat()` never attached an agent and the fallback check silently required one. Ved then asked for Microsoft Foundry support specifically. Foundry's endpoint shape was **looked up against Microsoft's own current REST reference rather than assumed** — an initial framing offered to Ved turned out to not match live docs (updated as recently as 2026-07-09), and the actual current wire format is simpler than what was guessed: `POST {endpoint}/openai/v1/chat/completions`, `api-key` header, no `api-version` needed. Landed as a fourth BYO-key provider (`8971ac4`), plus the agent-picker fix underneath it (`a898fd3`) — without which Foundry, or any personal key, would still have gone nowhere. **Confirmed working end to end by Ved against his own Foundry deployment.**

**Standing takeaway**: the "three-tier fallback already works, pointing at a real server is a Settings change" claim from `Glance.md`'s original design notes was wrong on two counts discovered only by actually running the thing — the auth swap was a real rewrite, and the fallback itself had been silently dead since 2026-08-30. Worth remembering next time a claim like that goes unexercised for two weeks.

## 2026-09-17/18 — the backend moved 22 commits, Glance caught up, and a real latency bug turned up
Ved repulled `metaglasses-backend` after some time away. Diff against the `integration` branch point: model provider **OpenAI → NVIDIA** (`moonshotai/kimi-k3`), a real **hosted trial + prod Supabase deployment** on Edge Functions (not just local FastAPI), image upload/attach support (`POST /v1/images`, `imageIds` on a message), a Postgres-backed pairing store, and the `system` field dropped from `ChatRequest` entirely.

**Glance updated to match, scoped explicitly as "frontend only, don't touch the backend":**
- `environments.ts` now points at the real hosted trial endpoint (`uitdzmwfqtsohgffhuom.supabase.co`) instead of Ved's Mac. Ved supplied the trial publishable key over chat — caught and flagged one real near-miss first: he initially pasted the **secret** key (`sb_secret_...`, full admin access) by mistake, which would have shipped inside the app bundle if written to this file; refused, explained why, asked him to rotate it and get the publishable one instead.
- The dead `system`-forwarding-to-backend code removed from `client.ts`/`ChatScreen.tsx` — backend silently ignored it anyway now. BYO-key is unaffected (goes straight to the provider).
- Built and ran on Ved's physical iPhone again. Hit and fixed, in order: curly vs. straight apostrophe in `--device` name (used `--udid` instead), missing provisioning profile (`--extra-params "-allowProvisioningUpdates"`), and an untrusted dev certificate (Settings → trust, on-device).
- **Root cause of "sign in works, chat doesn't":** not a bug in anything just built — a stale free-text Backend URL override, saved in `AsyncStorage` from earlier local testing, silently beat the new default forever. Fix: removed the free-text override entirely and replaced it with a **Dev/Prod toggle** in Settings — each side is a genuinely separate Supabase project/client instance, so switching signs the user out by design.
- **Root cause of "leaves the screen while thinking, comes back to nothing":** `ChatScreen`'s turn state (`voiceState`, the eventual reply) lived only in that screen's component state. React Navigation unmounts on back, so a fresh mount always showed a false "idle" and the landed reply/error had nowhere to render. Fixed by moving the whole turn — the async state machine, not just a flag — into a chat-keyed shared module, `src/chat/activeTurn.ts`, independent of screen mount lifecycle.

⚠️ **New finding, backend-side, not fixable from Glance: NVIDIA chat is slow enough to fail outright.** Timed directly against the live trial endpoint (not guessed) — `"Say hi"` took **93 seconds**; an ordinary question exceeded Supabase Edge Functions' **hard 150-second platform idle timeout** and was killed (`504 IDLE_TIMEOUT`) before NVIDIA ever replied. That ceiling isn't app-configurable — the backend's documented "5-minute" NVIDIA timeout never gets a chance to matter, Supabase cuts the connection first. The backend already built the right shape of fix for image responses (`NVCF-REQID` polling) but hasn't applied it to text chat. Write-up with both timed runs, the chart, and a runnable repro, for Adi: https://claude.ai/code/artifact/5a842f57-e65a-4d98-946d-b971430af441

**Then the glasses-photo-attach feature got scoped, which reopened the context-storage question.** Ved's instinct was that moving conversation history to the backend would make stitching a future glasses photo into a chat turn easier. Traced why that's not actually true: images are backend-stored by necessity either way (a vision model has to fetch bytes from somewhere), referenced by `imageId`, with the phone still sending the full transcript every request exactly as today — the "stitching" is one new field per turn, not an architecture change. Confirmed the backend's image storage is explicitly **temporary** (a `cleanup-pairing-images` function, 10-minute signed URLs), which if anything argues for keeping the phone as the durable copy, not less. **Decision: context storage stays phone-only**, reaffirmed rather than reversed. Audited `MetaGlassTest` to find what the feature actually needs on the DAT side: camera permission is wired, camera capture is not — no stream, no `capturePhoto`, no JS bridge, no `NSCameraUsageDescription`. Full scoping split by repo in [[Projects/MetaGlasses/Todo|Todo]] and [[Projects/MetaGlasses/Repository|Repository]].

**Built the phone-native version same day.** Rather than leave the photo feature as plumbing nobody could call, built the buildable-now half: a camera/photo-library picker (`media/imageSource.ts`, new — `takePhoto()`/`pickImageFromLibrary()`, both resolving to one shared `PickedImage` shape so a future glasses source is a third function here and nothing else), `uploadImage()`/`deleteImage()` against the backend's raw-bytes contract, a `message_images` schema migration, and an attach button + thumbnail preview in `ChatScreen`. **Confirmed working end to end**: attached a photo of a Lego set, the backend (NVIDIA) correctly identified it in the reply.

**Then asked to focus on unit tests and full coverage.** Scoped it honestly rather than chase 100% everywhere — native-bridge code (`Audio`, `MetaWearables`) can only be tested at the JS boundary, and screens have diminishing returns past behavioral smoke tests. Started with the modules that actually carry business logic: `db/` (migrations, every CRUD path, the image-attach schema — tested against a **real embedded SQLite** via a new `better-sqlite3`-backed test double for `op-sqlite`, not a stub that always returns `[]`), `api/client.ts` (every endpoint, error mapping, the 401 retry dance, stub fallback), and `chat/activeTurn.ts` (the turn state machine just extracted — typed vs. spoken input, BYO-key vs. backend routing, the busy guard, failure handling). Coverage on those went from 0-4% to 98-100%.

Ved then asked directly: **"are all connections to the backend tested?"** Answer was honestly no — closed the real gap: `pairing.ts` (100%, the token every request authenticates with), `auth/supabaseClient.ts` (100%, builds the Supabase connection per environment), `auth/AuthContext.tsx` (81%, token supply + 401 refresh + the Dev/Prod switch). Explicitly *not* part of that gap: `api/providers.ts`, the four BYO-key vendors called directly by design, bypassing the backend entirely.

**Two real bugs, both in the tests, not the app** — worth remembering for the next test-writing session: `.find()` grabbing the wrong event when a type fires twice per turn (every turn clears its notice/error banner first, then may set a real one — needs a last-match helper, not `.find()`); `jest.clearAllMocks()` only clears call history, not a `mockResolvedValue` a previous test configured, so tests further down a file silently inherited stale mock behavior from earlier ones and looked like they hung — `jest.resetAllMocks()` plus one `beforeEach` as the single source of truth per file fixed it. Also tried `@testing-library/react-native` for the `AuthContext` hook tests; `result.current` came back `undefined` on this RN 0.87/React 19 pairing, abandoned in favor of the hand-rolled `react-test-renderer` + `act()` pattern already proven in `App.test.tsx`.

**Coverage: 11% → ~39% overall**, 114 tests, all passing. Committed as three focused commits on `integration` — environment toggle (`8ad69b7`), photo attach + turn-state fix (`1c750d1`), test suite (`0f69cea`). Ved pushed and merged the PR into `main` himself.

## 2026-09-19 — DAT camera capture, built and confirmed working the same week it was scoped
Ved: "let's focus on building in the MetaGlassTest repository first. I want to send a request to take a picture on the glasses." Went straight to it rather than waiting on Adi — the gap audited 2026-09-18 turned out fully buildable in Ved's own repo.

**Linked `MWDATCamera`, which the Xcode project never had.** No Xcode GUI available in this environment, so added it by hand-editing `project.pbxproj` — four coordinated edits (`PBXBuildFile`, `PBXFrameworksBuildPhase`, `packageProductDependencies`, `XCSwiftPackageProductDependency`) mirroring the existing `MWDATCore`/`MWDATDisplay` entries exactly. Verified for real via `xcodebuild`, not just "looks right" — this caught a genuine compiler error immediately: `Stream` collides with Foundation's own `Stream` (the Swift name for `NSStream`); fixed by fully qualifying `MWDATCamera.Stream`.

**Built `capturePhoto()`, then broke it twice against real DAT/Mock-Device-Kit behavior, each time diagnosed from an actual error string, not guessed:**
1. **"A capability of this type is already active on the session"** on the second capture. The original design started a fresh camera stream per photo and tore it down (`stream.stop()`) afterward — but DAT allows only one camera *capability* per session, and stopping the stream doesn't release that registration. Fixed by setting the stream up once (first call) and reusing it for every capture after, tearing down only on `disconnect()`.
2. **"The camera stream stopped before a photo was captured"** — Mock Device Kit had no feed configured. `camera.setCapturedImage(fileURL:)` only sets what a capture *returns once already streaming*; it doesn't give the stream anything to stream, so without also calling `camera.setCameraFeed(cameraFacing: .front)` the stream went straight to `.stopped`. Found the exact argument label (`cameraFacing:`, not `facing:`) from a real compiler error — the docs never show the full signature.

**Also found and fixed a recurring UX trap during live debugging**: the test-harness app had one shared `error` state rendered once at the very bottom of the whole scrollable page. Every failed attempt looked like "nothing happens" because the actual error was off-screen below several other cards. Gave the Camera card its own `cameraError` state rendered directly under its button — this alone resolved two rounds of "nothing happens" that turned out to be real errors nobody could see.

**Confirmed working end to end, twice in a row**, via a screenshot: a generated placeholder image (no bundled asset — rendered at runtime with `UIGraphicsImageRenderer`, so nothing to add to the Xcode target) round-tripped through a real camera stream, Mock Device Kit standing in with the Mac's own camera as the feed.

**Live-debugging note for next time**: tried to drive the iOS Simulator directly (cliclick taps/drags + `xcrun simctl` screenshots) to self-verify without round-tripping through Ved. Drag gestures worked reliably; plain taps silently landed on the desktop behind the window despite window bounds checking out via AppleScript — never diagnosed why, abandoned after several attempts and handed verification back to Ved rather than keep burning turns on it.

## 2026-09-25 — Meta Connect 2026 findings: DAT 1.0 shipped, App Store path still stalled, and Meta announced a direct competitor
Ved asked for a read on Meta Connect 2026 (held 2026-09-23/24), prompted by news of "Muse," a newly-announced Meta agent. Researched via a forked subagent (web search across 6+ sources) rather than in the main thread, to keep the raw search output out of context.

**Muse is a real competitive threat, not a platform to build on.** It's Meta's own personal AI agent, made the Connect 2026 centerpiece: voice mode, video avatars, Mac computer-use, commerce integrations — and, directly overlapping this project, **it's coming to Meta's AI glasses for hands-free, wake-word activation with visual context, "act on what you're looking at."** That is the same voice+vision loop this project is building, shipping as Meta's own flagship glasses experience. There is a developer-facing side (Muse Spark 1.3 model API, a Connector Platform to plug a product into Muse's own task system) — but that's for *extending Muse*, not a toolkit for an independent competing agent. Confidence note: this is more consumer-press-sourced than independently verified — directionally right, not spec-precise.

**DAT 1.0 shipped, exiting year-long developer preview** — new capabilities include camera capture, voice-command streaming, and motion tracking, on both display and non-display glasses. **But the actual constraint blocking this project (App Store distribution) is unchanged**: Apple's MFi Product Plan authorization is still required, Meta's own site still only says a public submission path is "coming soon" with no date, and at least one outlet (VR.org) flagged that Connect's developer keynote was the obvious venue for a GA date and none appeared — calling a 2026 general-availability target "close to gone." Release-channels-only remains the reality, and the timeline looks like it's slipping rather than firming up.

**Web Apps gained one real capability**: **text input** now works on Ray-Ban Display web apps (dictation, handwriting, on-screen keyboard) — previously listed unsupported, this is a genuine change. Also new: wrist motion via Neural Band, GPS/device position, country expansion. **Camera and microphone remain unsupported for Web Apps** — nothing found contradicts the existing constraint that shaped the DAT-vs-Glance split in the first place.

**Other announcements, none changing this project's distribution math**: new hardware (Ray-Ban Meta Audio — camera-free, open-ear; Ray-Ban Meta Gen 3; Meta VR Glasses, Spring 2027 at $1,299.99, v207 SDK available now).

**Immediate consequence, still being talked through**: Ved is treating Muse as removing this project's edge and is thinking through a pivot — training/calling VLMs on how to perform specific tasks, then guiding a user through recreating that task via the lens + voice, rather than a general chat+vision assistant. Discussion in progress; not yet a decision.

Sources: meta.com/blog (Connect 2026 recap), developers.meta.com/blog (Connect 2026 dev recap), vr.org (wearables toolkit publishing piece), roadtovr.com (third-party SDK piece), therundown.ai (Muse feature rundown).
