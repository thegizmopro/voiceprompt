# VoicePrompt Android — Specification

Local-first voice dictation for Android. The WisprFlow experience without the
WisprFlow tradeoffs: no cloud, no account, no accessibility service required,
microphone permission only. First target: v1.0-android MVP.

Lineage: macOS original (`voiceprompt.py`) → Windows port (`voiceprompt_windows.py`,
v4.2) → this. The record → transcribe → insert architecture and the custom
vocabulary (`vocabulary.txt`) carry forward.

## 1. Problem and positioning

Wispr Flow on Android works well but requires: an Accessibility Service
(read screen content + observe text), microphone access routed through cloud
processing, and an account/subscription. For users whose dealbreaker is
permissions and privacy, there is no local-first alternative today. Woof
(woof.talk) proved local dictation with a compact model is viable on desktop;
nothing equivalent exists on Android.

**Positioning:** the free, local, offline, minimal-permission dictation keyboard.
Not a WisprFlow clone: fewer permissions is the product.

## 2. Principles (non-negotiable)

1. **All inference on-device.** Zero network calls after model download. The
   release build must work in airplane mode (a CI test asserts it).
2. **Minimum permissions.** v1 requests exactly: RECORD_AUDIO. No accessibility
   service, no overlay, no storage, no contacts, no network permission at
   runtime (declare `android.permission.INTERNET` only for the model downloader,
   or split it into a download-time only module / first-run flow).
3. **No account, no telemetry, no analytics.** Crash reporting optional and
   off by default.
4. **User-owned models.** Models download on demand, live in app storage,
   deletable from settings. Ship no model in the APK beyond a stub.

## 3. Architecture

```
app/                    Kotlin, minSdk 29 (Android 10), target latest
  ime/                  InputMethodService: keyboard row + mic key + preview strip
  engine/               whisper.cpp JNI wrapper (libwhisper_jni.so), VAD, audio capture
  models/               Model manager: download, verify (sha256), select, delete
  polish/               Post-processing: punctuation, filler rules, spacing
  benchmark/            Debug-only activity: WER quickcheck + latency harness
```

**Why IME-first:** a keyboard can insert text into any focused field by design.
No overlay, no accessibility service, no special permissions beyond the mic,
and no Play Store policy friction (accessibility APIs now require justification
and human review). The tradeoff: users must switch to our keyboard to dictate.
v2 adds the optional floating bubble with an honest permission disclosure.

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

## 5. v1.0 scope (MVP)

1. IME keyboard: standard minimal QWERTY (or a thin dictation bar above the
   user's existing keyboard, whichever proves simpler) with a prominent mic key.
2. Tap mic → record (live waveform in the preview strip) → tap again or silence
   timeout → transcribe → preview strip shows the text → tap to insert
   (commitText), or edit/cancel. Auto-insert after 1.5s if untouched
   (configurable).
3. Model manager: base.en default, small.en/medium.en optional download,
   sha256-verified, switchable, deletable.
4. Polish layer v0 (rule-based, always available, zero dependencies):
   capitalization at sentence start, smart spacing, filler removal toggle
   ("um", "uh", repeated words).
5. Settings: model choice, filler toggle, silence timeout, vocabulary editor.
6. Offline proof: airplane-mode integration test in CI.

### Polish layer (staged plan)

Positioning vs Gboard: we do not out-stream Google's recognizer. We win on
provable locality, custom vocabulary accuracy, and edited output.

- **Stage 0 (v1):** rules. whisper.cpp emits punctuation already; rules add
  capitalization, spacing, filler strip. Always works, every device, zero cost.
- **Stage 1 (v1.x):** system on-device AI via AICore (Gemini Nano), when
  available on the device:
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
  (getTextBeforeCursor) and pass it as style/continuation context to the
  polish pass. Local only; this is the WisprFlow context feature without
  any accessibility permission.
- **M0 checklist additions:** verify ML Kit genai dependencies do not force a
  network permission into the manifest; probe AICore availability and quotas
  on the target device; measure Stage 1 latency (target < 700ms for a short
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

## 7. Testing and benchmarks

- Latency harness: end-to-end (tap → text committed) on three device tiers
  (flagship, mid, old). Target: < 2.0s for a 10-second utterance on mid-tier
  with base.en int8.
- WER quickcheck: bundled 20-sample local set, per-model, run in the debug
  benchmark activity; log results, never upload.
- Battery: 30-minute continuous dictation session, < 8% battery on mid-tier.

## 8. Risks

| Risk | Mitigation |
| --- | --- |
| whisper.cpp too slow on low-end devices | base.en int8 default; Bark 3.0 spike; degrade gracefully with a "processing" state |
| Bark 3.0 license blocks use | whisper.cpp baseline ships regardless; Bark stays a spike |
| IME switching friction (users won't switch keyboards) | thin dictation-bar variant; v2 bubble for power users |
| AICore unavailable or quota-limited on device | Stage 0 rules always ship; Stage 2 LiteRT-LM bundle covers the rest |
| ML Kit genai deps add network permission silently | M0 manifest check; drop the dependency if violated and rely on Stages 0/2 |
| Play policy on IMEs (data-safety forms) | no-network manifest keeps the form simple: "no data collected" is literally true |
| Audio focus conflicts (media playing) | duck audio during recording like Woof's "Muzzle" (AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK) |

## 9. Milestones

- M0 (spike week): whisper.cpp binding runs on device; Bark 3.0 license +
  runtime feasibility answered; go/no-go on default engine.
- M1 (weekend 1-2): IME shell + record → transcribe → commitText path with
  base.en. Dogfood-ready.
- M2: model manager, polish layer, vocabulary, settings.
- M3: benchmark pass, airplane test, Play listing copy ("no data collected"),
  release v1.0.

## 10. What this is not

- Not a general AI assistant, not a chat app, not a cloud product. If a
  feature requires the network, it is out of scope for v1 and must be
  optional and disclosed in v2+.
