# MetaGlasses — Glance (repo: `VedEti123/Glance`)

Local path: `/Users/vedeti123/Documents/MetaGlasses/Glance/`

> Fully renamed from DispChat on 2026-08-30 — directory, Xcode project/workspace/scheme/target, iOS source folder, Android package, storage keys, database name and Keychain service. `git mv` throughout so history follows the files.
>
> **Local data was deliberately orphaned** rather than migrated: new DB file, new Keychain service, new AsyncStorage keys. Pre-launch with no real users, so a migration shim would have been permanent cruft. Cost: sign in again, test chats/notes gone, **API keys re-entered**.
>
> **Bundle ID: `vedeti123.glance`** (was the RN template default, which free provisioning rejects). Matches the convention used for MetaGlassTest.
>
> Two things a naive find-and-replace missed, worth remembering for next time: `android/app/build.gradle` carries the package in **both** `namespace` and `applicationId`, and `MainApplication.kt` keeps its own `package` line that moving the file does not update.

## Brand
Logo is a speech bubble with a voice waveform inside, on a near-black ground. Tagline: **"Talk to your AI, at a look."**

Colours **sampled from the logo PNGs pixel-by-pixel**, not approximated:

| Token | Value | |
|---|---|---|
| `ground` | `#151619` | exact logo backdrop |
| `accentAlt` | `#616FEA` | gradient start, periwinkle |
| `accent` | `#86E6CD` | gradient end, mint |
| `ink` | `#F4F4F0` | wordmark white, slightly warm |
| tagline | `#88EDCA` | |

- **Mint is the accent** — brightest against the dark ground, so it carries anything live or primary. Amber is gone entirely.
- **Periwinkle is the informational colour**, keeping status states on-brand rather than introducing a foreign hue.
- Eyebrow tracking widened to 2.2 to match the tagline's letter-spacing in the lockup.

**`LogoMark` / `Wordmark` are drawn in code**, not shipped as bitmaps — sharp at any size, recolour with the palette. Compromise: RN cannot stroke a gradient border, so the two endpoints are split across the bubble (periwinkle top/left, mint bottom/right), which reads as the same diagonal sweep at logo sizes. A true gradient would need `react-native-linear-gradient` with a masked border.

**App icon** generated from the supplied artwork with `sips` — tight crop, scaled to ~72% of a 1024 square, padded onto `#151619`. Single 1024 source in the asset catalog; Xcode derives the rest.

**Avoid colour emoji in UI chrome** — the settings gear rendered as Apple's 3D emoji and clashed badly. Fixed with `U+FE0E` (variation selector-15) to force text presentation so the glyph inherits the palette.

## Navigation — glass tab bar (2026-08-30)
Three top-level destinations: **Chats · Notes · Settings**, in a floating translucent bar.

- **Real blur, not fake translucency.** `@react-native-community/blur` wraps `UIVisualEffectView`, so the bar picks up and blurs whatever scrolls beneath it. That live refraction is what makes it read as glass rather than a grey panel — a semi-transparent colour does not. iOS 26 renders those system materials with the current design language automatically.
- ⚠️ **RN cannot call Apple's actual Liquid Glass API.** This is system materials + brand gradient, which is as close as RN gets without a custom native view.
- **The selected pill slides** (spring, stiffness 220 / damping 24, tuned to settle without bouncing — it moves on every tap). Sliding is what makes the bar read as one continuous surface rather than three buttons.
- A hairline of mint along the top edge mimics light catching the lip of a glass surface; a drop shadow keeps it floating rather than painted on.
- **The bar floats over content**, so every screen pads its scroll view by ~96pt to clear it.
- Structure: `RootStack { Tabs, Chat, Agents, AgentEditor, ApiKeys }` — the tabs are one stack entry, and detail screens push over them.
- Settings gear removed from the Chats header; it is a tab now.

## Notes (2026-08-30)
New `notes` table (migration 3): body, timestamps, `spoken` flag. List + capture screen.

Deliberately flat — no title, no folders, no tags. Notes get captured by voice while the wearer's hands are busy, and anything that needs filling in afterwards defeats the point. Tap the dot to record, or type.

## Wake word + hands-free capture (2026-09-02 → 09-04)
**"Hey Glance" / "Hey Jerry"** opens a chat; **"Hey Glance, start a note"** opens Notes and starts recording. Toggle in Settings.

