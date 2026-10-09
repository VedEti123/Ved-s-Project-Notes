# MetaGlasses — Todo

## ⚠️ Pivot in progress (2026-09-25/27) — B2B task-training assistant, real framing decisions now made
Triggered by Meta Connect 2026 — Meta's newly-announced **Muse** agent overlaps directly with this project's original voice+vision loop as Meta's own flagship first-party experience. See [[Projects/MetaGlasses/Meetings|Meetings]] for the full Connect 2026 findings.

**Direction, as of 2026-09-27**: a B2B task-training assistant — an expert records a task once, a trainee wearing the glasses gets correction as they attempt it. Not fully decided (breadth across "any task, any industry" is still unvalidated, and a first pilot vertical hasn't been picked), but real architecture/scope decisions have been made after three rounds of empirical testing across real datasets. Full detail — competitive landscape, prior-art analysis (`Trvise/VortexLens`), feasibility research, and all test results — lives in [[Projects/MetaGlasses/Pivot|Pivot]]. Summary of what's actually decided:

- **Verification method**: image-to-image comparison (live photo vs. the expert's reference photo) using a cloud VLM, with the model required to state its reasoning before its verdict — 13/13 correct on screws and hazelnuts (MVTec AD), plus 1 of 2 correct on welds from `Trvise/VortexLens`'s messier scraped dataset (the one miss was a false positive on two good welds from unrelated jobs). The strongest, most repeated result of anything tried.
- **Text-instruction conversion (video/reference → text) stays in the architecture**, despite underperforming image-to-image in every test so far — Ved's product-level bet that it pays off long-term, not a technical finding. Both approaches continue to be developed, not just the current leader.
- **No human-in-the-loop review of any kind** — explicitly rejected, including the confidence-threshold-to-human-reviewer pattern real competitors (RealWear/Scope AR/Taqtile) actually use. Maximum acceptable fallback under uncertainty: show the trainee the reference pictures on-screen to self-compare. This is a deliberate point of difference from the existing competitive set.
- **On-device inference deferred, not abandoned.** Viability testing continues on the existing cloud setup (Microsoft Foundry, `gpt-4.1-mini`) rather than standing up anything on-device yet.
- **Open next test**: does image-to-image comparison scale when the reference material is a full video (e.g., a YouTube video) rather than a single photo? Not yet tested — every round so far only compared single static frames.
- **Architecture & implementation plan (2026-09-27)**: `/Users/vedeti123/Documents/MetaGlasses/GlanceInstruct/PLAN.md` — human-readable overview plus a phased, task-by-task plan for coding agents (GI-0.1 … GI-5.2). Phase 0 is a validation gate; its task GI-0.3 is the video-reference test above. **Local first (decision D6, 2026-09-27):** everything runs on the Mac against a local Supabase stack in Docker until a full trainee session works end to end locally. Only the Foundry model call leaves the machine, and deploying a hosted backend is its own later task (GI-5.0). Nine open questions (Q1–Q9, Part 6) need Ved's call.

## Team assignments (2026-08-28)
Ved, Milos, and Adi (Adipat — see [[Projects/MetaGlasses/Repository|Repository]]) are splitting the three-part build. Maps onto the Rev 3 sequence below; owner noted per task.

- **Phone App — Ved**
  - [x] Receive/send Bluetooth audio to talk to the LLM (→ Rev 3 steps 2–4) — ✅ shipped in [[Projects/MetaGlasses/Glance|Glance]]
  - [x] Speech to text — ✅ on-device `SFSpeechRecognizer`, plus a wake word
  - [x] Text to speech — ✅ `AVSpeechSynthesizer`. **"Summarize before speaking" turned out unnecessary**: the single-`text`-field decision made brevity the system prompt's job, so there is nothing long to summarize.
  - [x] **Store offline chat context on the phone** — ✅ SQLite with versioned migrations. ⚠️ *Tension resolved*: the phone is the **source of truth** for conversation history; the backend holds only the lens's latest response per pairing token. The "backend owns current-step state" note applies to the *step cursor within one answer*, not to history.
- **Web app — Milos**
  - [ ] Send API calls to backend (sensor data not needed yet, "buttt we could use it later") — **unblocked as of 2026-09-04**: poll `GET /v1/display?token=`, no auth needed, the pairing token is the credential. Two prerequisites he doesn't control: his origin must be added to `CORS_ORIGINS` (trial/prod reject the wildcard), and a phone must be minting real tokens (blocked on Phase 3 below)
  - [ ] Display the LLM text returned from the backend — plus **client-side truncation as the safety net**, per the single-`text`-field decision
  - [ ] ~~**Connect to phone app (Bluetooth)**~~ — **dropped: not buildable, and unnecessary.** Web Apps have no Bluetooth. The clients coordinate through the backend via the pairing token instead. The underlying question — *can a voice app and a Web App run simultaneously?* — is unrelated and still open under Verify
  - [ ] Agree a **poll interval** (~2s; the lens dims at 20s, sleeps at 25s) and **what renders on a `401`** — an expired pairing is a normal Tuesday, not an error state
- **Backend — Adi** — ✅ **built 2026-09-04**, `metaglasses-backend/` in the working directory
  - [x] Call LLMs through a cloud provider — ✅ OpenAI, `gpt-5.6-sol` by default, `reasoning: {effort: low}`, 160-token cap. ⚠️ Silently reverses the recorded `claude-opus-5` pick in [[Projects/MetaGlasses/Reference|Reference]]'s stack table, and the model id is **unverified**
  - [x] Send the LLM response to the web app for display (text) — ✅ `GET /v1/display?token=`, unauthenticated, pairing token as capability
  - [ ] Receive sensor data from the glasses — **not built, and not needed yet** (was always "could use it later")
  - [x] *Not on the original list but built anyway*: Supabase Auth with JWKS verification, four environments, `POST /v1/state`, CI + GHCR container publishing
  - [ ] **Fallback model providers (2026-09-29, in progress)** — Adi is adding more model providers to `/v1/chat` so the backend can fall back to a different model if the first one fails, rather than the whole turn failing outright. Directly relevant to the NVIDIA timeout finding (93s–150s+ replies, sometimes hitting Supabase's 150s hard `IDLE_TIMEOUT`) — worth checking once shipped whether fallback also helps with *slow* responses, not just outright failures, since that's the more common real failure mode measured so far.

### Additional notes from the same session
- Camera feed will introduce latency to watch for — not urgent (camera is DAT-gated, post-voice-first), but flagged for whenever it's tackled
- Reconfirmed: audio must go through the phone via Bluetooth, "not part of Meta DAT yet" — phrasing implies this could change later; matches the existing Rev 2 finding but worth noting Meta could fold audio into DAT at some point
- Reference link: Meta's DAT toolkit capabilities overview — `https://wearables.developer.meta.com/docs/develop/dat/getting-started-toolkit` (added to [[Projects/MetaGlasses/Reference|Reference]])


## Next up (2026-09-04) — after the voice loop landed
Named by Ved. Both are consequences of the phone-owns-everything architecture finally being real: the voice loop works, so the two things it leans on are now the constraints.

### 1. Integrate with Adi's backend
**Updated 2026-09-04, after reading `metaglasses-backend/`.** The premise that pointing at a real server would be "a Settings change, not a code change" turned out to be **wrong** — it holds for the base URL and nothing else. Glance was built against endpoints nobody owned, and Adi built something meaningfully different. Fifteen discrepancies, numbered D1–D15, with six decisions for Adi numbered A1–A6:
https://claude.ai/code/artifact/a58cd0ce-75ab-41a4-9d61-9631bf2e4b56

**✅ Phases 0–3 done the same day, pushed to `integration` in both repos.** Commit-by-commit summary, verification performed, and what's still open for Adi/Milos:
https://claude.ai/code/artifact/25195d44-06b9-44de-9466-4e38ab2772fe

**What the code answered** (these were the open items here; they are closed as questions, not as work):
- [x] ~~Add auth to the contract~~ — **Adi chose Supabase Auth**, so the phone signs in directly and presents a JWKS-verified access token. `/v1/auth/login` and `/v1/auth/signup` are not being built by anyone; **Glance's login flow changes instead** (D1).
- [x] ~~Error shapes~~ — **agreed by construction.** Backend returns 401/403/413/422/429/503 and Glance already has copy for four of them. Only 403 and 422 are unmapped (D9). *Timeouts remain open* — see below.
- [x] ~~Streaming or not~~ — **not streaming**, decided by construction. Answers are one short sentence, so the latency floor this sets is acceptable.
- [x] ~~Re-ask what the backend is actually for~~ — **answered by what got built**: pairing + lens state, keyless users, one OpenAI call. It is not a chat proxy; BYO-key already took that. The agent/step runtime still does not exist (see Rev 3 step 1 below).

- [x] **Phase 0 — curl the backend before touching the app.** ✅ Done. Ran locally against a real local Supabase CLI stack (not just `AUTH_MODE=disabled`) — created a live test user, confirmed a genuine Supabase JWT round-trips through JWKS verification for `/v1/state` (204), the ownership-conflict path (403), and `/v1/chat` (503 with no key configured — proves auth passed before the model call failed). ⚠️ **D13 still NOT settled**: no OpenAI key was available this session, so whether `gpt-5.6-sol` actually resolves remains unverified. **Still needs a real reachable base URL** — everything above was local-only.
- [ ] **Phase 1 — settle the contract (A1–A6).** Still a conversation to have with Adi, not code. **A1 and A4 done unilaterally** (see below — safe, additive, didn't need his sign-off). **A2, A3, A5, A6 remain open asks.**
  - [x] **A1** optional `system` on `ChatRequest` — ✅ shipped backend-side (`e5fe88c`), appended to the server's own prompt rather than replacing it.
  - [x] **A4** pairing tokens are permanent, fixed the "re-pair on the phone" copy — ✅ shipped (`e5fe88c`).
  - [ ] **A3** drop the implicit lens-state writes in `/v1/chat` — open, needs Adi.
  - [ ] **A5** state the server's upstream deadline so the client's can sit above it — open, needs Adi.
  - [ ] **A6** trial Supabase URL + publishable key, email confirmation off on trial only — open, needs Adi. Blocks testing against anything but a local stack.
  - [ ] **A2** model override — still recommend skipping for v1.
- [x] **Phase 2 — replace the invented auth with Supabase** (D1, D2, D3, D11, D14). ✅ **Shipped** (`f119239`, `30cc5a3`). `@supabase/supabase-js` + Keychain storage adapter; `AuthContext`'s public surface unchanged; `ApiConfig.token` → async `getToken()` + one refresh-and-retry on 401; `src/config/environments.ts` added. `signIn`/`signUp`/`stubSignIn` and the orphaned `authTokenStore` deleted.
- [x] **Phase 3 — pairing and state** (D4, D5, D6). ✅ **Shipped** (`139284b`). 32-hex token, Keychain, never rotated; sent on `/v1/chat`; `/v1/state` posted on every voice transition, fire-and-forget. ⚠️ **Not done**: surfacing the pairing URL in Settings as text + QR — deferred, since the web app's own host isn't decided yet (that's Milos's side of Phase 5).
- [x] **Phase 4 — make a bad turn survivable** (D8, D9, D10) — ✅ **shipped 2026-09-19.** 403/422 copy was already mapped from earlier work. The other two:
  - **Byte-budget the transcript**: `buildTranscript` now trims from the oldest turn by actual UTF-8 byte size (~120KB target, under the backend's 256KB/413 cap) instead of a flat turn count — always keeps at least the single most recent turn even if it alone exceeds the budget, so a bad message can't produce an empty transcript.
  - **`AbortController` deadlines, timeout values revised from the original plan.** The original 12s chat / 3s state numbers predate the NVIDIA timeout finding — a 12s client timeout would now make backend-tier chat **fail 100% of the time**, since NVIDIA alone routinely takes 90–150s. Shipped instead: **160s for `/v1/chat`** (just past Supabase's own 150s platform ceiling, so the client always sees the server's real answer rather than guessing first), **45s for BYO-key providers** (a normal completions call, not NVIDIA's queue — a hang here is unusual), **8s for `/v1/state` and `/v1/display`** (fire-and-forget pings, fail fast). A hung request previously had **no client-side bound at all** — `voiceState` could stay on `thinking` forever with the mic gated.
  - **Spoken failure**: a turn that fails during spoken input now speaks the error aloud (`Audio.speak`) in addition to the text error banner — previously a failed voice question produced total silence, which defeats the point of a hands-free product.
- [ ] **Phase 5 — hand the lens leg to Milos** (D15). Still open. His origin into `CORS_ORIGINS`, poll interval, expired-pairing behaviour.

**✅ The product bug (D7) is fixed on both sides.** Backend accepts the optional `system` field (`e5fe88c`); Glance sends it on both the BYO-key and backend tiers (`139284b`). The same agent now answers consistently regardless of which tier ends up handling the turn.

**Also found and fixed while testing, not originally in the discrepancy list**: a chat had no agent-picker UI, so `hasOwnKey` was always false and **no saved key of any kind was ever reachable** from an ordinary new chat — see "Agent picker" below, now closed.

**Verified working end to end on a physical device** (Ved's iPhone, not just the simulator) with a live Microsoft Foundry deployment through the BYO-key path — see [[Projects/MetaGlasses/Glance|Glance]].

### 2. Context management and local storage
`buildTranscript` currently sends the **last 200 turns, unconditionally** (raised from 40 during the wake-word work, which was a stopgap, not a design). A turn count is not a budget.

- [ ] **Make the transcript budget token-aware**, not turn-aware. 200 short voice turns and 200 long typed ones are wildly different payloads, and the full transcript is **paid for on every single turn**. This is the item already flagged unowned since 2026-08-28 — it is now the live one.
- [ ] **Decide what happens to history past the budget**: hard truncation, a rolling summary folded into the system prompt, or retrieval over older turns. Truncation is cheapest and silently loses things the user can still *see* in the UI, which is the failure mode worth avoiding.
- [ ] **Notes and chats are separate silos.** A note cannot be pulled into a chat as context, which is arguably the main reason to have both in one app. "What did I note about X?" should work.
- [ ] **No search.** Notes are deliberately flat — no title, no folders, no tags — which is right for capture and wrong for retrieval once there are more than a screenful. op-sqlite supports **FTS5**; an index is cheap and does not compromise the flat capture model.
- [ ] **Nothing is ever pruned.** No archive, no delete-old, no size ceiling. Voice notes accumulate silently and the DB only grows.
- [ ] **Long notes truncate at ~60s** (`SFSpeechRecognitionTask` expiry). The 55s backstop ends them cleanly rather than fixing it; a real fix needs task restart plus transcript stitching. See [[Projects/MetaGlasses/Reference|Reference]].

### 3. Glasses photo → chat attachment (scoped 2026-09-18)
The eventual "look at what I'm seeing and explain X" turn: a photo from the glasses' camera, attached to a chat message, alongside the spoken question. Scoping this **reopened, then reaffirmed, the context-storage question** below — see [[Projects/MetaGlasses/Reference|Reference]] for the original 2026-08-28 decision.

- [x] ~~Should this move conversation history to the backend?~~ — **no, and the image feature is not a reason to.** `metaglasses-backend` already added real image support (`POST /v1/images` → `imageId`, `DELETE /v1/images/{id}`, a message's `imageIds[]` resolved server-side to signed URLs for the model). That storage is backend-side by necessity — a photo is a binary blob a vision model must fetch — **independent of where conversation history lives**. The phone still owns and sends the full transcript every request, unchanged; a turn just carries one more field, the `imageId`(s) it references. See the backend finding write-up for the latency context that came out of the same investigation: https://claude.ai/code/artifact/5a842f57-e65a-4d98-946d-b971430af441
- ⚠️ **Backend image storage is deliberately temporary, not an archive.** A `cleanup-pairing-images` Edge Function purges the bucket; the signed URL TTL is 10 minutes. If a photo should still be visible when scrolling back through chat history later, **the phone has to download and cache the bytes locally near capture time** — the backend's copy will be gone by then. This is now the one piece of new local-storage work the feature actually needs.

**Glance side — ✅ shipped 2026-09-18, phone-native version.** Built the phone-camera/photo-library path first rather than waiting on DAT — same backend pipe, different (buildable-now) source:
- [x] `src/media/imageSource.ts` (new) — `takePhoto()` / `pickImageFromLibrary()`, both resolving to one shared `PickedImage` shape. This is the seam a future glasses source plugs into; nothing downstream needs to change when it does.
- [x] `src/api/client.ts` — `uploadImage()` / `deleteImage()` against `POST/DELETE /v1/images`, raw-bytes body (not multipart), matching the backend's actual contract
- [x] `src/db/` — `message_images` child table (migration 4), `AttachedImage`/`NewAttachedImage` types, `addMessage`/`listMessages`/`buildTranscript` all updated. Base64 cached locally at attach time — the backend's own copy is temporary, this is the durable one.
- [x] `ChatScreen.tsx` — attach button (camera or library via an action sheet), thumbnail preview before sending, inline image rendering in the chat bubble
- [x] A photo always routes through the backend tier for now, even with a BYO-key agent selected — none of the four BYO-key providers have vision wired up yet, so sending it there would silently drop the photo. Flagged, not fixed.
- [x] **Confirmed working end to end on the physical iPhone** — attached a photo of a Lego set, the backend (NVIDIA) correctly identified it in the reply.

**MetaGlassTest / DAT side — ✅ camera capture shipped and confirmed working, 2026-09-19.**
Audited 2026-09-18 (permission wired, capture not); built and verified the same week, direct in `MetaGlassTest` (not Adi's to build after all — Ved's own repo, not blocked on anyone):
- [x] `checkCameraPermission()` / `requestCameraPermission()` — already exposed
- [x] `deviceSession` already populated on `connect()`
- [x] **`MWDATCamera` linked as a product** — wasn't in the Xcode project at all; added by hand-editing `project.pbxproj` (no Xcode GUI available), mirroring the existing `MWDATCore`/`MWDATDisplay` pattern
- [x] `capturePhoto()` — starts a camera stream, waits for `.streaming`, calls `capturePhoto(format: .jpeg)`, resolves with base64 JPEG. Bridged through to JS as `MetaWearables.capturePhoto(): Promise<CapturedPhoto>`
- [x] `NSCameraUsageDescription` added to `Info.plist`
- [x] **Two real DAT-lifecycle bugs found and fixed, both via actual compiler/runtime errors, not guessing:**
  - `Stream` collided with Foundation's own `NSStream` — needed the fully-qualified `MWDATCamera.Stream`
  - DAT allows exactly **one camera capability per session** — the original tear-down-after-every-photo design threw `"A capability of this type is already active on the session"` on a second capture. Fixed by setting the stream up once and reusing it for every capture after, only tearing down on `disconnect()`
- [x] **Mock Device Kit needs a feed, not just a result image** — `setCapturedImage(fileURL:)` alone only sets what `capturePhoto()` *returns once already streaming*; without also calling `setCameraFeed(cameraFacing: .front)` the stream has nothing to stream and goes straight to `.stopped`. Found the exact argument label (`cameraFacing:`, not `facing:`) from a real compiler error, not the docs
- [x] **Confirmed working end to end on the simulator, twice in a row** — a generated placeholder image (no bundled asset needed) round-tripped through a real DAT camera stream via Mock Device Kit, using the Mac's own camera as the feed source

### 4. Real unit test coverage (2026-09-18)
Named as a standing priority, not tied to one feature. Started with the modules that actually carry business logic rather than chasing an aggregate number — screens (mostly layout) and the BYO-key providers (a different network surface, bypasses the backend entirely) are explicitly not yet covered.

- [x] **Test infrastructure**: `__tests__/testUtils/opSqliteTestDouble.ts` wraps a real embedded SQLite (`better-sqlite3`) in op-sqlite's `execute()`/`close()` shape, so db tests exercise real migrations, foreign keys, and cascades — not a stub that always returns `[]`. Reusable for every future db test.
- [x] **db/** (`database.ts`, `chats.ts`, `agents.ts`, `notes.ts`) — ~100%. Migrations, every CRUD path, cascade behavior, the image-attach schema from item 3 above.
- [x] **api/client.ts** — 100%. Every endpoint, HTTP error mapping, the 401 retry-once dance, the stub fallback, raw-bytes image upload.
- [x] **chat/activeTurn.ts** — 98%. Typed vs. spoken input, BYO-key vs. backend routing (a photo always forces backend), the busy guard, per-chat isolation, failure handling.
- [x] **The full backend-connection chain**, prompted by a direct question ("are all connections to the backend tested?"): `pairing.ts` (100% — token reuse vs. minting, matches the backend's `^[0-9a-fA-F]{32}$` contract exactly), `auth/supabaseClient.ts` (100% — per-environment client memoization, throws clearly on an unconfigured project), `auth/AuthContext.tsx` (81% — session restore, sign-in/up/out, the Dev/Prod switch signing out of the environment being left).
- [x] **Two real bugs, both in the tests, not the app** — worth remembering: `.find()` grabbing the wrong event when a type fires twice in one turn (every turn clears its notice/error banner first, then may set a real one — fixed with a last-match helper); `jest.clearAllMocks()` only clears call history, not a `mockResolvedValue` a previous test configured, so later tests silently inherited stale mock behavior and appeared to hang — fixed with `jest.resetAllMocks()` and one `beforeEach` as the single source of truth per file.
- [x] Tried `@testing-library/react-native` for the `AuthContext` hook tests — `result.current` came back `undefined` on this RN 0.87/React 19 combination. Abandoned rather than fought; used the same hand-rolled `react-test-renderer` + `act()` pattern already proven in `App.test.tsx` instead. Worth retrying if the library catches up.
- [ ] **Not yet covered**: `api/providers.ts` (the four BYO-key vendors), `secrets.ts` (partial), `media/imageSource.ts`, `audio/Audio.ts`, screens.

**Coverage moved from 11% to ~39% overall** — the aggregate number matters less than which 39%: it's now concentrated in exactly the modules where a silent bug would be expensive (money spent on a bad request, a session pointed at the wrong project, a turn that hangs forever).

### Also carried forward
- [x] ~~**Agent picker when starting a chat.**~~ — ✅ **Shipped 2026-09-04** (`a898fd3`). Found while testing Foundry: without this, `hasOwnKey` was always false for a new chat, so *no* saved key from *any* provider was ever reachable — not a Foundry-specific bug, a standing one. "New chat" now asks which agent (or "no agent," unchanged default) before opening. Wake-word chats deliberately skip it — hands-free entry shouldn't gain a picker step.
- [ ] Tune the **voice-activity threshold** (3.5× the adaptive noise floor) against real glasses rather than AirPods in a quiet room.

## Build sequence (Rev 3 — 2026-08-26)
**Superseded the greenfield Rev 2 plan.** There is now a real codebase ([[Projects/MetaGlasses/Repository|Repository]]) with working DAT display + on-device LLM bridges and no audio. Build on it rather than starting over. Ordered so nothing needs hardware until step 4.

- [ ] **1. Agent runtime, no client.** HTTP endpoint: request in → **ordered list of steps** out. Provable with `curl`. Use structured outputs so steps come back as data, not prose. Holds current-step state per session. Unblocked by every hardware question below.
  - ⚠️ **Partially superseded, and still open (2026-09-04).** Adi's backend is the "HTTP endpoint, provable with `curl`" half — but it returns **one sentence, not an ordered list**, and holds **no step state**. That is not a miss: the 2026-08-28 single-`text` decision made brevity the system prompt's job and deleted the reason for structured steps. So what exists is a stateless one-shot wrapper, the *runtime* does not exist, and **nothing currently needs it to**. Reopen this only if "next"/"repeat"/"back" (step 6) turns out to need a server-side cursor — it may not, since the phone owns history and could hold the cursor itself.
- [x] **2. Add an `Audio` native module to the existing RN app.** ✅ **Done 2026-08-28** on branch `appDev`. `ios/MetaGlassTest/AudioModule.swift` + `.m`, `src/audio/Audio.ts` + `types.ts`, Voice loop card in `App.tsx`, mic + speech usage strings in `Info.plist`, files registered in the Xcode project. Builds clean, lint + Jest + tsc pass, runs on iPhone 17 sim. **Confirms `.allowBluetoothHFP` exists on the iOS 26 SDK** — the symbol Meta's docs hedged about.
  - Promise-based `NSObject` bridge matching the existing modules; **JS owns the state machine**, native holds no conversation state
  - `startListening(preferGlasses)` resolves with the final transcript; auto-endpoints after 1.5s of silence, or `stopListening()` forces it
  - `speak(text, preferGlasses)` **tears down capture first** — that's the mic gate, no barge-in
  - Uses on-device `SFSpeechRecognizer` when supported (confirmed supported, `en_US` on this machine)
  - `preferGlasses` flag already wired but defaults false → **step 4 is flipping a toggle, not writing code**
- [x] **3. Wire the voice loop to the agent runtime.** ✅ **Done for the BYO-key path 2026-09-04** — confirmed working end to end on a physical iPhone with a live Microsoft Foundry deployment: speak/type a request, an agent answers, using that agent's own instructions. ⚠️ **Not yet done for a keyless user**: the backend tier is fully wired (Phases 0–3 above) but still 503s with no OpenAI key configured, so the same loop for someone who brought no key remains blocked on Adi, not on the client.
- [ ] **4. Point audio at the glasses.** HFP route selection, verify `currentRoute` is the glasses. **Verifies the load-bearing claim** — that voice works with no DAT SDK linked. **Built and working; not yet run on the glasses.** `preferBluetooth` selects the HFP input and the app checks the route landed on it, tested against AirPods (same `.bluetoothHFP` path). Only the confirmation on real hardware is outstanding — which is the entire point of the step, so it stays open.
- [ ] **5. Attach the display to step state.** The bridge already does this — `display.showText()` exists and works. Feed it the current step so the lens mirrors what's being spoken. Small piece of work precisely because Adipat already built it.
- [ ] **6. Decide turn-taking, then make it feel good.** Stay in HFP vs. toggle to A2DP. Endpointing, barge-in, "next"/"repeat"/"back" voice commands (and/or Neural Band pinch to advance). Where a voice agent is won or lost.
- [ ] **7. Isolate DAT behind a build flag.** Do this *before* the app grows — currently ~10 files. Keeps a publishable voice-only build alongside the full voice+lens build.
- [ ] **8. Build the console last.** By then you'll have hand-written several agents and know what a definition needs.

### Design notes for the step model
- Agent output is a **structured list of steps**, not free prose — makes it renderable on the lens and navigable by voice
- Backend owns current-step state (the glasses retain nothing)
- Each step needs a short form for the lens (large type, no scrolling) and a spoken form
- Lens dims at 20s / sleeps at 25s — a long step needs the display re-fed

## Design decisions settled 2026-08-28
Answers from Ved; encoded in the API contract draft (link in [[Projects/MetaGlasses/Reference|Reference]]).
- [x] **Backend is stateless** — no conversation history server-side
- [x] **Phone owns all chat history, stored locally** — deliberate, avoids cloud storage cost
- [x] **Phone sends the full transcript every request** (matches how the Claude API works anyway)
- [x] **Lens shows the most recent response only** — no history, no navigation, deliberately simple
- [x] **STT and TTS run on the phone** — backend is text-in/text-out, never touches audio
- [x] **Audio is phone↔glasses Bluetooth only** — the web app can't do audio and never will
- [x] ~~Pick the voice framework (LiveKit vs Pipecat)~~ — **moot.** On-device STT/TTS + stateless backend means there's no audio pipeline to orchestrate
- [x] ~~Confirm cascaded vs speech-to-speech~~ — cascaded by construction, with STT/TTS on-device

### Still open from the original stack list
- [ ] **Decide whether the product has an async mode** ("ask, walk away, get told when done") — cheap now, expensive to retrofit
- [x] ~~Confirm backend language~~ — **Python, settled by Adi building it (2026-09-04).** FastAPI + `uv`, ruff + pytest, containers to GHCR. The original Python argument was framework-driven and became moot; it landed there anyway on Adi's own preference, which is the only reason that matters now.

## New decisions needed (from the contract draft, 2026-08-28)
- [x] ~~Agree a separate `display` field~~ — **decided 2026-08-28: single `text` field.** Ved pushed for simplicity and it's the better call. Keeping answers short becomes the *system prompt's* job (one step, one short sentence) — which is correct behaviour for a step-by-step assistant anyway, and makes "summarize before speaking" unnecessary since nothing is long to begin with. Accepted failure mode: models don't reliably obey length limits, so the web app truncates client-side as a safety net. Adding `display` later is **additive, not breaking**, so revisit only if real responses actually overflow
- [x] ~~Is barge-in in scope for v1?~~ — **decided 2026-08-28: no barge-in. Gate the mic while speaking.** Simpler build: no echo cancellation, no `setVoiceProcessingEnabled`, no half-duplex tuning — just a hard gate. Wearer waits for the sentence to end, which pairs well with the single short `text` field (little to interrupt). Makes the audio module a clean state machine: `idle → listening (mic OPEN) → thinking (mic OPEN) → speaking (mic GATED) → idle`, mirrored exactly by the API's `state` field
- [x] **Agree transcript trimming happens on the phone**, not silently server-side — **built, but crudely**: a flat last-200-turns cap with no token budget. See "Context management" above; the *policy* is settled, the *implementation* is not.
- [x] **Confirm the pairing model** — ✅ **built server-side 2026-09-04, exactly as proposed.** Phone mints a 32-hex token, sends it on `/v1/chat` and `/v1/state`; the lens polls `GET /v1/display?token=` with no auth, the token itself being the capability. Client side is unbuilt (Phase 3).
  - ⚠️ **The consequence nobody wrote down: the token can never be rotated.** It is baked into the Web App URL the user added by hand, and Web Apps have no text input to re-enter a new one. The backend's `401` copy currently says "Re-pair on the phone," which reads as *mint a new token* — that would break the URL. The **record** is ephemeral (in-memory, 1hr TTL from last phone request, wiped by any redeploy); the **token** must be permanent. Client fix: post `/v1/state` on foreground, which silently re-registers. Backend fix: A4, change the copy.

## Superseded — stack decisions (2026-08-25)
Kept for rationale; most are now answered above. Full detail in [[Projects/MetaGlasses/Reference|Reference]] + the voice stack artifact.

### Deferred — behind adapters, decide with real data
- [ ] ASR vendor: benchmark AssemblyAI Universal-3.5 Pro Realtime vs Deepgram Nova-3 **on real 8 kHz glasses audio** (published rankings used ordinary mics)
- [ ] TTS vendor: optimize latency + price only — HFP caps output at 8 kHz so fidelity is inaudible on-device
- [ ] Hosting: persistent workers required (Fly.io / Railway / Render / ECS) — **the agent runtime cannot be serverless**

## Repo issues to raise with Adipat
Detail in [[Projects/MetaGlasses/Repository|Repository]].
- [x] ~~Mock device may not be display-capable~~ — **CONFIRMED 2026-08-26.** `GlassesModel` has no display model (`metaRayBanDisplay` exists in `DeviceType` but not in the mock kit); running the app shows `connected · displayless`. Not Adipat's bug — a gap in Meta's mock kit. **The display half is hardware-only.** Reinforces voice-first: the voice loop is simulator-testable, the display half isn't
- [ ] **`external-accessory` in `UIBackgroundModes` + DAT linked unconditionally** → App-Store-blocked as built. Build-flag split is cheap now
- [ ] `NSLocationWhenInUseUsageDescription` is an empty string — App Store rejects empty purpose strings. Fill or remove
- [ ] Discuss whether on-device Gemma survives contact with the voice loop (~2.5GB resident, and ASR/TTS will be cloud anyway)
- [ ] Agree the audio module goes in *this* repo, following the existing bridge pattern

## Verify
- [ ] **Does the voice loop work with no DAT SDK linked?** *(blocking)* Load-bearing inference, not a documented guarantee — Meta publishes the audio guidance inside the DAT docs. Confirm at step 3.
- [ ] **Is 8 kHz mono good enough for your ASR?** *(blocking)* Telephone quality, beamformed. Modern ASR copes but accuracy drops. Test before designing interactions needing reliable names/addresses/numbers. **Can be answered early with AirPods** — same `.bluetoothHFP` path as the glasses, so no hardware needed. Requires a real iPhone; the Simulator has no Bluetooth stack.
- [ ] How does the wearer *start* a conversation? DAT lists "voice invocation" as a reviewable permission but gives no API detail. Without it, sessions start from the phone. Ask Meta or search the dev forums.
- [ ] Can a voice app and a Web App run simultaneously? If the phone holds the BT audio route while the glasses browser renders a Web App, that's a *publishable* route to voice + lens. Undocumented, may not work, cheap to test after step 3.
- [ ] Attend Meta Connect (Sept 23–24, 2026) — if DAT publishing opens, the build-flag split stops mattering and Build B ships.

## Answered — no longer open
- [x] **Can the agent be spoken to through the glasses?** Yes — HFP mic in, A2DP/TTS out, via plain AVFoundation. This reversed the Rev 1 architecture.
- [x] Confirm whether DAT apps can publish to the App Store — **no, still blocked** (`ExternalAccessory`/MFi, confirmed verbatim in Meta docs). But *linking DAT* is what causes it, which is why the build flag works.
- [x] Build an architecture flowchart — done (Rev 2, three diagrams); see artifact link in [[Projects/MetaGlasses/Reference|Reference]]
- [x] Research glasses sensor/capability access — full matrix in [[Projects/MetaGlasses/Reference|Reference]]
- [x] Reconcile React Native with DAT — RN is console-only either way; the client is Swift. Less compelling now, since a plain web console is faster.
- [x] Verify the CocoaPods/React-Native-bridging claim — **false.** DAT ships via Swift Package Manager. Cut.
- [x] Decide fate of the Mise cooking prototype — **archived** to `archive/mise-cooking-app/` with an `ARCHIVED.md`.

## Reopened
- [ ] **MediaPipe / Gemma on-device inference** — Rev 1 cut these because a web app can't reach phone-native models. On a native iOS client that reason no longer holds. Not v1 (cloud is faster to build and likely lower latency for a voice loop), but revisit for privacy or offline. Would also finally settle the unverified "local iPhone inference is inefficient" claim.

## Optional — Web App client
Only if you want lens visuals in front of the general public before DAT publishing opens. Can't do voice, so it's a secondary client, not the platform.
- [ ] Generic shell at `/a/<agentId>`, 600×600, D-pad nav
- [ ] Resolve the text-input contradiction (Meta's docs say unsupported; the bundled `add-text-input` skill says HTML inputs open an on-glasses composer)

## Dropped — pre-pivot cooking-app todos
Serving-size scaling, hosting Mise, QA checklist against Mise, offline service worker, Apple subscription-billing terms. All tied to the archived cooking direction.
