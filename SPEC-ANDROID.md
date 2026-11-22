# VoicePrompt Android — Specification (v3)

Local-first voice dictation for Android: a keyboard that cannot phone home.
Built on-device (whisper.cpp), zero INTERNET permission in release, no account,
no accessibility service. First target: v1.0-android MVP.

Lineage: macOS original (`voiceprompt.py`) → Windows port (`voiceprompt_windows.py`, v4.2) → this.
The record → transcribe → insert architecture and the custom vocabulary (`vocabulary.txt`) carry forward.
Spec history: v1 by Arty (`afcd5b3` + `b930984`) → v2 review pass (Myles, `a3393dc`) → **v3: adversarial review folded in (Claude Code, 2026-11-21; verdict was REVISE-FIRST).**

> **v3 changelog (all changes trace to the adversarial review; platform facts marked [probe] are M0 measurements, not spec assertions):**
> 1. Engine reality: every NNAPI mention deleted (TFLite-only, deprecated in Android 15; ggml has no NNAPI backend). Targets are CPU-only. "GGUF" corrected to ggml quantized (`q8_0`/`q5_1`). Whisper's padded 30s encoder window is now a stated cost fact; GPU (Vulkan/OpenCL, Adreno-tuned) demoted to post-v1 experiment — Mali (the reference device) is its worst case.
> 2. Network: `base.en` (~80 MB q8_0) is bundled in the APK and the release build ships with **no `INTERNET` permission at all**. Larger models arrive by SAF file import or an optional separate downloader flavor. "Works in airplane mode" is replaced by two harder checks: a build-time manifest assertion and a runtime sockets-closed test with networking on.
> 3. Distribution: two product flavors from day one (`foss` / `play`). F-Droid facts corrected: reproducible builds are optional (signature-keeping), not required; the real FOSS blockers are ML Kit GenAI artifacts, prebuilt ONNX AARs, and gated Gemma weights. ONNX Runtime dropped entirely.
> 4. Market: §1 rewritten — offline dictation keyboards already exist (FUTO Voice Input, Transcribro, Sayboard). Our edges: custom vocabulary, polished output, zero-INTERNET provability, and being the recognizer other keyboards call (RecognitionService + voice subtype).
> 5. Insertion: dictation renders as **composing text** in the target field (platform idiom); the preview-strip auto-insert race is eliminated by design. Stage 1 polish may never auto-insert; acceptance guard added.
> 6. Failure modes: error-state table added (silence hallucination, silenced mic during calls, revoked permission, corrupt model, storage full, process death). VAD pre-check before inference.
> 7. Native: engine runs in a separate `:engine` process (a crash costs one dictation, not the keyboard); 16 KB page alignment enforced; single runtime (ggml Silero VAD); `medium.en` dropped from v1.
> 8. Privacy: `allowBackup` off; mic disabled in password fields; incognito flag respected; cursor context treated as untrusted input; crash logs local and user-exported only.
> 9. M0 is now a quantitative gate with thresholds, fallbacks, and kill criteria; a mid-range device joins the matrix; the Bark spike moved post-v1; M1 loses silence-timeout (tap-to-stop).

## 1. Problem and positioning

Offline dictation keyboards already exist on Android: **FUTO Voice Input / FUTO
Keyboard** (whisper, offline, paid-adjacent ecosystem), **Transcribro**
(whisper.cpp, offline, minimal permissions), **Sayboard** (Vosk, on F-Droid),
plus Gboard itself does on-device voice typing on the reference device.
"Fewer permissions than WisprFlow" is not an open field.

**What none of them offer together — our actual edges:**
1. **Zero-INTERNET provability:** the release manifest has no network
   permission at all. Not "works offline" — *cannot* transmit.
2. **Custom vocabulary:** names, places, and fleet terms bias transcription
   (capped at ~150 prompt tokens, editor with a counter).
3. **Polished output:** fillers stripped, repeats removed, dictation that
   looks typed — with a deterministic guard that polish never changes meaning.
4. **A recognizer other keyboards can call:** we register a
   `RecognitionService` and declare a voice IME subtype, so HeliBoard,
   FlorisBoard, and any `SpeechRecognizer` client can hand dictation to us
   and get text back. The minimal built-in QWERTY is a fallback, not the plan.

**Two-keyboard honesty (unchanged from v2):** switching *to* VoicePrompt is the
keyboard picker (globe key or nav-bar switcher); switching *back* is one tap
(`switchToPreviousInputMethod`). v2's optional bubble is where dictation and
any keyboard coexist.