- **Routing is deliberately lopsided** (`src/wake/intent.ts`): only an explicit note phrase opens a note, everything else becomes a chat. Mistaking a question for a note silently buries it in a list nobody is looking at; mistaking a note for a question just returns an answer that can be ignored. Matched as **whole phrases**, so "what does note mean in music" still opens a chat.
- **"Jerry" is an alias for "Glance"** — a plain two-syllable name recognisers already know, so it lands far more reliably than a coined word. Both match against a family of near-misses ("hey glans", "hey glants", "hey gerry"): missing a wake is a worse failure than occasionally waking on "hey plants".
- **Two earcons, synthesised in memory** rather than shipped as assets — note is brighter/higher (880 → 1318.5 Hz), chat warmer/lower (523.25 → 698.5 Hz). Both rise, because both mean "started", but they sit in different registers so they are told apart by ear alone, which is the whole point on a wearable. The cue plays **to completion before the recogniser opens**, so it never transcribes itself.
- ⚠️ **A cue must configure the audio session itself.** It was inaudible on Bluetooth because the route was only established inside `startListening`, which runs after. `.playAndRecord` without `.defaultToSpeaker` also routes to the earpiece.

## Endpointing — the fix that took four attempts (2026-09-04)
The turn would not end on its own; pressing stop always worked. That asymmetry was the whole clue — manual stop and the silence timer call the same `endAudio()`, so `endAudio` was never the suspect. Four independent bugs, each hidden behind the last:

1. **The countdown re-armed on every partial result.** `SFSpeechRecognizer` keeps emitting *changed* partials long after the speaker stops, refining what it already has. Re-arming on those resets the countdown forever.
2. **The wake loop seized the microphone during the capture's settle window.** `startWakeWord` refused while a dictation `task` existed — but during the 1.1s Bluetooth settle there is no task yet, so the guard passed. Both paths then **shared one transcript buffer**, and the orphaned wake session blanked what had already been dictated.
3. **Transcript churn is not a silence signal at all.** See [[Projects/MetaGlasses/Reference|Reference]] — endpointing now reads **microphone level** against an adaptive noise floor.
4. **`autoRecord` was a boolean checked against a ref set once.** Notes is a *tab*, so it never unmounts: the first wake recorded and every one after navigated to a screen that just sat there. Now a timestamp token compared against the last one handled.

Also fixed along the way: the wake loop **died permanently** the first time a capture took the mic, because preemption and the wearer switching wake off both resolved `cancelled: true` and the loop broke on either. Native now marks preemption separately.

**Guards that came out of it**, all in `AudioModule.swift`:
- a **session generation counter** — every recognition callback carries the value it was created under, so a superseded task can never write into the session that replaced it
- an empty result **can never blank a non-empty transcript** (cancelling a task is how every capture ends, and it can deliver one last empty result)
- a **55s max-capture backstop** — recognition tasks expire around a minute anyway, so ending cleanly beats surfacing that as an error
- teardown **resolves a capture caught mid-settle** instead of leaving the promise hung

Timings: 8s to start speaking, **1.5s of silence to end** (notes were 3.5s and are now the same as chat — the longer window sounded right for dictation but in practice just left the note open).

⚠️ **Untuned number**: the voice threshold is 3.5× the adaptive noise floor, chosen without measuring against real glasses. If notes start cutting off mid-sentence somewhere loud, that is the dial, not a new bug.

## Signing (2026-09-04) — reverses the earlier "strip the team" policy
`DEVELOPMENT_TEAM = 488KMWBRJ4` is now **committed**. A Team ID is a public identifier, not a credential — it ships inside the code signature of every released app, and signing additionally requires team membership plus a certificate and its private key. It had been stripped from every commit on an early instruction, not because of any risk found in it, which left a permanent phantom diff on the working tree.

Anyone on the same Apple Developer team can now clone and build with no signing setup. ⚠️ Only an **Organization** membership can have team members; an Individual membership is one person permanently. If a second developer outside the team needs to build, move the setting to a gitignored `ios/Local.xcconfig` read by a Podfile `post_install` hook — do **not** go back to stripping it by hand.

## Original notes (as DispChat)

