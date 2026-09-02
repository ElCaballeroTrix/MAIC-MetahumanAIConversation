# MAIC — MetaHuman AI Conversation

**Real-time conversational AI for MetaHumans in Unreal Engine 5.**

A MetaHuman that listens, answers in its own voice, and *acts on your level* — changing the world through AI tool calls, not just talking about it. Speech in, speech out, live lip sync, and a function-calling bridge into your gameplay code.

Core in C++, integration layer in Blueprint. Built by two developers.

> **About this repository.** MAIC is a commercial plugin distributed on Fab. This repo is a technical write-up: architecture, data flow, and the non-obvious problems we had to solve. It does not contain the production source.

## ⚠️DISCLAIMER. PLUGIN NOT OFFICALLY IN THE MARKET (WIP)
---

## Demo

<img width="800" height="450" alt="MAIC_Gif" src="https://github.com/user-attachments/assets/772fd6ba-d5c2-479d-a555-3d3b7ca2bfbd" />


---

## Pipeline

```
Player holds Talk ──▶ mic captured via Submix ──▶ downmix + resample → 24 kHz PCM16
                                                          │
                                     base64 chunks ──▶ OpenAI Realtime API (WebSocket)
                                                          │
                        ┌─────────────────────────────────┴────────────────┐
                        ▼                                                  ▼
              function call events                            audio deltas (PCM16 24 kHz)
                        │                                                  │
             argument fragments buffered                     ┌─────────────┴─────────────┐
                        │                                    ▼                           ▼
             dispatch into gameplay                 USoundWaveProcedural       OVRLipSync → visemes
          (lights, subtitles, scripts…)                (playback)                        │
                        │                                              viseme queue + fixed-rate timer
             synthetic follow-up prompt                                                  │
             so the character reports back                            15 OVR visemes → ARKit curves
                                                                                         │
                                                                    custom FAnimNode injects curves
                                                                       into the face AnimGraph
```

---

# The AI layer

### 1. Tool calling: streamed fragments reassembled into gameplay actions

This is what turns a talking head into a character with agency. Ask it to dim the lights, and the lights dim.

The Realtime API doesn't deliver a function call in one piece. It arrives as a sequence of events:

- `conversation.item.added` → the tool *name*
- `response.function_call_arguments.delta` → the arguments, **in fragments**
- `response.output_item.done` → now it's safe to act

The component holds the pending tool name, accumulates argument fragments in a buffer, and only parses and dispatches once the call is complete. Dispatch is a lookup from tool name to a native handler that reaches into gameplay: scene lights, UI state, external processes.

The subtle part is what happens *after* execution. The model has no idea whether the action succeeded — as far as it's concerned, it emitted a function call into the void. Left alone, the character acts and then goes silent, which reads as a bug to the player. MAIC feeds a short synthetic prompt back into the session immediately after dispatch, instructing the character to report the concrete result. So the lights change *and* the character says "done, they're blue now." Closing that loop is what makes tool use feel conversational rather than mechanical.

### 2. Designing the session: personality, modality, and turn-taking

Everything about how a character behaves is negotiated in a single `session.update` at connection time: voice selection, audio formats in both directions, transcription, turn detection, the tool schema, and the character's instructions.

Two decisions worth calling out:

**Instructions live outside the engine.** A character's personality can be authored in a plain `.txt` file referenced by path, optionally merged with inline text set per-instance. Writers iterate on how Julius Caesar talks without opening Unreal, recompiling, or touching a Blueprint. Instruction text is escaped before being embedded in the session payload — quotes and newlines in prose would otherwise break the JSON silently.

**Server-side voice activity detection with interruption enabled.** The model decides when the player has stopped speaking, and an in-progress response can be cut off when the player starts talking again. That's what makes the exchange feel like a conversation instead of a walkie-talkie.

### 3. One event stream, many features

A single WebSocket message handler is the state machine behind everything the plugin does. Response transcripts feed subtitles and chat bubbles; audio deltas feed playback and lip sync; function-call events feed the dispatcher; server errors are surfaced explicitly, because a rejected session config otherwise fails silently and leaves you debugging a character that simply won't speak.

The same component supports three interaction modes over that one connection: hold-to-talk voice, typed questions that come back as speech, and a full ChatGPT-style text chat — configurable per character.

### 4. Per-character conversation budget

Characters can be given a hard limit on how many questions a player may ask, with a configurable audio response when the budget runs out, and a reset hook. Useful for pacing narrative beats — and, more practically, for capping API spend in a shipped demo.

---

# The audio & animation layer

### 5. Viseme timing: a jitter buffer for the face

The naive approach — generate visemes from each audio chunk and apply them on arrival — fails, because **WebSocket chunks do not arrive at the rate audio plays back.** Deltas come in network bursts; the audio queue drains at a constant 24 kHz. Driving the face off arrival time makes the mouth run ahead of the voice and then stall.

MAIC decouples them. Incoming PCM is cut into fixed 20 ms frames; each frame produces one viseme set pushed onto a queue. A separate repeating timer pops **exactly one frame per tick** and broadcasts it to the animation instance. Arrival rate and consumption rate become independent, so the mouth stays locked to the audio regardless of jitter. When the queue empties, the timer broadcasts a zeroed viseme set, so the mouth closes instead of freezing mid-phoneme.

This is the single decision that separates "the mouth moves" from "the character is talking."

### 6. Getting mic audio out of Unreal in the format the API wants