**M0 competitive teardown (1 day):** Transcribro and FUTO installed, used,
measured (latency, WER on our samples, permission list). We spec against the
real thing, not the imagined one.

## 2. Principles (non-negotiable)

1. **All inference on-device; no network capability in release.** The release
   manifest contains no `INTERNET` permission (CI asserts it), and a runtime
   test with networking enabled fails if any socket opens.
2. **Minimum permissions.** Exactly: `RECORD_AUDIO`. No accessibility service,
   no overlay, no storage permission (SAF for imports). The RECORD_AUDIO
   prompt is owned by the first-run onboarding Activity, because an
   InputMethodService has no Activity context (§5).
3. **No account, no telemetry, no analytics.** Crash reporting is a local,
   user-exported log (the Windows port's forensics model) — nothing leaves
   the device, ever, including crashes.
4. **User-owned models.** `base.en` ships in the APK (the zero-network
   guarantee requires it); larger models are imported by the user via SAF
   from sources we document, sha256-verified at import **and at every load**.

## 3. Architecture

Two product flavors, from day one:

- `foss`: no ML Kit, no prebuilt AARs, no Google-restricted artifacts.
  This is the F-Droid build and the default.
- `play`: may include Stage 1 polish (ML Kit GenAI) behind an availability
  probe and in-app disclosure.

```
app/                      Kotlin, minSdk 29, target latest, NDK r28+ (16 KB aligned)
  ime/                    InputMethodService: mic key, waveform, composing-text flow
  onboarding/             First-run Activity: RECORD_AUDIO, enable-keyboard walkthrough
                         (incl. copy designed for Android's "may collect all text you
                         type" IME warning; state re-checked via
                         InputMethodManager.getEnabledInputMethodList())
  engine/                 :engine process (bound service): whisper.cpp JNI, VAD,
                         audio capture — isolated so a native crash costs one
                         dictation, not the keyboard
  models/                 Bundled base.en + SAF import, sha256 verify at import and
                         load, switchable, deletable
  polish/                 Stage 0 rules (always) + guarded polish (§6)
  recognize/              RecognitionService + voice IME subtype (§7)
  benchmark/              Debug-only: WER + latency harness (sustained-load mode)
```

## 4. Engine plan

- **Baseline: whisper.cpp (CPU-only).** ggml quantized `q8_0` default,
  `q5_1` candidate. NEON, big+mid-core threads. No NNAPI (does not exist for
  ggml; deprecated platform-wide). GPU (Vulkan/OpenCL) is a post-v1
  experiment: tuned for Adreno, and the reference device is Mali.
- **Cost fact:** whisper's encoder always processes a padded 30s window; a
  10s utterance costs nearly a 30s one. `audio_ctx` reduction is a measured
  M0 experiment (latency vs WER), not an assumption.
- **Models in v1:** `tiny.en` / `base.en` / `small.en` only. `medium.en` is out
  (RAM pressure on an IME process makes it a liability).
- **VAD:** whisper.cpp's built-in Silero VAD (ggml — no second runtime;
  [probe] against the pinned whisper.cpp version). VAD gates inference: no
  speech detected → "didn't catch that," no transcription attempt.
- **Custom vocabulary:** `vocabulary.txt` port as initial-prompt/glossary,
  capped at ~150 tokens (whisper prompt ceiling ~224; longer prompts leak
  into output). Editor shows the token count.
- **Bark 3.0 spike: moved post-v1.** If revisited: pin the exact HF repo +
  commit, license review, ONNX/ExecuTorch conversion, mid-range RTF gate.
- **Reference devices:** Pixel 8 Pro (reference), newest Pixel Pro
  (flagship), **Galaxy A5x-class mid-range (gating)**, one older device
  (informational).
- **Cold-start policy:** prewarm on `onStartInputView`; unload after idle.
  Sustained-load (10 min) latency is reported alongside cold numbers —
  Tensor G3 throttles.

## 5. v1.0 scope (MVP)

1. **Dictation as composing text (decided).** Tap mic → record (waveform) →
   tap-to-stop (silence-timeout is v1.1) → transcript lands in the target
   field as composing text (`setComposingText`); backspace or cancel clears
   it; any keypress commits it. No separate preview strip, no auto-insert
   timer, no internal editor — the race class is gone by construction.