The **publishable** app, started 2026-08-30 at `/Users/vedeti123/Documents/MetaGlasses/DispChat/`.
Distinct from [[Projects/MetaGlasses/Repository|MetaGlassTest]], which is the DAT/display prototype and is **not** App Store eligible.

## The rule that shapes everything
**No DAT SDK, ever, in this app.** Linking it pulls in `ExternalAccessory` and triggers MFi rejection. That means:
- ✅ Voice — mic in, speech out, over Bluetooth HFP (needs no DAT)
- ❌ **No lens display** — that is DAT-only. Lens visuals are the Web App's job (Milos), not this app's.
- ❌ No camera

## "Connecting to the glasses" without DAT
There is **no in-app pairing**, and this is not a limitation to work around — it is how it works. For audio the glasses are an ordinary Bluetooth peripheral: the user pairs in **iOS Settings → Bluetooth**, and iOS owns routing. The app can only *detect* what the OS offers and choose to capture through it.

So the Settings screen shows device status and a route toggle, not a connect flow. `getAvailableInputs()` was added to the audio module for exactly this — it lists paired Bluetooth inputs so the UI can name them.

## Stack
React Native 0.87.0 / React 19.2.3 (identical to MetaGlassTest, so the audio module ported unchanged), TypeScript, CocoaPods for iOS deps.

