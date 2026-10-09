# MetaGlasses

> **Pivot (2026-08-24)**: this project is no longer a cooking app. See Purpose below. The "Mise" recipe-app work is preserved in [[Projects/MetaGlasses/Meetings|Meetings]] and [[Projects/MetaGlasses/Reference|Reference]] as history/possible reusable groundwork, not the current direction.

## Purpose
A platform for building **custom AI agents on phone or web, then deploying them to run on Meta Ray-Ban Display glasses**. Instead of one purpose-built app (the earlier cooking concept), the goal is a general agent-creation-and-deployment layer — users define an agent, and it becomes something that shows up/works on the glasses.

## The interaction — settled 2026-08-26
**This closes the "what is an agent" question that had been open since the 2026-08-24 pivot.** Stated by Ved:

> **Voice input, plus speech output that outlines the steps you need to take.** Display has a real spot in the final app, but realistically voice in / speech out is what's needed.

So an agent is: **you speak a request, and it talks you through what to do, step by step** — with the lens optionally mirroring the current step.

That pairing suits the hardware unusually well. Audio is transient and hands-free but can't be re-read; the display persists (until it dims at 20s) but can't be spoken to. Speaking the step while showing it means the wearer hears it once and can glance back — the lens becomes a **reinforcement surface for the current step**, not a UI to navigate. That also sidesteps the biggest Web Apps limitation, since D-pad navigation stops mattering when nothing needs navigating.

**Priority is explicit**: voice is the product, display is the supporting surface. Build voice first; the display layer already exists in the codebase (see [[Projects/MetaGlasses/Repository|Repository]]) and can be attached to the step state once voice works.

## Status
**Architecture decided 2026-08-25 (Rev 2); interaction settled 2026-08-26.** Build path and product shape are now both defined. See "Architecture decision" below and the reference artifacts (links in [[Projects/MetaGlasses/Reference|Reference]]).

**As of 2026-09-04 the voice half works.** [[Projects/MetaGlasses/Glance|Glance]] (`VedEti123/Glance`) is the publishable app: wake word, on-device speech-to-text, spoken answers, hands-free note capture, local chat/note storage, bring-your-own API key. It runs on Ved's phone. **It links no DAT**, which is what keeps it App Store eligible.

That leaves **two codebases with opposite halves**: `Adipat2003/MetaGlassTest` ([[Projects/MetaGlasses/Repository|Repository]]) has the DAT display and on-device LLM bridges but no audio and cannot be published; Glance has the voice loop and deliberately no display. They are not being merged — the build-flag split (Rev 3 step 7) is how they eventually reconcile.

**As of 2026-09-04 there is also a backend, and it is now reconciled with Glance.** Adi's `metaglasses-backend/` is in the working directory: FastAPI, the three contracted routes plus health, Supabase Auth, one OpenAI call. Read against Glance the same day — fifteen discrepancies mapped first, because the app was built against endpoints nobody owned and the two would not have talked — then **built the same day**, on a branch called `integration` in both repos, pushed (no PR yet). Plan, then what actually shipped:
https://claude.ai/code/artifact/a58cd0ce-75ab-41a4-9d61-9631bf2e4b56
https://claude.ai/code/artifact/25195d44-06b9-44de-9466-4e38ab2772fe

**Verified against a real local Supabase project** (not mocks) and **run on a physical iPhone** (not just the simulator) — confirmed working end to end with a live Microsoft Foundry deployment, added the same day as a fourth BYO-key provider alongside Anthropic/OpenAI/Google. Testing it also surfaced and fixed a standing bug: a new chat never had an agent attached, so the BYO-key fallback had been silently unreachable since agents were added on 2026-08-30 — nobody had noticed because nobody had tried a personal key on a plain new chat before. Detail in [[Projects/MetaGlasses/Glance|Glance]].

**Current job**: the two things not yet closed. First, **someone with no key of their own still gets nothing** — the backend tier is fully wired but has no `OPENAI_API_KEY` configured, and the model id it defaults to (`gpt-5.6-sol`) has never been confirmed to resolve; both need Adi. Second, **real context management** for the chats and notes now accumulating locally — untouched by today's work. Both in [[Projects/MetaGlasses/Todo|Todo]].