2. **Field awareness:** capture `EditorInfo` identity at record start; the mic
   key is disabled in password fields; dictation cancels cleanly on field
   change or keyboard switch. Incognito fields
   (`IME_FLAG_NO_PERSONALIZED_LEARNING`) never write history or vocabulary.
3. **Mic permission via onboarding Activity** (§3), with deep-links into
   settings when denied and enable-state re-checks after the system dialogs.
4. Model manager: bundled `base.en` default; SAF import for `tiny.en`/`small.en`;
   hash+size verified at import **and before every native load**; free-space
   pre-check ≥ 1.2× model size.
5. Polish Stage 0 (rules, always on): capitalization, smart spacing, filler
   removal toggle.
6. Settings: model choice, filler toggle, vocabulary editor (with token
   counter), crash-log export.
7. Offline proof (v3 form): release-manifest assertion (no `INTERNET`) +
   sockets-closed runtime test with networking on.

### Error-state table (v1 ships all of these)

| State | Detection | Behavior |
| --- | --- | --- |
| Mic permission denied/revoked | onCaptureFail, permission re-check | "Finish setup" card, deep-link to settings |
| Mic silenced (call in progress, privacy toggle) | `isClientSilenced()` + RMS floor | "Mic unavailable (in a call or muted?)" — never transcribes silence |
| Silence / noise only (whisper hallucination guard) | VAD pre-check + `no_speech_prob` | "Didn't catch that" |
| Corrupt model file | size+hash check at load | Refuse load, offer re-import; never hand bad mmap to native |
| Storage full | pre-check | Import blocked with explanation |
| Engine process death mid-transcribe | binder death callback | Restart engine, one lost dictation, keyboard lives |

## 6. Polish layer (staged, guarded)

Positioning: we win on provable locality, vocabulary accuracy, and **edited**
output — never on raw recognizer scale.

- **Stage 0 (v1):** deterministic rules. Always work, zero cost, no meaning risk.
- **Stage 1 (v1.x, `play` flavor only, conditional):** ML Kit GenAI
  Proofreading/Prompt via AICore. **Conditions:** M0 probe must call it
  *from inside the IME process while another app is focused* [probe:
  ML Kit GenAI may require a foreground caller, which an IME is not].
  A "blocked" or "device unsupported" result (support centers on Pixel 9+ /
  recent Galaxy [probe]; reference 8 Pro may not qualify; GrapheneOS users
  have no AICore at all) removes Stage 1 from v1.x entirely.
- **Acceptance guard (all polish, deterministic):** polished output is
  rejected and raw text shown if any negation or number changed, token edit
  distance exceeds a threshold, or length shifts beyond ±X%. **Polished text
  is never auto-committed unseen.**
- **Stage 2 (v2.x):** user-imported Gemma-class cleanup model via LiteRT-LM.
  Gemma weights are gated and non-OSI-licensed [probe exact terms] → the
  `foss` path is user import (SAF), never bundled; document sources. This is
  the stage this audience actually wants; it may be promoted ahead of
  Stage 1 if the M0 probe fails.
- **Context awareness (v1.x):** `getTextBeforeCursor` context is untrusted
  input: delimiter-wrapped, length-capped, guard-checked; null (WebViews,
  browsers, some Compose fields) is a normal case, not an error.

## 7. Integration (new)

- **RecognitionService:** VoicePrompt registers as a system speech
  recognizer, so any app using `SpeechRecognizer` can get local
  transcription. This is the distribution channel that skips the
  two-keyboard problem entirely.
- **Voice IME subtype** (`android:imeSubtypeMode="voice"`): HeliBoard,
  FlorisBoard, and other FOSS keyboards can hand off from their mic key and
  receive committed text. Being the FOSS ecosystem's dictation engine is a
  better position than being its 40th keyboard.
- **M0:** prove the handoff works (HeliBoard → VoicePrompt → `commitText` →
  return) on a real device.

## 8. Distribution

- **F-Droid (`foss`):** inclusion needs source-buildable everything (no ML
  Kit, no prebuilt AARs — both dropped from `foss`). Reproducible builds are
  an optional later decision (they let us keep our signature; native NDK
  builds are the hard part — budget separately). Anti-features declared
  honestly (none expected).
