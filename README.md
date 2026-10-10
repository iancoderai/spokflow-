# SpokFlow

Speak freely. SpokFlow turns your voice into ideas, information and practical
checklists. Speech stays on the phone. New recordings require internet access to
verify the 3-day free trial or a store purchase; existing notes remain available
offline.

## Requirements

Node.js **22.13 or newer**, as required by [Expo SDK 57](https://docs.expo.dev/versions/latest/).

JDK **17 or 21**, compatible with the generated Gradle/Android toolchain. Prebuild pins Gradle to
`C:/Program Files/Eclipse Adoptium/jdk-17.0.20.101-hotspot` when it exists; on
another machine set `SPOKFLOW_JDK` to your JDK 17/21 folder (a wrong path fails
the prebuild).

Android libraries inherit the project's NDK version, including SQLCipher's
SQLite module.

## Build and install (Android)

```bash
npm ci                         # exact versions and hashes from package-lock.json
npm run fetch-model            # once: verifies Whisper (264 MB) + local summary model (1.67 GB)
adb devices -l                 # serial of a connected phone, USB debugging on
npm run android -- --device
```

`npm run android` checks the release configuration and the model, prebuilds,
assembles the **release** APK signed with your upload key, verifies it, then
installs it. The APK carries the Hermes bundle and the model, so the installed
app never talks to this computer again — no Metro, no Wi-Fi. It needs the real
store keys and legal details (Release configuration) and your upload key
(Release signing); for a quick test on your own phone see Local test builds.

Do not run `npx expo run:android --device` directly: it installs a *debug*
build under the same application ID (`com.spokflow.app`) and replaces your
standalone install with one that needs Metro on the PC.

Other entry points:

| Command | Does |
| --- | --- |
| `npm run apk` | Rebuild the release APK without a device |
| `npm run install-apk` | Verify + install the existing APK (no Gradle) |
| `npm run verify-apk` | Check the expected signing fingerprint, manifest, permissions/exports, backup rules, shipped config, bytecode, native libraries and model hash |
| `npm run open:android` | Launch the installed app over ADB |
| `npm run android:dev` + `npm start` | Metro dev loop — tethered to this laptop |
| `npm run ios` / `npm run ios:dev` | Release / dev build on macOS |

Keep the same Android signing key and install over the current app to
preserve local notes. Never uninstall or clear app storage — that deletes them.
Android refuses an update signed with a different key: a phone holding a
debug-signed build (every build made before release signing existed) cannot
take an upload-signed one without uninstalling, which deletes its notes. Keep
using test builds on that phone, or export an encrypted backup before reinstalling.

### Release configuration

`app.json` → `expo.extra` holds the RevenueCat **public** SDK keys
(`revenueCat.ios` starts with `appl_`, `revenueCat.android` with `goog_`) and
the publisher details the legal texts show (`legal.controller`, `legal.email`,
`legal.address`). `npm run apk`, `npm run ios` and EAS builds first run
`scripts/check-release-config.cjs`, which refuses to build while a required value
is missing or blank, a `REPLACE_` placeholder is left, an SDK key or contact email
has an invalid format, or any value is a RevenueCat
**secret** key (`sk_…`; if one was ever committed, revoke it in the dashboard).
`npm run verify-apk` repeats these checks on the config
packaged inside the APK.

### Release signing

Release builds are signed with your upload key and stop without it; they never
fall back to the public debug key. Create the key once, then back it up with its
passwords outside the repository — losing it means no more updates:

```bash
keytool -genkeypair -v -keystore spokflow-upload.jks -alias upload -keyalg RSA -keysize 2048 -validity 10000
```

Add to `~/.gradle/gradle.properties` (forward slashes), or set as environment
variables:

```properties
SPOKFLOW_UPLOAD_STORE_FILE=C:/Users/you/keys/spokflow-upload.jks
SPOKFLOW_UPLOAD_STORE_PASSWORD=...
SPOKFLOW_UPLOAD_KEY_ALIAS=upload
SPOKFLOW_UPLOAD_KEY_PASSWORD=...
```

On Google Play, enrol in Play App Signing so Google holds the app signing key
and this key only uploads. Set `SPOKFLOW_SIGNING_SHA256` to the independently verified
SHA-256 fingerprint of the release/upload certificate. For an existing published app,
use its existing key; do not generate a replacement through the example above.

### Local test builds

`SPOKFLOW_TEST_BUILD=1` lets a local build keep the placeholders and sign with
the debug key, for your own phone only. Every check prints
`TEST BUILD ONLY - do not distribute`: anyone could re-sign such an APK, and it
sells nothing (no store keys). EAS builds refuse test signing.

```powershell
$env:SPOKFLOW_TEST_BUILD = "1"; npm run android -- --device
Remove-Item Env:SPOKFLOW_TEST_BUILD
```

### Building elsewhere

`eas.json` has a standalone `preview` APK profile:

```bash
npx eas-cli build --platform android --profile preview
```

EAS checks the release configuration, then downloads and verifies the model at
build time. It signs with the credentials it manages (`npx eas-cli credentials`):
give it your existing upload key rather than a new one, or Android rejects the
update. EAS Android uses the required `verifySpokflowRelease` Gradle task, which
builds and verifies artifacts before EAS can upload the application archive. Production builds
produce an AAB and a companion APK from the same release variant for verification.
The expected certificate fingerprint must also be configured in EAS.
The AAB signature check trusts only that pinned public certificate and uses
strict JDK verification to reject unsigned additions to an otherwise signed archive.

## How it works

- **Capture** — `@fugood/react-native-audio-pcm-stream`, 16 kHz mono PCM, kept
  in memory: no audio file is written.
- **Transcription** — `whisper.rn` runs multilingual Whisper Small Q8_0 from the
  [whisper.cpp model repo](https://huggingface.co/ggerganov/whisper.cpp/tree/5359861c739e955e79d9a303bcbc70fb988958b1),
  pinned to that commit and embedded as a native APK asset. The download uses
  HTTPS to Hugging Face hosts only, redirects included. Size and SHA-256 are
  verified at download and prebuild; a truncated or substituted model fails the
  build.
- **Language** — French and English share one model. Detection selects the
  language automatically, keeping the first successful language for the note.
  Unusable results are retried in the other supported language; a language
  already returned by detection is not decoded again for the same batch.
  Failed recognition is not saved as a note. No translation.
- **Accuracy** — every pass uses the model's full audio context and the original
  PCM, including quiet words and pauses. Shortened context and amplitude-based
  silence removal are disabled; short notes can take longer to process.
  Pause batches wait for at least 12 seconds of speech context, reducing short
  passes. Vocabulary hints and up to 400 trailing characters from this note
  guide the fast decoder. Saved transcripts share guarded clock-time formatting;
  word substitutions, negations and missing list items require recognition from
  audio and are not guessed by text cleanup.
- **Scheduling** — the model loads on the first non-silent batch, not at start-up.
  Audio uses 30-second windows, cut at the quietest point in the final six
  seconds, or a speaker pause, with the remainder finished on stop. Only the
  next window is assembled; queued buffers stay in place until consumed.
  Digital silence skips inference, including a zero-only tail inside the last
  capture buffer. The context is released when the app backgrounds;
  failed transcription or checkpoint writes stop capture. Microphone events target
  100 ms buffers. Recordings are capped at 15 minutes and queued audio at 90 seconds.
- **Idea summaries** — `src/lib/summaryEngine.ts` runs the multilingual
  [Qwen3 4B Instruct 2507, IQ3_XXS](https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF) through `llama.rn`, on this phone only. Its model
  file is pinned by revision, byte count and SHA-256, bundled alongside Whisper,
  and verified in release artifacts. Android copies it to private cache once
  (about 1.67 GB of additional disk space); iOS uses the bundled resource.
  Whisper is released before text inference, and text contexts are released
  after each summary or cancellation. Pitches such as Second Chance Food stay
  ideas through their hypothetical examples and launch plans. English is the
  default summary language; Settings also offers French and mixed French/English.
  Mixed mode copies the substantive source sentences directly, keeping their
  French/English switches without text inference. Short bold headings,
  bullets, bold numeric figures and numbered sequences appear in the note.
  The internal prompt follows source order, preserves terms, names, figures,
  examples and every idea, removes filler and forbids outside facts or advice.
  Each source sentence is translated separately before its paragraph is assembled.
  A separate local review checks each point against its source using exact source quotes, without rewriting
  accepted text. Rejected text is regenerated once and checked again; corrections
  compare the actual draft and repaired text. A fixed JSON schema requires a
  translation for every substantive source sentence, with paragraph headings.
  Deterministic checks reject missing source entries,
  changed or missing figures, common spoken quantities and units, obvious wrong-language output and
  incomplete JSON. Long sources are processed in token-sized batches without
  truncation. If generation or verification fails, all source text is retained
  and the note explains that it is not a verified summary. Regenerate summary
  rebuilds older notes using the saved transcript, without overwriting concurrent
  edits. These checks do not prove semantic accuracy; review important details.
  Local model latency and memory use still require device measurements.
  Keep transcription stops a long summary without losing the saved source.
- **Extraction** — `src/lib/checklistEngine.ts` splits dictation into tasks,
  ideas and information entries offline, and titles the note from its contents.
  An announced idea stays an idea through its action sentences; an explicit
  list heading switches back to tasks. The extractive idea fallback keeps all
  sentences instead of stopping at four. Generated titles are capped
  at eight words / 60 characters. Task destinations retain their verbs.
  English/French task extraction shares one model-independent pass: acknowledgments
  and unfinished fragments stay off checkboxes, related instructions share topic
  headings, and supported corrections replace times, retract cancelled tasks,
  remove list items, and retain conditions, deadlines and safety constraints.
  Independent actions keep separate checkboxes; interrupted sentences are joined.
  Prohibitions and no-op branches never become rows; available alternatives and
  conditional exceptions become positive tasks. Completed items are removed,
  groceries retain their quantities without verbs, and deferred work keeps its date.
  Shared timing and priorities appear once in the topic heading. Numbered or
  Markdown sections keep their dictated order. Supported relative dates and
  clock times order inferred topic sections. Evidence precedes conditional issue
  creation. A spoken "wait" keeps the candidate available for a following
  correction before saving.
  The full tomorrow-plan fixture checks all 49 actions and three dependency notes,
  including verified push before branch creation, both onboarding themes, all
  three meeting questions, and both approval outcomes. Run
  `npm run check-checklist -- --show-plan` to inspect its generated list.
  Headings are stored in the existing checklist JSON; older notes remain compatible.
  This remains deterministic cleanup: unclear names, numbers or misheard words
  are kept in the transcript rather than guessed. It does not guarantee semantic
  accuracy for arbitrary speech. Run `npm run check-checklist` for bilingual
  correction/grouping checks; native speech accuracy still needs device recordings.
- **Storage** — an encrypted SQLCipher database (`spokflow.db`) whose random
  256-bit key stays in the Android Keystore / iOS Keychain, on this device only.
  Deletion uses `secure_delete` and a WAL checkpoint; flash storage cannot guarantee
  physical erasure. Notes are
  excluded from cloud backup and phone-to-phone transfer, and iOS keeps them
  unreadable while the phone is locked. A plaintext `dotnote.db` from an older
  version is copied, checkpointed, cleared and deleted; migration failures and
  conflicting records are surfaced. Missing keys preserve existing database files.
  Settings offers passphrase-encrypted backup/export and restore for up to
  10,000 notes and 100 MB. Export validates restore compatibility; oversized or
  damaged legacy records are kept but need separate recovery. Export before
  reinstalling.

### Native patches

Both run on `postinstall` and before Android builds; both are idempotent and
fail if an upstream change makes a patch unsafe. Each was reviewed against one
version (`@fugood/react-native-audio-pcm-stream` 1.1.4, `whisper.rn` 0.7.2) and
stops on any other: review the patch again before upgrading. A JS-only reload
cannot apply them — rebuild natively.

- `scripts/fix-audio-capture.cjs` — awaits native start-up, propagates capture
  errors, removes listeners, keeps Android's first buffers, flushes iOS input
  before stopping, and passes decoded PCM buffers without a second copy.
- `scripts/fix-whisper-decoder.cjs` — bilingual detection and failed-decoder
  guards; reuses the first window's encoder result from language detection when
  offset and context are unchanged. CPU workers are reused across decoding
  graphs, sleep between graphs instead of polling, and are freed with the model.
  Android loads a matching dot-product / i8mm CPU variant when supported.
  Recovery is capped at two temperatures with one candidate each (previously
  six temperatures and two candidates), and the final attempt must pass the
  log-probability guard. An initial acronym vocabulary includes ATM; a nonzero
  text-context budget enables it. Model weights and full audio context stay intact.
  The silence check computes only the probability it reads, using the unchanged
  log-softmax. Native jobs move their audio through the worker queue instead of
  copying it at each handoff.

Completed live batches are persisted as encrypted checkpoint notes. Finalization
reuses the checkpoint ID. Force termination can still lose audio that has not been
decoded; raw audio is never persisted. No on-device speedup has been measured.

## Security

- **Screens** — notes stay hidden until native capture and app-switcher
  protection initializes. Failed setup offers retry; verify behavior on devices.
- **Links** — every `spokflow://` link or outside intent opens the home screen
  only. Notes, the paywall and settings cannot be opened from outside.
- **Purchases** — each new recording requires a newly signed server response,
  newer than the SDK baseline and persisted last authorization. Cache dates in
  the future are rejected; an advancing clock enforces expiry during recording.
- **Trial** — 3 days, no card, from RevenueCat's signed `firstSeen` date, so
  the device clock can't extend it. A reinstall gets a new anonymous ID and a
  new trial. Store introductory offers still show on the paywall.

These checks run on the phone. A rooted phone or a repackaged app can still get
around them. Server-side authorization protects server operations, but attestation
cannot make an offline executable unpatchable. This app has no paid server operation.

## Development

```bash
npm run typecheck
npm run lint
npm run check-checklist
npm run check-summary
npm run check-slicing
npm run check-transcription
```

The transcription checks mock native inference and verify unchanged PCM,
sub-second padding, queued/stop buffers, bilingual retries, faults and cancellation.
They do not measure speech recognition accuracy, phone latency or battery use.
Summary checks mock native inference to verify classification, source coverage,
quantity/unit checks, repair, rejection, cancellation and storage. For a real
content check, start `llama-server` with the pinned summary GGUF, a 4096-token
context, JSON/Jinja support and one slot on `127.0.0.1:8183`, then run
`npm run check-summary-model`. It exercises the app's pipeline against the full
Second Chance Food pitch in English and its bakery example in French and mixed
mode. It writes drafts, review decisions and results to the system temporary
directory. Inspect the text against the source; an automated pass is not proof
of semantic accuracy, and desktop timings do not predict phone performance.
Summary parsing caches live only during extraction and are cleared afterwards.
To compare summary content and desktop timings against a copy made before editing:

```bash
npm run check-checklist -- --compare /path/to/original/checklistEngine.ts --benchmark
```

All comparison fixtures must retain identical titles and item content (random
item IDs are excluded). The benchmark uses short plans and generated long English,
French and quoted notes; these timings do not represent native inference.

Postinstall also adapts `query-string` 7.1.3 to the fixed `decode-uri-component`
0.5.0 default export, and anchors Expo CLI 57.0.27's prebuild rename to the root
`app.json`, preventing a recursive scan and modification of dependency copies.
Both adapters stop on an unreviewed upstream version.
Before release, install the rebuilt APK
on the target phone and dictate short French/English phrases (including "go to
the cash machine" and "aller à l'ATM"), plus a longer note with pauses.
Compare time from stop to saved note and energy use against the previous build
on the same phone and recordings. Check the stored transcript as well
as the extracted tasks. The original failing audio is needed to confirm that
specific acoustic regression.

## Paywall

Weekly, monthly, yearly and lifetime offers use RevenueCat. Configure its public
SDK keys in `app.json` (Release configuration) and matching store products to
enable billing; prices always come from the store. Without a key a release
build refuses recording and shows no prices; release configuration checks reject
placeholder keys. Development builds without a key unlock Pro for testing.
