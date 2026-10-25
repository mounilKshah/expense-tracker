To execute this project with your exact constraints—using a React PWA wrapped in Capacitor, minimizing Android Studio involvement, and strictly adhering to a sub-4GB RAM environment—we must fundamentally rethink the ML deployment strategy.

In a 4GB RAM Android device, the OS and system services consume roughly 1.5GB to 2GB. This leaves your app with a working memory limit of about **300MB to 500MB** before the Low Memory Killer Daemon (LMKD) abruptly terminates it. Running a standard 1B or 3B generative SLM (which requires 1GB+ RAM) inside a Capacitor WebView (which has its own Chromium memory overhead) will reliably crash the app.

Here is the Staff Engineer blueprint to build this locally, efficiently, and entirely within the web ecosystem (WASM/WebGPU) to bypass native Android development.

### The Tech Stack

1. **Frontend & UI:** React 18+ (Vite), TypeScript, TailwindCSS. Vite provides blazing-fast HMR for web testing.
2. **Mobile Bridge:** Ionic Capacitor. It allows you to build the app entirely as a PWA, test in your browser, and compile to an Android APK via the CLI (`npx cap build android`) without ever opening the Android Studio IDE.
3. **Local Database:** `@capacitor-community/sqlite`. This plugin provides full SQLite native support on the device but falls back to `sql.js` (WebAssembly SQLite) when running in your browser as a PWA.
4. **State Management:** Zustand. It is infinitely lighter than Redux and prevents unnecessary re-renders in the DOM, which is crucial for WebView performance.
5. **The Machine Learning Engine (WASM-based):**
Since we want to avoid native JNI/C++ compilation (llama.cpp) and stay within the JavaScript/PWA ecosystem under tight memory constraints, we will use **Transformers.js** (by Hugging Face). It runs ONNX models directly in the browser/WebView using WebAssembly (WASM) and WebGL/WebGPU.

---

### The < 4GB RAM Machine Learning Strategy

Instead of using a generative LLM (like Llama 3 or Qwen) to *generate* a JSON response, we will use an **Extractive NLP Model**—specifically a quantized Token Classification/Named Entity Recognition (NER) model.

* **Model Choice:** A quantized MobileBERT or DistilBERT fine-tuned for financial NER.
* **Memory Footprint:** ~40MB to 80MB (compared to 1.5GB for an SLM).
* **Speed:** Milliseconds per inference, zero thermal throttling.
* **Execution:** Runs entirely inside a Web Worker in your React app so the UI thread doesn't freeze.

### Step-by-Step Implementation Plan

#### Phase 1: Environment & Database Setup (Days 1-3)

1. **Initialize the App:**
Setup Vite with React and TypeScript. Install Capacitor CLI and initialize it.
2. **Database Abstraction Layer:**
Set up `@capacitor-community/sqlite`. Create a singleton service that initializes the schema I provided earlier (`accounts`, `transactions`, `account_balances` view).
*Architectural Note:* Use SQLite's `PRAGMA synchronous = NORMAL;` and `PRAGMA journal_mode = WAL;` (Write-Ahead Logging) to ensure high-throughput insert performance when batch-processing historical SMSes.

#### Phase 2: The SMS Interception Pipeline (Days 4-7)

You need to read SMSes both historically and in real-time. Since we are avoiding Android Studio, you will rely on existing Capacitor plugins.

1. **The Plugin:** Use a community plugin like `capacitor-sms-retriever` or `@capacitor/sms`.
2. **Historical Syncing:**
Query the `content://sms/inbox`. Do **not** pull all messages into the React state at once. Fetch them in batches of 100, pass them through the extraction pipeline, save to SQLite, and clear the JS array to allow garbage collection.
3. **Idempotency Hashing:**
Compute a hash `SHA-256(date + sender + body)` in JavaScript. Use this as the SQLite primary key. This allows the user to re-sync their inbox safely without duplicating ledgers.

#### Phase 3: The Hybrid Processing Engine (Days 8-12)

Create a dedicated Web Worker (`extractor.worker.ts`) in React. The worker prevents the Capacitor UI from freezing when processing strings.

1. **The Fast Path (Regex Engine):**
The worker first checks the SMS against an array of Regex templates (e.g., `/(?:Rs\.?|INR)\s*([\d,]+\.?\d*).*?debited/i`). 90% of messages will match here.
2. **The ML Fallback (Transformers.js):**
If the regex fails, the worker loads the ONNX model into WASM memory.
```typescript
// Inside extractor.worker.ts
import { pipeline, env } from '@xenova/transformers';

// Tell Transformers.js not to use local cached files in PWA mode, fetch from CDN or local assets
env.allowLocalModels = false; 

const extractor = await pipeline('token-classification', 'Xenova/bert-base-NER');
const results = await extractor(smsText);
// Map NER tags (ORG for Merchant, MISC/MONEY for Amount)

```


*Note: You can train a custom tiny model on HuggingFace and compile it to ONNX, hosting the `.onnx` file in your React `public/` folder so it ships inside the Capacitor app bundle offline.*

#### Phase 4: Ledger & UI Computation (Days 13-15)

Since you have a single-user offline app, do not write complex JavaScript map/reduce functions to calculate balances. Let the C-compiled SQLite engine do the work.

1. **UI Rendering:** Fetch data by querying the SQLite `account_balances` View.
2. **Pagination:** When displaying the transaction history list in React, use cursor-based pagination (e.g., `LIMIT 20 OFFSET 0`) or an Infinite Scroll component. Rendering 5,000 DOM nodes in a WebView will crash the app faster than any ML model.

### Key Architectural Takeaways for this Stack

* **PWA to Native:** By keeping the ML layer in WebAssembly (Transformers.js) and the DB in Web-SQL/SQLite, you can develop and debug the entire application at `localhost:5173` in your Chrome browser. When ready, `npx cap sync android` wraps that exact web code into an APK.
* **Memory Safety:** Bypassing generative SLMs in favor of quantized NER models guarantees the app stays well under the 4GB total system RAM constraint, leaving plenty of overhead for the Capacitor Chromium instance.
* **Offline Guarantee:** Because the ONNX weights are bundled in your React `public` folder and SQLite lives in the app's sandboxed storage, airplane mode processing is 100% guaranteed. No data leaves the device.ˀ