- **Play (`play`):** data-safety form stays literally true ("no data
  collected"). Real compliance work is 16 KB alignment + target-API; the
  Accessibility declaration only arrives with v2's bubble. Keep both
  channels' listing copy aligned.
- **Channel/signature note:** Play and F-Droid builds sign differently;
  switching channels means a reinstall. Settings and vocabulary export/import
  via SAF from day one so nothing is hostage to the channel.

## 9. Testing and benchmarks

- **Three separate latency metrics** (never one blended number):
  (a) stop-tap → transcript-ready (engine KPI), (b) commit latency after
  transcript, (c) any UI delay. Measured at 5/10/30/60s utterances, on the
  four-device matrix, cold and after 10 minutes sustained.
- **Targets (M0 gate, §11) are CPU-only numbers on the reference device.**
- **Incremental transcription (M1 architecture decision):** transcribe
  VAD-segmented chunks while the user is still talking, so finish latency is
  bounded by the last chunk, not the utterance length.
- **WER:** ≥ 100 utterances of the owner's real dictation (fleet vocabulary
  included), scored against Gboard offline on the same audio. The bar:
  within +3 points after vocabulary biasing.
- **Battery (measure, publish):** mAh per 100 dictations of 15s each, plus
  idle drain with the keyboard shown. Real number in the listing.

## 10. Risks

| Risk | Mitigation |
| --- | --- |
| CPU base.en misses latency target on mid-range | Pre-written fallback rule: tiny.en default / q5_1 (decided by M0 data, not hope) |
| ML Kit GenAI unusable from IME process | M0 probe from inside the IME; "blocked" deletes Stage 1; Stage 2 promoted instead |
| Two-keyboard friction caps adoption | RecognitionService + voice-subtype handoff (§7); QWERTY fallback; v2 bubble |
| IME enable-dialog drop-off | Onboarding copy written for the scary dialog; state re-checks; positioning leads with zero-INTERNET |
| Native crash kills keyboard | `:engine` process isolation |
| FOSS channel purity regressions | `foss` flavor CI: no proprietary deps, manifest assertion, 16 KB check |
| Play/F-Droid signature split confuses users | SAF export/import of settings from day one |
| Gboard on-device good enough for most | Compete where it cannot: zero-INTERNET, vocabulary, polish, FOSS integrations |

## 11. Milestones

**M0 — quantitative gate. Passes only if ALL hold; each has a stated fallback.**

1. **Engine (CPU-only, Pixel 8 Pro):** `base.en` q8_0 (or q5_1): stop →
   transcript ≤ 2.0s for a 10s clip, ≤ 4.0s for 30s; cold load ≤ 1.0s; peak
   RSS ≤ 300 MB; sustained (10 min) latency ≤ 1.5× cold. *Fallback:*
   tiny.en default, base.en opt-in.
2. **Mid-range gate (Galaxy A5x class):** ≤ 2× the reference numbers.
   *Fallback:* tiny.en default below a CPU-benchmark threshold.
3. **Accuracy:** WER on ≥ 100 own-voice samples ≤ Gboard offline + 3 points
   after vocabulary biasing. *Fallback:* reposition §1 (vocabulary-first).
4. **IME platform probes:** mic capture while another app is focused;
   silenced-capture detection during a call; `switchToPreviousInputMethod`;
   HeliBoard voice-subtype handoff.
5. **Stage 1 probe from the IME process:** works / blocked / unsupported.
   Blocked or unsupported → Stage 1 is out of v1.x.
6. **Build:** release manifest has no `INTERNET`; 16 KB alignment passes.
7. **Competitive teardown:** Transcribro + FUTO measured on the same harness.

- **M1 (weekend 1-2, trimmed):** IME shell + onboarding + composing-text flow
  + engine service + tap-to-stop. (Silence-timeout moved to v1.1; it needs
  VAD tuning.) Dogfood-ready.
- **M2:** model manager (bundle + SAF import), Stage 0 polish, vocabulary
  editor, settings, error states.
- **M3:** benchmarks (three metrics, four devices, sustained mode), manifest
  + sockets tests, F-Droid metadata (`foss`), Play listing (`play`),
  release v1.0.

## 12. What this is not

- Not a general AI assistant, not a chat app, not a cloud product. In
  release, it is not even *capable* of the network.
- Not a typing-keyboard replacement: no glide, no prediction parity — a
  permanent design boundary. The QWERTY is a fallback.
- Not the first offline dictation keyboard (§1) — the attempt is to be the
  most provable, most vocabularied, and the one other keyboards call.