| Concern | Choice |
|---|---|
| Navigation | `@react-navigation/native-stack` v7 |
| Chat storage | **`@op-engineering/op-sqlite`** — a real `chats`/`messages` schema with migrations |
| Key-value | `@react-native-async-storage/async-storage` v3 |
| Voice | `AudioModule` ported from MetaGlassTest, plus `getAvailableInputs()` |
| Auth | **`@supabase/supabase-js`** (2026-09-04) — session in the Keychain via a custom storage adapter, not AsyncStorage. Needs `react-native-url-polyfill` (Hermes's `URL` is incomplete) and `react-native-get-random-values` (the pairing token needs a real CSPRNG), both loaded first in `index.js` |

## Structure
```
src/api/client.ts            backend client + local stub + pairing/state calls
src/api/providers.ts          BYO-key: Anthropic, OpenAI, Google, Microsoft Foundry
src/auth/AuthContext.tsx     session (Supabase), base URL
src/auth/supabaseClient.ts   lazy Supabase client construction
src/config/environments.ts   per-build API + Supabase config (dev/trial/prod)
src/pairing/pairing.ts       mints + persists the 32-hex lens pairing token
src/settings/SettingsContext.tsx   preferBluetooth toggle
src/db/database.ts           SQLite open + versioned migrations
src/db/chats.ts              chat/message repository, transcript builder
src/audio/                   ported audio module
src/screens/                 Login, ChatList, AgentPicker, Chat, Settings
src/components/ui.tsx        Button, Card, Field, Banner, StatusDot
src/theme/tokens.ts          palette, spacing, type scale
ios/DispChat/AudioModule.swift + .m
```

## Design decisions
- **The phone is the source of truth.** The JS layer owns the voice state machine; the native module holds no conversation state, and the backend holds nothing at all.
- **Chats never leave the device.** Kept on sign-out — they are the user's data and the server never had a copy. Matches the "no cloud storage cost" decision.
- **Transcript trimming happens on the phone** (`buildTranscript`, **now last 200 turns**, raised from 40 during the wake-word work). Deliberate: the backend is stateless and paid per turn, and server-side trimming would silently drop history the user can still see. ⚠️ A turn count is not a budget, and the real server caps at **256 KB** — see the integration section below.
- **Voice in → speech out; typed in → read answer.** Only speaks back when spoken to.
- **Backend stub when no URL is set.** The whole app is usable end to end before Adi's server exists. ⚠️ ~~Pointing at the real one is a Settings change, not a code change.~~ **False, established 2026-09-04** — that holds for the base URL and nothing else. See below.
- **Visual language**: near-black ground, amber reserved *only* for "live" (recording, active route, primary action), 52pt minimum touch targets — it gets used one-handed while walking.

## BYO-key model (2026-08-30) — supersedes "backend holds the keys"
Ved asked for users to add their own API keys in the phone UI. That reverses the 2026-08-28 decision that the backend holds provider credentials, and it changed the security requirements:

- **Keychain, not AsyncStorage.** `react-native-keychain` added; **both the auth token and API keys** now live there (`ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY`). An API key in AsyncStorage rides along in an unencrypted device backup, and it is the user's money.
- **Keys never reach our backend.** With a key saved, the app calls Anthropic / OpenAI / Google / **Microsoft Foundry** (added 2026-09-04) **directly** from the phone (`src/api/providers.ts`). Better privacy, and we are not custodians of other people's billable credentials. Foundry is structurally different from the other three — every resource has its own hostname, so it needs an endpoint alongside the key, stored via a separate `apiEndpointStore`.
- **Three-tier fallback**: user's own key → backend → local stub. Works at every stage of rollout. ⚠️ **Silently unreachable until 2026-09-04**: `hasOwnKey` only gets checked when a chat has an agent, and nothing let you choose one — see the agent picker fix above, and [[Projects/MetaGlasses/Todo|Todo]].
- Keys are checked against the provider's expected prefix on save, so a wrong-provider paste fails immediately instead of becoming an opaque 401 mid-conversation.

## Agents, authored on the phone (2026-08-30)
`agents` table (migration 2): name, instructions, provider, model. List + editor screens, two starter agents seeded on first launch.

- **`BASE_INSTRUCTIONS` is inherited by every agent** — one short step per turn, no markdown or lists. This is where the "answers must fit a non-scrolling 600×600 lens and be speakable" constraint is finally *enforced* rather than just documented. Shown read-only in the editor.
- **Deleting an agent keeps its chats** — `ON DELETE SET NULL`, not cascade. Losing history by tidying up an agent would be a nasty surprise.

## Onboarding (2026-08-30)
Five pages before sign-in: talk/one step back · works through your glasses · bring your own model · build your own agents · your chats stay here. Swipeable, skippable, shown once.

Written to set expectations rather than sell — the glasses page states outright that audio drops to call quality while the mic is open, "that is how Bluetooth voice works, not a fault." Better learned in onboarding than mistaken for a bug.

## Backend integration — shipped 2026-09-04
Adi's backend now exists (`metaglasses-backend/`, documented in [[Projects/MetaGlasses/Reference|Reference]]). Read against this app, mapped into fifteen discrepancies (D1–D15) and six asks for Adi (A1–A6), then **built the same day** on a branch called `integration` in both repos, pushed, not yet a PR. Plan, then commit-by-commit summary of what actually shipped:
https://claude.ai/code/artifact/a58cd0ce-75ab-41a4-9d61-9631bf2e4b56
https://claude.ai/code/artifact/25195d44-06b9-44de-9466-4e38ab2772fe

**Verified against a real local Supabase CLI stack** (not just mocks) — a live test user's Supabase JWT round-trips correctly through the backend's JWKS check. **Then installed and run on a physical iPhone** (not just the simulator) over the same Wi-Fi as the dev machine — the first time the reconciled app has run on real iOS hardware. Confirmed working end to end with a live Microsoft Foundry deployment through the BYO-key path (see below). This does **not** touch the separate, still-open "voice with no DAT SDK" hardware claim in [[Projects/MetaGlasses/Overview|Overview]] — no glasses or Bluetooth were involved, only the phone's own mic.

### Microsoft Foundry — fourth BYO-key provider
Added alongside Anthropic/OpenAI/Google. Verified against Microsoft's own current REST reference rather than assumed (the shape in general circulation is stale): `POST {endpoint}/openai/v1/chat/completions`, `api-key` header, standard OpenAI-shaped body, `api-version` optional and defaulting to `v1` — no dated preview string to keep in sync. Structurally different from the other three: every Foundry resource has its own hostname, so `Settings → API keys` now collects a resource endpoint alongside the key, and the agent editor takes a freeform deployment name instead of picking from a fixed model list.

### Agent picker — a standing bug this surfaced, not a Foundry-specific one
While testing Foundry, found that `createChat()` never attached an agent, and with no agent, `ChatScreen`'s BYO-key check was always false — **no saved key, from any provider, was ever reachable from an ordinary new chat.** This had been sitting silently since agents were added on 2026-08-30; nobody had noticed because nobody had tried a key on a plain new chat before. "New chat" now navigates to a picker (agent, or "no agent" — today's unchanged default) before opening; wake-word chats deliberately skip it.