**2026-09-17/18 — the backend moved a lot, and Glance caught up.** Adi pushed 22 commits past the `integration` branch point: the model provider switched **OpenAI → NVIDIA** (closing the "no key configured" blocker above, but see the new finding below), a real hosted **trial + prod Supabase deployment** exists (Edge Functions, not just local FastAPI), and image upload/attach support landed server-side. Glance was updated to match: `environments.ts` now points at the real hosted trial endpoint instead of Ved's Mac, the Settings screen's free-text backend-URL field was replaced with a **Dev/Prod toggle** (each a separate Supabase project — switching signs you out), and the dead `system`-forwarding-to-backend code was removed (the backend dropped that field; BYO-key is unaffected). Also fixed: chat turns used to live only in `ChatScreen`'s component state, so leaving the screen mid-reply and coming back showed a false "idle" and silently dropped the answer — turn state and results now live in a chat-keyed module (`src/chat/activeTurn.ts`) instead, independent of screen mount lifecycle.

⚠️ **New finding: NVIDIA chat replies are slow enough to fail outright.** Timed against the live trial endpoint — a trivial "Say hi" took 93 seconds; an ordinary question exceeded Supabase Edge Functions' **hard 150-second platform idle timeout** and was killed (`504 IDLE_TIMEOUT`) before any reply came back. That limit isn't app-configurable — the backend's own "5-minute" NVIDIA timeout setting never gets a chance to apply. The backend already has the right shape of fix built for image responses (`NVCF-REQID` polling) but not wired up for text chat yet. Full write-up, timings, and a runnable repro for Adi: https://claude.ai/code/artifact/5a842f57-e65a-4d98-946d-b971430af441

**Also 2026-09-18 — scoped the glasses-photo-attach feature, which reopened and reaffirmed the context-storage decision.** Conversation history stays phone-owned (see below); the backend's new image storage doesn't change that calculus since it's backend-side by necessity either way and is deliberately **temporary** (a cleanup job purges it, signed URLs expire in 10 minutes) — not designed to double as a photo archive. Audited `MetaGlassTest`: camera *permission* is wired, camera *capture* is not — no stream, no `capturePhoto`, no JS bridge, no `NSCameraUsageDescription` in `Info.plist`. Full scoping, split by repo, in [[Projects/MetaGlasses/Todo|Todo]].

**Built the phone-native half same day, then a real unit test suite.** Photo attach shipped against the phone's own camera/library rather than waiting on DAT — `takePhoto()`/`pickImageFromLibrary()` both resolve to one shared shape a future glasses source plugs into with no downstream changes. Then, prompted to focus on test coverage: moved from 11% to ~39% overall, concentrated entirely in the modules that carry business logic (db, the API client, the turn state machine, pairing, auth) rather than chasing the aggregate number — screens and the BYO-key providers are explicitly still uncovered. Full breakdown, including two real bugs caught in the tests themselves, in [[Projects/MetaGlasses/Todo|Todo]]. All of it — environment toggle, photo attach, turn-state fix, test suite — shipped as three commits (`8ad69b7`, `1c750d1`, `0f69cea`) on `integration`, PR'd and **merged to `main`** the same day.

**2026-09-19 — the glasses-side gap closed too: DAT camera capture built and confirmed working.** Turned out not to need Adi after all — built directly in `MetaGlassTest`, same week it was scoped. Two real DAT-lifecycle bugs found via actual compiler/runtime errors (not guesswork): DAT allows exactly one camera capability per session, so the original tear-down-per-photo design broke on a second capture; and Mock Device Kit needs a live feed (`setCameraFeed(cameraFacing:)`), not just a result image (`setCapturedImage`), or the stream never leaves `.stopped`. Confirmed end to end on the simulator, twice in a row, via Mock Device Kit using the Mac's own camera as the feed. Full detail in [[Projects/MetaGlasses/Todo|Todo]] and [[Projects/MetaGlasses/Repository|Repository]].

⚠️ **2026-09-25 — Meta Connect 2026: a direct competitor, and the App Store path is still stalled.** Meta's new **Muse** agent — the Connect 2026 centerpiece — is coming to Meta's own AI glasses for hands-free, wake-word, visual-context assistance ("act on what you're looking at"): the same voice+vision loop this project builds, now shipping as Meta's first-party flagship experience, not something to build on top of. **DAT 1.0 shipped** (camera capture, voice-command streaming, motion tracking, exiting preview) but the actual blocker — Apple's MFi Product Plan requirement for App Store distribution — is unchanged, with one outlet calling a 2026 GA timeline "close to gone." **Web Apps gained real text input support** (previously unsupported); camera/mic remain unsupported there. Full findings in [[Projects/MetaGlasses/Meetings|Meetings]]. Ved is now actively working through whether this changes the product direction — full pivot write-up (competitive landscape, prior-art analysis, two feasibility research threads) in [[Projects/MetaGlasses/Pivot|Pivot]]; tracked as an open question in [[Projects/MetaGlasses/Todo|Todo]].