The Realtime API expects mono PCM16 at 24 kHz. Unreal hands you interleaved float samples at whatever rate the audio device runs.

A custom `ISubmixBufferListener` does the conversion per buffer: downmix N channels to mono → resample to 24 kHz (fast ×2 decimation for the common 48 kHz case, generic linear interpolation otherwise) → float `[-1,1]` to PCM16 little-endian → hand the bytes to the game thread via `AsyncTask`, since the submix callback runs on the audio thread and the rest of the pipeline is not thread-safe.

Downstream, an accumulator batches those bytes into chunks large enough to justify a WebSocket frame before base64-encoding them. Sending every buffer individually would flood the socket with tiny messages.

### 7. Visemes → ARKit: mapping where no standard exists

Oculus LipSync outputs 15 visemes (`sil, PP, FF, TH, DD, kk, CH, SS, nn, RR, aa, E, I, O, U`). MetaHumans are driven by ARKit blendshape curves (`JawOpen`, `MouthFunnel`, `MouthPucker`, `MouthStretch*`, tongue controls…). **There is no canonical mapping between the two.**

We built a weighted mixing matrix by hand. Jaw opening, for instance, is a weighted sum of the vowels plus smaller contributions from `dd/nn/rr/kk`, then attenuated by how closed the mouth currently is — so a plosive can't come out open-mouthed. Every coefficient came from iterating against reference footage. The matrix is exposed so users can retune it for their own characters.

### 8. Injecting curves without rebuilding the face rig

Rather than asking users to rewire the MetaHuman face AnimGraph, MAIC ships a custom `FAnimNode_Base` that evaluates the incoming pose and writes **only** into `Output.Curve`. Users duplicate Epic's `ABP_Face_PostProcess`, drop in one node, and their existing facial animation keeps working — AI lip sync layers on top instead of replacing it.

Integration cost: one node. That mattered more for adoption than any other design choice in the plugin.

### 9. Decoupling the player from the NPCs

The player's Talk button holds no reference to any MetaHuman. It broadcasts a Gameplay Tag through `GameplayMessageSubsystem`; every AI character listens and independently decides whether the instigator is inside its own hearing radius. Adding a tenth NPC to a level requires zero changes on the player side. Direct `StartRecording`/`EndRecording` calls remain exposed for users who'd rather not use the message bus.

A smaller detail with a real cause: input binding runs through a retry timer, because component initialization order isn't guaranteed and the owner's `UEnhancedInputComponent` may not exist yet when `BeginPlay` fires.

---

## Architecture

| Piece | Role |
|---|---|
| `UAIConversationComponent` | Core. Extends `UAudioComponent`. Owns the WebSocket, the Realtime event state machine, audio queueing into a `USoundWaveProcedural`, the viseme buffer, and tool dispatch. |
| `UAudioRecordingComponent` | Extends `UAudioCaptureComponent`. Mic capture, proximity gating, start/stop via Gameplay Tags. |
| `MAIC_SubmixListener` | Audio-thread submix listener: downmix, resample, PCM16 conversion. |
| `ULipSyncAnimInstance` | Receives visemes, converts them to ARKit curve values. |
| `FMAIC_ModifyArkKitPoses` | `FAnimNode_Base` that writes curves into the face pose. |
| `UStartTalkComponent` | Player-side Enhanced Input → Gameplay Tag broadcaster. |
| `UMAIC_RichTextBlock` | Rich text with typewriter reveal, for chat and subtitles. |
| `UMAICSettings` | `UDeveloperSettings` — endpoint and credentials in Project Settings. |

**C++ owns** networking, audio DSP, protocol parsing, and the animation node.
**Blueprint owns** component wiring, delegate-to-widget binding, montages, and per-character configuration.

The split was deliberate: everything latency- or thread-sensitive is native; everything a designer needs to touch is exposed.

---

## Features built on top of the core

- **Tool calling into gameplay** — the character changes the world, not just the conversation
- **Voice, text, or both** — per-character response modality
- **Live subtitles** driven by response transcripts
- **ChatGPT-style text chat**, with typewriter reveal
- **Welcome speeches** — lip sync generated from a pre-authored audio file, so a character can speak before the player says anything
- **Per-character personality** via inline instructions or an external `.txt` file
- **Question limits** per character, with a configurable "out of questions" response
- **Proximity gating** with an editor debug visualization

---

## Stack

Unreal Engine 5 · C++ · Blueprints · MetaHuman · OpenAI Realtime API (WebSocket) · Oculus LipSync · Enhanced Input · GameplayMessageSubsystem · ARKit blendshape curves

---

## Known limitations & roadmap

- **Tools are registered in C++.** Adding one means editing the schema and the dispatcher. A data-driven tool registry (DataAsset + Blueprint-implementable dispatch) is the top roadmap item.
- **Credentials live on the client.** Suitable for prototypes, demos and internal projects; a shipping title needs a backend proxy issuing ephemeral tokens.
- **Transcription language and model are fixed** in the session config; moving both into settings is planned.
- **Downsampling uses decimation without an anti-aliasing pass** — acceptable for speech, but an FIR/sinc filter would be more correct.
- **No reconnection strategy** for dropped WebSocket connections yet.

---

## Authors

[Santiago Caballero](https://github.com/ElCaballeroTrix/Santiago_Portfolio) · [David Alamo](https://www.linkedin.com/in/davidalamo/) — designed and built together.

## Availability

Available on Fab: `[link]`