**What already matches, with no work at all** — worth knowing so the list below doesn't read as worse than it is. `TranscriptMessage` is `{role, content}` and so is the server's `ConversationMessage`; `ChatResponse` is `{responseId, text, createdAt}` and so is the server's; `VoiceState` and the server's `PairingState` are the *same four strings*. None of that was coordinated — both were written from the same contract draft.

**The four changes that actually cost something — all shipped 2026-09-04:**

1. **✅ Auth is a rewrite, not a swap** (D1–D3, commits `f119239`, `30cc5a3`). `@supabase/supabase-js` added, sign-in goes directly to Supabase, `signIn`/`signUp`/`stubSignIn` deleted from `client.ts` — the password never reaches FastAPI. `AuthContext`'s public surface stayed byte-identical, **no screen needed touching**. supabase-js got a **Keychain storage adapter** built on the `put`/`read`/`drop` primitives in `secrets.ts` (puts the refresh token in the Keychain by construction; the orphaned `authTokenStore` was deleted). `ApiConfig.token` became an **async `getToken()`** resolved per request, plus one refresh-and-retry on a 401 — a Supabase access token lasts about an hour and this app sits backgrounded for days.
2. **✅ Pairing now exists** (D4, commit `139284b`). A 32-hex token, minted once, Keychain, sent on `/v1/chat` and `/v1/state`, **never rotated**. ⚠️ Still deferred: surfacing it in Settings as a URL + QR — the web app's own host isn't decided yet, so that's blocked on Milos, not on this.
3. **✅ `voiceState` now leaves the phone** (D6, commit `139284b`). `/v1/state` posted on every transition, **fire-and-forget** — a status update never adds latency to a turn the wearer is waiting on, and it silently re-registers the pairing after the server's 1hr TTL or a redeploy.
4. **Voice-turn failure handling — still open** (D8–D10). No request deadline yet: a hung `fetch` still leaves `voiceState` on `thinking` forever with the mic gated. Needs an `AbortController` (~12s) and a **spoken** error, a byte budget on `buildTranscript` (~120KB against the server's 256KB cap), and copy for `403`/`422`. The only one of the four not yet done.

**✅ The product bug this exposed (D7) is fixed on both sides.** `sendChat()` now takes an optional `system` string — `BASE_INSTRUCTIONS` + the agent's own `instructions`, sent on **both** the BYO-key and backend tiers (`139284b`), matched by the backend's own optional `system` field (`e5fe88c`). The same agent now answers consistently regardless of which tier handles the turn.

**Still unverified**: the backend defaults to `gpt-5.6-sol`, which is not in this app's own model list, and no OpenAI key has been available in testing to actually check it. If that id is wrong, every backend-tier call returns `503` with no explanation — this is why every backend-tier chat currently 503s regardless.

## Known gaps before shipping
- ✅ ~~Auth token in AsyncStorage~~ — **moved to Keychain 2026-08-30** along with API keys.
- ✅ ~~Bundle ID still the RN template default~~ — **`vedeti123.glance`, 2026-08-30**.
- ✅ ~~No signing team~~ — **committed 2026-09-04**, see Signing above.
- ✅ ~~The API contract has no auth endpoints yet~~ — **resolved 2026-09-04, but not the way this assumed.** `/v1/auth/login` and `/v1/auth/signup` were never going to exist: Adi chose Supabase Auth, so the phone signs in directly and presents a JWKS-verified token. `AuthContext` rewritten to match, session in the Keychain via Supabase's own storage adapter. See above.
- ✅ ~~No agent picker when starting a chat~~ — **shipped 2026-09-04.** Turned out to be more than a missing convenience: without it, `hasOwnKey` was always false for a new chat, so the BYO-key fallback was unreachable regardless of any key saved. See above.
- **Long notes truncate at ~60s** — `SFSpeechRecognitionTask` expires there. The 55s backstop ends them *cleanly* rather than fixing it; a real fix needs task restart plus transcript stitching.
- Onboarding still uses colour emoji (🎙 🕶 🔑), slightly off-brand against the drawn logo.

## API drift found while building (RN 0.87 era)
- **AsyncStorage v3 renamed the multi-key API**: `multiGet`/`multiSet`/`multiRemove` → `getMany`/`setMany`/`removeMany`, and they take/return **records**, not tuple arrays.
- **`StatusBar backgroundColor`** is Android-only and no longer in the iOS types.
