# VoicePrompt Android — Specification (v2)

Local-first voice dictation for Android. The WisprFlow experience without the
WisprFlow tradeoffs: no cloud, no account, no accessibility service required,
microphone permission only. First target: v1.0-android MVP.

Lineage: macOS original (`voiceprompt.py`) → Windows port (`voiceprompt_windows.py`,
v4.2) → this. The record → transcribe → insert architecture and the custom
vocabulary (`vocabulary.txt`) carry forward.

> **v2 changelog (2026-11-21, Myles review pass over Arty's v1, afcd5b3 + b930984):**
> 1. Form factor decided: dictation-first IME with minimal QWERTY fallback and a
>    one-tap switch-back key. The "thin dictation bar above the user's existing
>    keyboard" variant is removed — Android runs one IME at a time, so it cannot
>    be built as written.
> 2. Microphone permission flow added: an IME cannot request runtime permissions
>    (no Activity context), so a first-run onboarding Activity owns the prompt.
> 3. Latency targets made hardware-conditional (reference device: Pixel 8 Pro);
>    "processing" is a designed-for normal state; auto-insert timer starts at
>    transcript-ready, not at the tap.
> 4. F-Droid added as a first-class channel; Play IME review prep added to M0.
> 5. Battery target changed from promise to measured-and-published number.
> Everything else preserves Arty's v1 text.

## 1. Problem and positioning

Wispr Flow on Android works well but requires: an Accessibility Service
(read screen content + observe text), microphone access routed through cloud
processing, and an account/subscription. For users whose dealbreaker is
permissions and privacy, there is no local-first alternative today. Woof
(woof.talk) proved local dictation with a compact model is viable on desktop;
nothing equivalent exists on Android.

**Positioning:** the free, local, offline, minimal-permission dictation keyboard.
Not a WisprFlow clone: fewer permissions is the product.

**Two-keyboard honesty:** v1 is a dictation tool you deliberately switch to,
not a Gboard replacement. While VoicePrompt is the active IME, typing runs on
our minimal engine (no glide, no server-grade prediction — permanent design
boundary). Gboard stays the daily driver for thumbs; the built-in switch-back
key makes the round trip one tap. v2's optional bubble is where dictation and
any keyboard coexist.

## 2. Principles (non-negotiable)

1. **All inference on-device.** Zero network calls after model download. The
   release build must work in airplane mode (a CI test asserts it).
2. **Minimum permissions.** v1 requests exactly: RECORD_AUDIO. No accessibility
   service, no overlay, no storage, no contacts, no network permission at
   runtime (declare `android.permission.INTERNET` only for the model downloader,
   or split it into a download-time only module / first-run flow). Consequence:
   the RECORD_AUDIO prompt needs a first-run onboarding Activity, because an
   InputMethodService has no Activity context and cannot request runtime
   permissions itself (§5).
3. **No account, no telemetry, no analytics.** Crash reporting optional and
   off by default.
4. **User-owned models.** Models download on demand, live in app storage,
   deletable from settings. Ship no model in the APK beyond a stub.

## 3. Architecture

```
app/                    Kotlin, minSdk 29 (Android 10), target latest
  ime/                  InputMethodService: keyboard row + mic key + preview strip
  onboarding/           First-run Activity: mic permission, model download,
                       enable-keyboard walkthrough, vocabulary seed
  engine/               whisper.cpp JNI wrapper (libwhisper_jni.so), VAD, audio capture
  models/               Model manager: download, verify (sha256), select, delete
  polish/               Post-processing: punctuation, filler rules, spacing
  benchmark/            Debug-only activity: WER quickcheck + latency harness
```

**Why IME-first:** a keyboard can insert text into any focused field by design.
No overlay, no accessibility service, no special permissions beyond the mic,
and no Play Store policy friction (accessibility APIs now require justification
and human review). The tradeoff: users must switch to our keyboard to dictate —
v1 embraces that with a one-tap switch-back key rather than pretending
otherwise. v2 adds the optional floating bubble with an honest permission
disclosure.

## 4. Engine plan

- **Baseline:** whisper.cpp (`whisper.android` bindings), GGUF int8 quantized.
  - Default: `base.en` int8 (~80 MB) for speed on mid-range devices.
  - Optional: `small.en` / `medium.en` for accuracy, user's choice (mirrors the
    Windows port's model dropdown).
  - NNAPI / GPU delegate where available; pure-CPU fallback path.
- **Spike (week 1):** evaluate Underdog Bark 3.0 (Hugging Face, ConwayResearch).
  163 MB with claimed ~4.9% WER is exactly phone-sized. Gate on: (a) license
    permits redistribution/use, (b) convertible to ONNX Runtime Mobile or
    ExecuTorch, (c) real-time factor ≥ 1.5x on a mid-range device. If it passes,
    it becomes the default; whisper.cpp stays the fallback.
- **Custom vocabulary:** port `vocabulary.txt` as an initial-prompt/glossary
  (names, places, fleet vocabulary). User-editable in settings.
- **VAD:** Silero VAD (ONNX) for endpointing, same as the Windows port's
  `vad_filter`.
- **Reference device: Pixel 8 Pro (Tensor G3).** Published latency numbers are
  defined against it; the newest Pixel Pro is the flagship tier and one older
  device is the informational floor. The < 2.0s target (§7) assumes the
  NNAPI/GPU delegate is active; CPU-only can run 2-3x slower, and the UI treats
  "processing…" as a normal state, not an error.

## 5. v1.0 scope (MVP)

1. **Dictation-first IME (form factor, decided):** prominent mic key, live
   waveform + transcript preview strip, over a minimal but functional QWERTY
   (tap typing, basic autocorrect seeded from the same vocabulary engine) so a
   user is never trapped mid-sentence — plus a **one-tap switch-back key** that
   returns to the previously active keyboard. Explicitly never built: glide /
   swipe typing, server-grade prediction.
2. **Mic permission via onboarding:** the first-run Activity requests
   RECORD_AUDIO and runs the enable-keyboard walkthrough; until granted, the
   keyboard shows a "finish setup" state that deep-links into it (and to system
   settings if denied).
3. Tap mic → record (live waveform in the preview strip) → tap again or silence
   timeout → transcribe → preview strip shows the text → tap to insert
   (`commitText`), or edit/cancel. Auto-insert 1.5s after the transcript preview
   renders (configurable) — anchored to transcript-ready, never to the tap.
4. Model manager: base.en default, small.en/medium.en optional download,
   sha256-verified, switchable, deletable.
5. Polish layer v0 (rule-based, always available, zero dependencies):
   capitalization at sentence start, smart spacing, filler removal toggle
   ("um", "uh", repeated words).
6. Settings: model choice, filler toggle, silence timeout, vocabulary editor.
7. Offline proof: airplane-mode integration test in CI.

### Polish layer (staged plan)

Positioning vs Gboard: we do not out-stream Google's recognizer. We win on
provable locality, custom vocabulary accuracy, and edited output.

- **Stage 0 (v1):** rules. whisper.cpp emits punctuation already; rules add
  capitalization, spacing, filler strip. Always works, every device, zero cost.
- **Stage 1 (v1.x):** system on-device AI via AICore (Gemini Nano), when
  available on the device (Pixel 8 and newer carry it):
  - ML Kit GenAI **Proofreading** with `InputType.VOICE` (grammar, homophones,
    punctuation; tuned for voice input).
  - ML Kit GenAI **Prompt API** for strict cleanup: remove fillers, accidental
    repeats, and self-taken-back words; fix punctuation; do not paraphrase or
    add facts. Short prompt, chunked input.
  - Zero model weight in our APK; inference on-device; must be probed with
    availability checks and graceful fallback to Stage 0.
  - Tradeoff to disclose in-app: the polish model is Google-managed (local
    inference, system-updated). Users who reject that use Stage 0 or 2.
- **Stage 2 (v2.x):** LiteRT-LM (`com.google.ai.edge.litertlm`) with a bundled
  Gemma cleanup model (or a small dictation-cleanup adapter) for devices
  without AICore and for user-owned weights end to end. (The MediaPipe LLM
  Inference API is maintenance-only; new work goes to LiteRT-LM.)
- **Context awareness (v1.x):** the IME can read text around the cursor
  (`getTextBeforeCursor`) and pass it as style/continuation context to the
  polish pass. Local only; this is the WisprFlow context feature without
  any accessibility permission.
- **M0 checklist additions:** verify ML Kit genai dependencies do not force a
  network permission into the manifest; probe AICore availability and quotas
  on the reference device; measure Stage 1 latency (target < 700ms for a short
  utterance) before promoting it to default.

## 6. v1.x / v2 roadmap

- v1.1: history (last 50 dictations, local only, clearable), export to file;
  AICore polish (Stage 1) behind availability probe.
- v1.2: multilingual model support, language picker; cursor-context polish.
- v2.0: optional Flow-Bubble overlay + Accessibility Service for
  dictate-anywhere without switching keyboards. Presented as an explicit
  opt-in with a permission explainer. This is the only feature that ever
  needs accessibility, and it stays optional.
- v2.x: LiteRT-LM Gemma polish bundle (Stage 2) for devices without AICore
  and for user-owned weights; widget trigger; Wear companion.

## 7. Distribution

- **Play (primary listing):** the data-safety form stays literally true —
  "no data collected." Expect the IME-category questionnaire and human review;
  the no-network manifest keeps answers simple. Prepare both in M0.
- **F-Droid (first-class from v1.0):** the audience that reads permission
  lists lives there; for this positioning it may out-convert Play. Requires
  reproducible builds, FOSS dependencies at build time (NNAPI/GPU delegates
  remain optional), and metadata drafted in M0.

## 8. Testing and benchmarks

- Latency harness: end-to-end (tap → text committed) on the three device tiers
  (flagship = newest Pixel Pro, reference = Pixel 8 Pro, one older floor
  device, informational). Target: < 2.0s for a 10-second utterance on the
  reference device with base.en int8 + delegate active; CPU-only numbers are
  recorded and published as the slow path.
- WER quickcheck: bundled 20-sample local set, per-model, run in the debug
  benchmark activity; log results, never upload.
- Battery (**measure, then publish — never promise**): 30-minute continuous
  dictation session on the reference device; record the real drain in the
  benchmark activity and put the actual number in the listing.

## 9. Risks

| Risk | Mitigation |
| --- | --- |
| whisper.cpp too slow on low-end devices | base.en int8 default; Bark 3.0 spike; "processing" is a designed-for state with cancel, never a spinner-of-death |
| Bark 3.0 license blocks use | whisper.cpp baseline ships regardless; Bark stays a spike |
| IME switching friction (users won't switch keyboards) | decided form factor with one-tap switch-back key; listing teaches the two-keyboard workflow; v2 bubble for power users |
| Mic prompt cannot be requested from the IME (no Activity context) | first-run onboarding Activity owns RECORD_AUDIO; deep-link fallback to settings |
| AICore unavailable or quota-limited on device | Stage 0 rules always ship; Stage 2 LiteRT-LM bundle covers the rest |
| ML Kit genai deps add network permission silently | M0 manifest check; drop the dependency if violated and rely on Stages 0/2 |
| Play policy on IMEs (data-safety forms, IME questionnaire) | no-network manifest keeps the form simple: "no data collected" is literally true; questionnaire prepped in M0 |
| Audio focus conflicts (media playing) | duck audio during recording like Woof's "Muzzle" (AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK) |

## 10. Milestones

- M0 (spike week): whisper.cpp binding runs on device; Bark 3.0 license +
  runtime feasibility answered; go/no-go on default engine. **Added:** verify
  the IME mic-permission approach on a real device; F-Droid reproducible-build
  check + metadata draft; Play IME questionnaire prep.
- M1 (weekend 1-2): IME shell + onboarding Activity + switch-back key +
  record → transcribe → commitText path with base.en. Dogfood-ready.
- M2: model manager, polish layer, vocabulary, settings.
- M3: benchmark pass, airplane test, Play + F-Droid listing copy
  ("no data collected"), release v1.0.

## 11. What this is not

- Not a general AI assistant, not a chat app, not a cloud product. If a
  feature requires the network, it is out of scope for v1 and must be
  optional and disclosed in v2+.
- Not a typing-keyboard replacement: no glide typing, no prediction parity.
  That is a permanent design boundary, not a v1 limitation.
