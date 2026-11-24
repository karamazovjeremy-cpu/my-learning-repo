# LearnHarness Mobile — Reverse-Engineering & Porting Plan for Poco M5

## 1. Reverse-Engineered Upstream Architecture (`amosblomqvist/learnharness`)
Based on [amosblomqvist/learnharness](https://github.com/amosblomqvist/learnharness) ([architecture.md](https://github.com/amosblomqvist/learnharness/blob/main/docs/architecture.md), [development.md](https://github.com/amosblomqvist/learnharness/blob/main/docs/development.md), [getting-started.md](https://github.com/amosblomqvist/learnharness/blob/main/docs/getting-started.md), [system-prompt.md](https://github.com/amosblomqvist/learnharness/blob/main/src/agent/system-prompt.md), and [package.json](https://github.com/amosblomqvist/learnharness/blob/main/package.json)):

| Desktop Subsystem | Upstream Implementation | Poco M5 Mobile Port |
| --- | --- | --- |
| **Desktop Host (`electron/`)** | Electron 44 + standalone Next.js 16 server on loopback | Lightweight Kotlin Android WebView shell (`MainActivity.kt`, ~18 MB APK) |
| **Agent Engine (`src/agent/`)** | `@earendil-works/pi-agent-core` + API keys / CLI | **Gemini App Bridge** (`GeminiBridge.kt` via `Intent.ACTION_SEND` -> `com.google.android.apps.bard` + auto-clipboard import on `onResume`), plus optional Direct API key & Offline Simulator |
| **Pedagogical Loop (`Prober / Teacher / Practitioner`)** | Server-owned turn runner + SSE (`/api/chats/:id/stream`) | On-device state machine + structured JSON tool packet protocol (`probe`, `recall`, `plan_init`, `quiz`, `explain`, `problem`, `conclude`) |
| **Thinking Fields & Dictation** | Thinking textarea + Deepgram API key | Thinking textarea + Confidence slider + native Android `SpeechRecognizer` (zero API cost) |
| **Knowledge Graph (`graph/*.jsonl`)** | Canvas 2D / d3-zoom + local JSONL files | Touch-native Canvas 2D + Chain Table + `localStorage` & JSONL export (`nodes.jsonl`, `chains.jsonl`) |

## 2. Poco M5 Hardware & Xiaomi HyperOS Constraints
Target specs ([Xiaomi Poco M5 Specs](https://www.po.co/global/product/poco-m5/specs/), [Wikipedia](https://en.wikipedia.org/wiki/Poco_M5)):
- **SoC**: MediaTek Helio G99 (6 nm, 2x 2.2 GHz Cortex-A76 + 6x 2.0 GHz Cortex-A55), Mali-G57 MC2 GPU
- **RAM / Storage**: 4 GB or 6 GB LPDDR4X RAM, 64 GB / 128 GB UFS 2.2
- **Display / OS**: 6.58" 1080x2408 LCD, Android 12 -> Android 14 (Xiaomi HyperOS)

Key mobile optimizations implemented in this port:
1. **No Electron / Node.js / Puppeteer**: Eliminates 500+ MB RAM footprint that Xiaomi HyperOS aggressively kills on 4–6 GB devices.
2. **Canvas 2D Graph (Zero WebGL/Rive)**: Smooth pan/zoom on Mali-G57 MC2 without GPU context loss.
3. **HyperOS Split-Screen & Floating Window Safe**: `android:configChanges="orientation|screenSize|smallestScreenSize|screenLayout|keyboardHidden"` in `AndroidManifest.xml` prevents WebView reloads when resizing into HyperOS Split-Screen alongside the Gemini app.

## 3. Why Gemini App Bridge Works Best for Jio Google AI Pro Users
- The [Jio Google Gemini Pro offer](https://www.jio.com/jcms/en-in/jio-google-gemini-terms-and-conditions/) provides Google AI Pro access inside the consumer **Gemini mobile app / web**, not paid Gemini API billing.
- On Android, the official Gemini app (`com.google.android.apps.bard`) does **not** expose a public background IPC or silent prompt/response API to third-party apps ([StackOverflow](https://stackoverflow.com/questions/78635448/how-to-launch-android-gemini-app-with-an-initial-query)).
- **Solution**: `GeminiBridge.kt` compiles each LearnHarness turn into a strict JSON-schema prompt packet, copies it to the clipboard, and launches `Intent.ACTION_SEND` directly to `com.google.android.apps.bard` (falling back to Android Sharesheet). When you copy Gemini's JSON reply and switch back to LearnHarness, `MainActivity.onResume()` automatically detects the JSON payload in your clipboard and notifies the UI!

## 4. How to Build the APK for Your Poco M5

### Option A — No PC Needed (GitHub Actions)
1. Create a new GitHub repository and upload the contents of this folder.
2. Open the **Actions** tab and run **Build LearnHarness Poco M5 APK** (`.github/workflows/build-apk.yml`).
3. Download the `LearnHarness-PocoM5-debug` artifact, unzip it on your Poco M5, and install the APK.

### Option B — Android Studio (PC)
1. Open this folder in Android Studio (JDK 17, Android SDK 34).
2. Connect your Poco M5 via USB debugging (or click **Build -> Build Bundle(s) / APK(s) -> Build APK(s)**).
3. Install `app/build/outputs/apk/debug/app-debug.apk` on your Poco M5.
4. In Xiaomi HyperOS: **Settings -> Apps -> LearnHarness -> Battery saver -> No restrictions** so your active session stays alive while switching to Gemini.