⚠️ **What exists is not yet the agent runtime.** Rev 3 step 1 asked for an ordered list of steps with a per-session step cursor; the backend returns one sentence and holds no step state. That follows correctly from the single-`text` decision — brevity became the system prompt's job — but it means step 1 stays open, and nothing currently needs it closed.

⚠️ **Still unconfirmed on hardware**: the load-bearing claim that voice works with no DAT linked has been proven against AirPods (identical `.bluetoothHFP` path) but not against the glasses themselves.

The Mise cooking prototype is **archived** as of 2026-08-25 — moved to `/Users/vedeti123/Documents/MetaGlasses/archive/mise-cooking-app/` with an `ARCHIVED.md` explaining what's still worth borrowing (the five-input nav model, 600×600 no-scroll layout, additive-display typography) and what isn't (everything recipe-related). Reference only; not groundwork. The top level of the working directory is now clear for the new project.

## Architecture decision (2026-08-25, Rev 2) — voice-first on iOS, DAT behind a build flag

> **Rev 1 said "Web Apps, shelve iOS." That was wrong.** Ved pushed back: the wearer must be able to *speak to the agent through the glasses*, which Web Apps cannot do (no microphone). Rev 2 corrects it. Rev 1's reasoning about distribution was sound — it just optimized for the wrong constraint, and missed a third route.

### The finding that changed the plan
**The voice loop uses no DAT APIs at all.** Every audio example in Meta's docs is plain AVFoundation — `AVAudioSession` with `.allowBluetoothHFP` + an `AVAudioEngine` tap for mic in, `AVSpeechSynthesizer` over A2DP for speech out. Mic permission is a **standard iOS dialog**, not a DAT permission. The docs say: *"Use mobile platform functions to access the device over Bluetooth."*

For audio, the glasses are just a Bluetooth headset. **A plain iOS app with no DAT SDK linked can speak and listen through them** → no `ExternalAccessory` → no MFi conflict → **App Store publishable**. DAT is only needed for the lens display and camera.

*(Caveat: strong inference from the docs, not an explicit guarantee — Meta publishes this guidance inside the DAT docs. Verify empirically; it's the load-bearing claim.)*

### Three routes, not two
| Route | Wearer gets | Reach |
|---|---|---|
| Web App | Lens display, D-pad | Anyone with a URL |
| **Bluetooth audio app** | **Full voice loop** | **App Store** |
| DAT app | Voice + lens + camera | Invite-only channels |

### The structural decision that matters most
**Linking the DAT SDK is what makes an app unpublishable** — not Bluetooth, not the mic. So isolate every DAT import behind one module and one build flag. One codebase then yields two products: a voice-only App Store build for everyone, and a voice+lens build for pilot testers and demos. If Meta opens publishing (possibly announced at Connect, Sept 23–24), flip the flag — no rewrite.

### Audio constraint to design around now
**HFP and A2DP are mutually exclusive.** *"Activating HFP switches the glasses away from A2DP, and audio output quality drops to 8 kHz mono for the duration of the session."* Either stay in HFP (telephone quality both ways, instant turn-taking — right for conversation) or toggle to A2DP for hi-fi output (~2s route settle each way — one-shot Q&A only). The mic also beamforms onto the wearer, heavily suppressing the room.

### Layers
1. **Authoring** — agent builder console. Plain web first.
2. **Backend** — agent registry, runtime, ASR/LLM/TTS, tools + auth. **Identical for all three routes. Build this first** — it's unblocked by every open hardware question.
3. **Clients** — interchangeable heads on one API: web shell, voice iOS app, DAT app.
4. **Glasses** — pure I/O. No compute, no retained state.

### Tool dispositions (Rev 2)
- **Core**: AVFoundation (the voice loop), Swift/iOS (the client), Wearables MCP (`search_dat_docs`)
- **Phase 2**: `MWDATCore`, `MWDATDisplay`, `MWDATMockDevice`, `DisplayAccess` sample — *promoted from shelved*; needed for the lens, kept behind the build flag
- **Optional**: Web Apps + Chrome simulator — *demoted from core*; still the only public route to lens visuals, but can't do voice, so it's a secondary client
- **Later**: React Native / Expo — console only, and weaker now that the client is Swift regardless; a plain web console is faster
- **Reopened**: MediaPipe / Gemma — Rev 1 cut them because a web app can't reach phone-native inference; on a native client that reason evaporates. Not for v1 (cloud is faster to build and likely lower latency), but legitimate later for privacy/offline
- **Cut**: CocoaPods — unchanged; the "translates RN to native" claim is wrong, and DAT ships via Swift Package Manager

## Key research findings (2026-08-24)
- **Herald reclassified**: it's an app-store/platform for the glasses, not a competing product — its own "AI generation app" isn't built for complex tasks. This corrects the earlier competitive framing in [[Projects/MetaGlasses/Reference|Reference]], which treated Herald/Sous primarily as a cooking-app competitor. That competitive analysis is now moot given the pivot away from cooking, but the platform-mechanics facts about Herald (review process, no payment rails, etc.) likely still matter for distribution regardless of what gets built.
- Running LLMs locally on iPhone is reportedly inefficient (per Reddit-sourced research cited in the meeting); vision tasks need hardware acceleration even when run on-phone. This is informal secondhand research, not verified firsthand — worth confirming before it drives architecture decisions.
- Open question, explicitly flagged as a TODO in the meeting: whether there's a way to run agent compute on the user's phone and connect/stream results to the glasses display.

## Superseded: the DAT-first plan (2026-08-25 morning)
Earlier the same day, follow-up research had settled on a native iOS build using `MWDATCore` + `MWDATDisplay`, starting from Meta's `DisplayAccess` sample and testing against `MWDATMockDevice`. **That plan is now shelved**, not because the technical detail was wrong — it was accurate, and is preserved in [[Projects/MetaGlasses/Reference|Reference]] — but because it didn't account for distribution. Confirmed directly in Meta's docs: *"Publishing to the App Store is not currently supported... since the SDK currently uses the `ExternalAccessory` framework, it will lead to App Store rejection due to Apple's MFi program and privacy manifest requirements."* The alternative is invite-only release channels only. That kills DAT as a delivery path for a platform, though it remains correct if mic/camera ever become essential enough to accept the distribution ceiling.

## Team (2026-08-28)
Three people, three components: **Ved** — Phone App (voice loop, speech-to-text/text-to-speech, on-phone chat context); **Milos** — Web app (displays LLM output, calls the backend, connects to the phone app over Bluetooth); **Adi** (Adipat — see [[Projects/MetaGlasses/Repository|Repository]]) — Backend (cloud LLM calls, relays responses to the web app, receives sensor data from the glasses). Detailed task breakdown in [[Projects/MetaGlasses/Todo|Todo]].

Two things were flagged here as worth resolving explicitly rather than letting each person assume differently. **Both are now answered by the code as built (2026-09-04):**

1. ~~Whether conversation context lives on the phone, in the backend, or both~~ — **the phone owns all history.** The backend stores one record per pairing token holding only the latest response and state, expiring an hour after the last phone request. The old "backend owns current-step state" note applies to a step cursor within one answer, which does not exist yet and is not needed.
2. ~~Milos's "connect web app to phone app over Bluetooth"~~ — **not buildable as written, and now unnecessary.** The two clients coordinate through the backend: the phone mints the pairing token, the web app reads it from its own URL, and `GET /v1/display` is the only channel between them. The underlying question — *can a voice app and a Web App run simultaneously?* — is separate and still open in [[Projects/MetaGlasses/Todo|Todo]].

## Notable Notes
- **The App Store question is now answered, and it drove the whole decision.** The item flagged in earlier notes as "worth confirming" was confirmed on 2026-08-25 against Meta's own docs: still blocked. Distribution for DAT = invite-only release channels (testers need Meta accounts, invited by email; in Dev Mode only one third-party app can be registered at a time).
- **Text input is the one genuinely blocking unknown.** Meta's Web Apps capability table lists text input as *unsupported*, but the bundled `meta-wearables-webapp:add-text-input` skill claims standard HTML inputs open an on-glasses composer with handwriting + voice. These contradict. The answer determines whether an agent can take an open-ended prompt on-device or only menu selections — test on real hardware.
- Display behavior to design around: the lens **dims at 20s of inactivity and sleeps at 25s**, so a slow-thinking agent must keep the display fed. And because the display is additive, **black pixels are fully transparent** — dark grounds with bright type is a legibility requirement, not a style choice.
- The Herald platform-mechanics facts (review process, no payment rails) may still matter for distribution — see [[Projects/MetaGlasses/Reference|Reference]]. The cooking-era competitive analysis around Sous is fully moot.
- Meeting notes were informal/shorthand in places (e.g. "CocoaPods as a translator," "Reactive Native for HTML") — captured faithfully in [[Projects/MetaGlasses/Meetings|Meetings]]. The CocoaPods claim has now been checked and is wrong; it's iOS's dependency manager and has no role here.
