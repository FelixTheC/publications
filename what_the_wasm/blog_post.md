# What the WASM — A Polyglot Deep Dive
into the ABI, the JS↔WASM boundary, linear memory, shared memory, and the real cost of going polyglot

---

> "If WASM+WASI existed in 2008, we wouldn't have needed to create Docker. That's how important it is."
>
> — Solomon Hykes, Co-founder of Docker

---

### TL;DR

- WASM is not "faster JavaScript". It is a **portable, sandboxed, stack-machine ABI** with its own memory model — every JS↔WASM call crosses a typed boundary that you pay for in copies and marshaling.
- Going polyglot (Rust + C++ + Go + Python + JS in one tab) is **technically possible today**, but each language pays a different tax: binary size, glue code, GC, or runtime weight.
- The interesting senior-level questions are **not** "which language?" but: *Who owns the linear memory? When do you copy vs. share? How do you ship threads? How do you debug it in production?*
- The Component Model + WASI 0.2 are quietly turning WASM from a browser feature into a **language-agnostic ABI for software in general**.

---

## Part 1 — The Mental Model You Actually Need

### 1.1 What WASM really is

Forget "binary format for the web" — that's the marketing line. Operationally, WASM is:

1. A **stack-based virtual ISA** with four numeric types (`i32`, `i64`, `f32`, `f64`) plus references (`funcref`, `externref`).
2. A **single linear memory** per instance — a contiguous, resizable `ArrayBuffer` of bytes addressed by `i32` offsets (today; `memory64` is in flight).
3. A **module** with explicit `imports` and `exports`. Nothing is ambient — every syscall, every DOM call, every `console.log` must be imported by the host.
4. A **deterministic, capability-based sandbox**. No filesystem, no network, no clock — unless the host hands them in.

That last point is what makes Hykes's quote land: WASM is a **process model**, not a language runtime.

### 1.2 The five-minute history (skip if familiar)

- **2015** — Mozilla/Google/Microsoft/Apple announce a joint binary format.
- **2017** — MVP ships in all four major browsers. C/C++/Rust → near-native in the browser.
- **2019** — W3C standard. [WASI](https://wasi.dev/) launches → WASM escapes the browser.
- **2022** — [WasmGC](https://github.com/WebAssembly/gc) lands → Kotlin/Java/Dart/Scheme without shipping their own GC.
- **2024** — [WASI 0.2 + Component Model + WIT](https://component-model.bytecodealliance.org/) → typed, language-agnostic interfaces. This is the real inflection point.

### 1.3 Where it's actually used in production (2025)

- **Browser-side heavy apps:** Figma (C++), Photoshop Web (C++/Emscripten), AutoCAD Web, Google Earth, 1Password (Rust crypto core).
- **Edge compute:** Cloudflare Workers, Fastly Compute@Edge, Fermyon Spin, wasmCloud — cold starts in **single-digit milliseconds** vs. ~100–500 ms for containers ([Fastly numbers](https://www.fastly.com/blog/how-lucet-spectre)).
- **Plugin systems:** Envoy filters, Istio, Shopify Functions, Zellij, Lapce.
- **Databases:** SingleStore, SurrealDB, and ScyllaDB embed WASM as a UDF runtime.

---

## Part 2 — The Boundary Is the Product

Almost every interesting WASM design decision happens at the **boundary between the host (JS, Wasmtime, Envoy) and the guest module**. Understanding it is the difference between "I shipped a 40 MB blob that hangs the tab" and "I shipped a 400 KB module that beats native".

### 2.1 Linear memory in one picture

```
 ┌────────────────────────────────────────────────────────────┐
 │  WASM Instance                                             │
 │                                                            │
 │   Linear Memory (one big Uint8Array, host-visible)         │
 │  ┌──────────────────────────────────────────────────────┐  │
 │  │ stack │ static data │ heap → → → → → → → → → →  ░░░ │  │
 │  └──────────────────────────────────────────────────────┘  │
 │     ▲                                                      │
 │     │  i32 offsets (pointers) — NOT host pointers          │
 │     │                                                      │
 │   exports: process(ptr: i32, len: i32) -> i32              │
 │   imports: env.log(ptr: i32, len: i32)                     │
 └────────────────────────────────────────────────────────────┘
            ▲                       ▲
            │ JS can read/write     │ JS calls exports;
            │ the same buffer       │ WASM calls imports
            │ via Module.HEAPU8     │
```

Two rules that drop out of this:

1. **WASM cannot hold a JS object directly.** It holds an `externref` *handle*, or it holds bytes you copied in. Strings, arrays, DOM nodes — none of them exist inside linear memory unless you serialize them.
2. **The host can see everything in the guest's memory.** The sandbox protects the host *from* the guest, not the other way around. Secrets in linear memory are visible to any JS in the page.

### 2.2 The four ways data crosses the boundary

| Mechanism | Cost | When to use |
|---|---|---|
| **Numeric arg/return** (`i32`/`i64`/`f64`) | ~ns, zero-copy | Hot inner loops, handles, lengths |
| **Copy via `HEAPU8.set(...)`** | O(n) memcpy | Small/medium buffers (< few MB) |
| **Shared `WebAssembly.Memory({shared: true})`** | Zero-copy, needs COOP/COEP | Threads, workers, large buffers |
| **`externref` handle table** | ~ns indirection | Passing JS objects (DOM, fetch responses) without serializing |

The mistake I see most often in code review: people serialize a 50 MB image to base64, pass it as a string, then complain WASM is "slow". They measured the JSON parser, not the module.

### 2.3 Threads, `SharedArrayBuffer`, and why your headers matter

WASM threads = `SharedArrayBuffer` + atomics + `Worker`. Browsers gate `SharedArrayBuffer` behind two response headers (Spectre mitigation):

```
Cross-Origin-Opener-Policy:   same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Without both, `WebAssembly.Memory({shared: true})` throws and any pthreads-using module silently degrades to single-threaded — usually visible only as a 4–8× regression in production. This is the single most common "works in dev, breaks behind the CDN" footgun.

### 2.4 The Component Model in one paragraph

Core WASM only speaks `i32`/`i64`/`f32`/`f64`. The **Component Model** adds a higher-level type system — strings, lists, records, variants, resources — described in **WIT** (`*.wit` files). Tools like [`wit-bindgen`](https://github.com/bytecodealliance/wit-bindgen) and [`jco`](https://github.com/bytecodealliance/jco) generate the marshaling glue per language. The practical upshot: in 2024+ you can declare `fn ocr(image: list<u8>) -> result<string, error>` once and have Rust, Python, Go (via TinyGo), and JS bindings generated. This is what kills the "ad-hoc JSON-over-i32-pointers" era we've been living in.

---

## Part 3 — PolyGlyph: A Polyglot Case Study

To make the boundary concrete, let's build **PolyGlyph**: a fully client-side OCR pipeline that processes images **without ever leaving the browser**. No backend, no GPU cluster, no PII over the wire. It's deliberately polyglot — not because that is wise (it usually isn't), but because it forces every interesting WASM problem into one repo.

### 3.1 Why polyglot at all?

The honest answer for a senior audience: **most of the time, don't**. Pick one language (probably Rust) and own the toolchain. Go polyglot only when you have:

- A non-trivial **legacy native codebase** you cannot rewrite (typical for ML inference, codecs, CAD kernels).
- A **third-party model or library** that exists only in one ecosystem (e.g., `onnxruntime` C++, `pandas` in Python).
- A **team-skills constraint** where rewriting is more expensive than paying the polyglot tax.

PolyGlyph hits all three, which is why it's a useful teaching example.

### 3.2 Role split

| Module | Language | Why it lives in WASM | Real cost |
|---|---|---|---|
| Image preprocess (grayscale, contrast) | **Rust** | Tight loop on pixel bytes; zero-cost FFI via `wasm-bindgen` | ~50–200 KB `.wasm` |
| OCR inference (CRNN + CTC decode) | **C++ / Emscripten** | Reuse `onnxruntime` and existing C++ kernels | 2–10 MB `.wasm` + model |
| PII masking (regex) | **Go / TinyGo** | Re-use existing enterprise regex rules from Go services | 300–800 KB (TinyGo) vs ~2–10 MB (stdlib Go) |
| Post-analysis (stats, dataframe) | **Python / Pyodide** | Reuse `pandas`/`numpy` snippets unchanged | **~10 MB** runtime + pkgs |
| UI | **Vue 3 / JS** | DOM, file input, orchestration | — |

### 3.3 End-to-end data flow

**Follow the bytes**, and note the colour of every boundary crossing.

```
   ┌──────────────┐ file
   │  <input>     │────────────┐
   └──────────────┘            ▼
                       ┌───────────────┐
                       │   Vue (JS)    │  RGBA bytes in JS heap
                       │  drawImage    │
                       └──────┬────────┘
                              │ (1) HEAPU8.set  ── copy ──►
                              ▼
                  ┌─────────────────────────┐
                  │ Shared Linear Memory    │  one big Uint8Array
                  └───┬──────┬──────┬───────┘
                      │      │      │
                      ▼      ▼      ▼
                  ┌──────┐ ┌─────┐ ┌────────┐
                  │ Rust │ │ C++ │ │  Go    │
                  │ pre- │ │ OCR │ │ regex  │
                  │ proc │ │ CRNN│ │ PII    │
                  └──┬───┘ └──┬──┘ └───┬────┘
                     │ (2)    │ (3)    │ (4)
                     │ in-place│ ptr   │ string via syscall/js
                     ▼ pixels  ▼ → str ▼
                  ┌─────────────────────────┐
                  │   JS orchestrator       │
                  └──────────────┬──────────┘
                                 │ (5) string copy into Pyodide
                                 ▼
                            ┌─────────┐
                            │ Python  │  pandas-style metrics
                            │ Pyodide │  (own linear memory!)
                            └────┬────┘
                                 │ (6) PyProxy.toJs()
                                 ▼
                            ┌─────────┐
                            │  Vue UI │
                            └─────────┘
```

Key thing to internalize: **Rust, C++, and Go each run as separate WASM instances with their own linear memories.** They look like "one app" only because JS is the broker copying bytes between them. There is **no** shared heap. Every arrow above is either a copy or an `externref` handoff.

### 3.4 Project structure

Even with all these different languages we put them all into one project with a structure similar to this.
```shell
.
├── README.md
├── index.html
├── modules
│   ├── cpp_inference
│   │   ├── CMakeLists.txt
│   │   ├── README.md
│   │   └── src
│   │       └── ocr_bridge.cpp
│   ├── go_compliance
│   │   ├── README.md
│   │   ├── go.mod
│   │   └── main.go
│   └── rust_preprocessor
│       ├── Cargo.toml
│       ├── README.md
│       └── src
│           └── lib.rs
├── package.json
├── public
│   ├── models
│   │   └── text_recognition_CRNN_EN_2021sep.onnx
│   └── wasm
│       └── wasm_exec.js
├── python_scripts
│   └── analysis.py
├── src-vue
│   ├── App.vue
│   └── main.js
└── vite.config.js
```
This helps us to divide our workload into different sections where our
core engines are living under `/modules` our frontend logic is in `/src-vue` and our data processing is in `/python_scripts` and in `/public/models`.

### 3.5 Rust — the zero-overhead reference

Rust is the canonical WASM target for a reason: `wasm-bindgen` + `wasm-pack` produce small modules (often <100 KB after `wasm-opt -Oz`) with no GC and no runtime to ship. More importantly, it has the cleanest **borrow-as-slice** boundary: `&mut [u8]` on the Rust side is *the same bytes* as `Uint8Array` on the JS side — no copy on the way in.
```Rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn preprocess_image_channels(pixels: &mut [u8], width: u32, height: u32) {
    // High-speed, memory-safe pixel manipulation
    for i in (0..pixels.len()).step_by(4) {
        let r = pixels[i] as f32;
        let g = pixels[i+1] as f32;
        let b = pixels[i+2] as f32;

        // Convert to grayscale to optimize for OCR text visibility
        let gray = (0.299 * r + 0.587 * g + 0.114 * b) as u8;
        pixels[i] = gray;
        pixels[i+1] = gray;
        pixels[i+2] = gray;
    }
}
```
**What's actually happening under the hood:**

- `wasm-bindgen` generates a `.js` shim that, for `&mut [u8]`, does *not* `memcpy` — it hands Rust a pointer (`i32`) into JS's view of the WASM heap and a length. The loop mutates pixels in place. Round-trip cost ≈ a single function call.
- Contrast this with the equivalent in raw Emscripten C, where you'd typically `Module._malloc(n)` + `HEAPU8.set(...)` + call + `Module._free(...)`. That's two extra memcpys and two boundary crossings per call.
- **Senior takeaway:** if your hot path is "JS calls Rust 60 times per frame on a `Uint8Array`", you want `wasm-bindgen` with `&mut [u8]` and you want to avoid `JsValue` round-tripping. Profile in DevTools → Performance; the "WebAssembly" tracks show inline call costs.

**Size budget (release profile, `wasm-opt -Oz`, real PolyGlyph build):**

```
rust_preprocessor_bg.wasm   72 KB   (38 KB brotli)
rust_preprocessor.js        18 KB   (auto-generated glue)
```

That's small enough to inline into your main bundle. Keep it that way: every `use serde_json` or `getrandom` you add can pull in tens of KB.

### 3.6 C++ — when you *must* reuse a native codebase

C++ is in PolyGlyph for exactly one reason: **ONNX Runtime ships a C++ API and a maintained Emscripten build**. Rewriting ONNX in Rust is a multi-year project; cross-compiling it is a weekend. This is the realistic case for C++/WASM in 2025 — not "C++ is fast", but "C++ is what the library exists in".

Rather than reproducing 100 lines of ONNX/CTC code here's the **interface stub** that matters.

```cpp
// Three exports are all JS needs to see:
extern "C" {
  EMSCRIPTEN_KEEPALIVE uint8_t*    allocate_wasm_buffer(int size);
  EMSCRIPTEN_KEEPALIVE void        free_wasm_buffer(uint8_t* ptr);
  EMSCRIPTEN_KEEPALIVE const char* process_image_to_string(
      uint8_t* rgba_data, int width, int height);
}
// Inside: bilinear resize → grayscale-normalize → onnxruntime CRNN session
//         → CTC greedy decode → return pointer into a static std::string buffer.
```

**What's interesting here is *not* the ONNX code — it's the ABI:**

- `allocate_wasm_buffer` / `free_wasm_buffer` exist because **JS owns the lifecycle of bytes in linear memory**. There is no GC reaching across the boundary; if JS forgets to call `free_wasm_buffer`, you have a textbook C leak that lives until the page is closed.
- `process_image_to_string` returns a `const char*` — a **pointer into the module's static memory**. The JS side reads it with `Module.UTF8ToString(ptr)`, which walks bytes until a NUL. If C++ ever returns binary data this way, you'll truncate at the first zero byte. Use `(ptr, len)` pairs for anything non-textual.
- The `static std::string g_output_text` is a deliberate, well-known footgun: the next call overwrites it. Fine for a single-threaded pipeline; a race condition the moment you introduce Web Workers.
- ONNX Runtime's Emscripten build is **~8–12 MB** (`.wasm` + glue), and the CRNN model is another **~10 MB**. That's the real cost of "just running ONNX in the browser". Streaming-compile (`WebAssembly.compileStreaming`) and serve with `Content-Encoding: br` — Brotli typically cuts `.wasm` by ~50%.

**When to actually do this:** when the model is small, the user has bandwidth, and inference latency or privacy beats round-tripping to a server. Otherwise: keep ONNX server-side and use WASM only for the preprocessing/postprocessing.

<details>
<summary>Click to expand the full C++ reference implementation</summary>

```CPP
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <onnxruntime_cxx_api.h>
#include <emscripten.h>

// The 36-character vocabulary used by OpenCV's English CRNN model
const std::string CHARSET = "0123456789abcdefghijklmnopqrstuvwxyz";

// Global string buffer to safely hold text for JavaScript extraction
std::string g_output_text;

// Basic CTC Greedy Decoder: takes raw model probabilities and turns them into text
std::string ctc_decode(const float* output_data, int sequence_length, int alphabet_size) {
    std::string text = "";
    int prev_class_idx = -1;

    for (int i = 0; i < sequence_length; ++i) {
        int max_class_idx = 0;
        float max_prob = output_data[i * alphabet_size];

        // Find character with highest probability for this time-step (ArgMax)
        for (int j = 1; j < alphabet_size; ++j) {
            float prob = output_data[i * alphabet_size + j];
            if (prob > max_prob) {
                max_prob = prob;
                max_class_idx = j;
            }
        }

        // In CTC, the last index (alphabet_size - 1) is reserved for the "blank" token
        if (max_class_idx != (alphabet_size - 1) && max_class_idx != prev_class_idx) {
            text += CHARSET[max_class_idx];
        }
        prev_class_idx = max_class_idx;
    }
    return text;
}

extern "C" {

    EMSCRIPTEN_KEEPALIVE
    uint8_t* allocate_wasm_buffer(int size) { return new uint8_t[size]; }

    EMSCRIPTEN_KEEPALIVE
    void free_wasm_buffer(uint8_t* ptr) { delete[] ptr; }

    EMSCRIPTEN_KEEPALIVE
    const char* process_image_to_string(uint8_t* rgba_data, int width, int height) {

        // 1. Preprocess: OpenCV CRNN expects a single-channel Grayscale float input
        // Model dimensions required by OpenCV Zoo: 100x32 (Width x Height)
        int target_width = 100;
        int target_height = 32;
        std::vector<float> input_tensor_values(target_width * target_height, 0.0f);

        // Simple bilinear resize & grayscale conversion mapping into the 100x32 buffer
        for (int y = 0; y < target_height; ++y) {
            for (int x = 0; x < target_width; ++x) {
                int src_x = x * width / target_width;
                int src_y = y * height / target_height;
                int src_idx = (src_y * width + src_x) * 4;

                // Convert RGBA to Grayscale value normalized between 0.0 and 1.0
                float r = rgba_data[src_idx + 0];
                float g = rgba_data[src_idx + 1];
                float b = rgba_data[src_idx + 2];
                float gray = (0.299f * r + 0.587f * g + 0.114f * b) / 255.0f;

                input_tensor_values[y * target_width + x] = gray;
            }
        }

        // 2. Inference setup
        static Ort::Env env(ORT_LOGGING_LEVEL_WARNING, "OpenCWOcr");
        static Ort::SessionOptions session_options;
        static Ort::Session session(env, "models/text_recognition_CRNN_EN_2021sep.onnx", session_options);

        // Model shape input format: [Batch, Channels, Height, Width] -> [1, 1, 32, 100]
        std::vector<int64_t> input_shape = {1, 1, target_height, target_width};
        auto memory_info = Ort::MemoryInfo::CreateCpu(OrtArenaAllocator, OrtMemTypeDefault);

        Ort::Value input_tensor = Ort::Value::CreateTensor<float>(
            memory_info, input_tensor_values.data(), input_tensor_values.size(),
            input_shape.data(), input_shape.size()
        );

        // Input and Output layer names from the OpenCV ONNX file configuration
        const char* input_names[] = {"input"};
        const char* output_names[] = {"output"};

        auto output_tensors = session.Run(
            Ort::RunOptions{nullptr},
            input_names, &input_tensor, 1,
            output_names, 1
        );

        // 3. Extract output shape parameters
        // Output tensor shape is [Sequence Length (typically 25), Batch (1), Vocabulary Size (37)]
        auto output_shape = output_tensors[0].GetTensorTypeAndShapeInfo().GetShape();
        int seq_len = output_shape[0];
        int alphabet_size = output_shape[2];

        float* raw_output = output_tensors[0].GetTensorMutableData<float>();

        // 4. Decode the text natively using C++ CTC algorithm
        g_output_text = ctc_decode(raw_output, seq_len, alphabet_size);

        // Return C-string pointer directly across the WASM boundary to JavaScript
        return g_output_text.c_str();
    }
}
```


</details>

### 3.7 Go — and the TinyGo vs. stdlib trade-off

The reason Go is in PolyGlyph: the company already has battle-tested PII-masking regexes in Go services and doesn't want a second source of truth. Two paths to ship Go as WASM, and the choice is consequential:

| | **stdlib `GOOS=js GOARCH=wasm`** | **TinyGo** |
|---|---|---|
| Binary size | 2–10 MB (drags the full Go runtime) | 300–800 KB |
| Goroutines | Full scheduler | Cooperative subset |
| `reflect` / `encoding/json` | Full | Partial / patchy |
| Startup time | ~100–300 ms parse+init | ~10–30 ms |
| Drop-in for arbitrary Go libs | Mostly yes | **No** — many deps don't compile |

For a single regex function, TinyGo is the obvious choice. For "compile our existing 50k-LOC service", you'll fight TinyGo and probably need stdlib Go — at which point you should question whether this belongs in the browser at all.

The `syscall/js` API used below is the **pre-Component-Model** way to talk to JS — verbose, untyped, but stable. With WASI 0.2 + `wit-bindgen-go` this is starting to look very different; expect the boilerplate to disappear in the next 12 months.
```Go
package main

import (
	"regexp"
	"syscall/js"
)

// Exported function to JavaScript
func maskSensitiveData(this js.Value, args []js.Value) interface{} {
	text := args[0].String()

	// Enterprise Regex to find and mask credit card numbers locally
	re := regexp.MustCompile(`\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b`)
	maskedText := re.ReplaceAllString(text, "[REDACTED_CARD]")

	return maskedText
}

func main() {
	// Block execution and register function in JS global scope
	c := make(chan struct{}, 0)
	js.Global().Set("maskSensitiveData", js.FuncOf(maskSensitiveData))
	<-c
}
```

### 3.8 Python via Pyodide — useful, but the elephant in the bundle

[Pyodide](https://pyodide.org/) is **CPython compiled to WASM** plus a curated set of scientific packages. For PolyGlyph's post-analysis (a few `pandas`-style metrics on a string) it's overkill — but if your data scientists already wrote the script and you don't want to port it, Pyodide turns "rewrite in JS" into "ship the runtime".

The trade-off is **brutal in bytes**:

- Core Pyodide runtime: **~6.5 MB** Brotli-compressed, ~11 MB uncompressed.
- Add `numpy`: +3.5 MB. Add `pandas`: +8 MB. ([Pyodide size docs](https://pyodide.org/en/stable/usage/wasm-constraints.html))
- First-load cold start: typically 1–3 s on a fast connection, dominated by package download + module instantiate.

**Practical patterns:**

- Load Pyodide **lazily** on first use, not at page load. Show a "warming up" state.
- Cache the runtime in a Service Worker — repeat visits become instant.
- Use **PyProxy** carefully: every `pyodide.globals.get('foo')(jsObj)` is a boundary crossing that converts via a proxy table; for tight loops, pre-convert.
- Consider whether `mathjs` / `arquero` / hand-written JS would do the same job in 200 KB.
```Python
// Load the Pyodide WASM Runtime
let pyodide = await loadPyodide();

// Run a raw Python data analysis script over the text inside the browser
let pythonScript = `
def analyze_text(ocr_output):
    words = ocr_output.split()
    metrics = {
        "word_count": len(words),
        "is_invoice": "invoice" in ocr_output.lower()
    }
    return metrics
`;

await pyodide.runPythonAsync(pythonScript);
const analyze = pyodide.globals.get('analyze_text');
const results = analyze(finalOcrText);
console.log("Python analysis metrics:", results.toJs());
```

### 3.9 Bundling: Vite, COOP/COEP, and the things that bite in production

This is the section worth printing out and pinning to your monitor. It's where most "it works on my machine" WASM stories die.

**Build-time:** three separate toolchains (`wasm-pack`, `tinygo`, `emcmake`) produce three separate `.wasm` + glue pairs into `public/wasm/`. Keep them out of `node_modules` and out of Vite's transform pipeline — they're already optimized.
```json
{
  "name": "polyglot-wasm-ocr",
  ...
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "--- WASM COMPILATION SCRIPTS ---": "",
    "build:rust": "cd modules/rust_preprocessor && wasm-pack build --target web --out-dir ../../public/wasm/rust",
    "build:go": "cd modules/go_compliance && tinygo build -o ../../public/wasm/go_compliance.wasm -target wasm main.go && cp $(tinygo env TINYGO_ROOT)/targets/wasm_exec.js ../../public/wasm/",
    "build:cpp": "cd modules/cpp_inference && mkdir -p build && cd build && emcmake cmake .. && emmake make",
    "build:all-wasm": "npm run build:rust && npm run build:go && npm run build:cpp"
  },
  "dependencies": {
    ...
  },
  "devDependencies": {
    ...
  }
}
```
**Runtime headers — the COOP/COEP story:**

The `vite.config.js` below sets two HTTP response headers on the dev server. They look innocuous but they unlock `SharedArrayBuffer` (Spectre mitigation, see §2.3). Without them, anything using WASM threads — that includes `onnxruntime-web`'s threaded build, FFmpeg-wasm, Pyodide's threaded matrix ops — silently falls back to the slow single-threaded path. **Set these headers at your CDN/edge too, not just in dev**, or your prod build will quietly regress. The same flags also break embedding cross-origin iframes/images that don't send `Cross-Origin-Resource-Policy`, so plan for it.
```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/react-refresh';
import vue from '@vitejs/plugin-vue';
import wasm from 'vite-plugin-wasm';
import topLevelAwait from 'vite-plugin-top-level-await';

export default defineConfig({
    plugins: [
        react(),
        vue(),
        wasm(),
        topLevelAwait()
    ],
    server: {
        headers: {
            // Required security profiles to unlock local high-performance WebAssembly memory buffer threads
            'Cross-Origin-Opener-Policy': 'same-origin',
            'Cross-Origin-Embedder-Policy': 'require-corp',
        },
    },
});
```

### 3.10 Vue frontend — JS as the broker

Vue is doing two jobs here: the UI, and **orchestration of three independent WASM instances**. The interesting bit is not the framework — it would look identical in React or Solid — but the **order, ownership, and copy semantics** of the calls.

All modules run in the **same browser tab thread by default**. That's good (zero serialization between modules via shared JS heap references) and bad (one infinite loop in Rust freezes everything, including the UI). For production-grade pipelines, move the WASM modules into a `Worker` and use `postMessage` with `Transferable`s (`ArrayBuffer.transfer`) to move ownership without copying.

**Note on the snippet below:** the `tokenizer.decode` call is illustrative — **not** a copy-paste production code.
```javascript
<script setup>
import {ref, onMounted} from 'vue';
import initRust, {preprocess_image_channels} from '../../public/wasm/rust/rust_preprocessor.js';
import {AutoTokenizer} from '@huggingface/transformers';

const status = ref('Initializing pipeline...');
const ocrText = ref('');
const metrics = ref(null);

onMounted(async () => {
  // Load Rust Engine
  await initRust();

  // Load Go Engine
  const go = new window.Go();
  const goWasm = await WebAssembly.instantiateStreaming(fetch('/wasm/go_compliance.wasm'), go.importObject);
  go.run(goWasm.instance);

  // Load Python Engine
  window.pyodide = await window.loadPyodide();

  status.value = 'Pipeline ready for execution.';
});

async function handleImageProcess(file) {
  const img = await loadImageElement(file);
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  canvas.width = img.width;
  canvas.height = img.height;
  ctx.drawImage(img, 0, 0);

  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const rawPixels = imageData.data;

  // Allocate and copy pixels into WASM linear memory heap
  const wasmBufferPtr = Module._allocate_wasm_buffer(rawPixels.length);
  Module.HEAPU8.set(rawPixels, wasmBufferPtr);

  // Call the updated C++ processor function
  // It processes the image, runs the ONNX model, decodes the CTC matrix,
  // and returns a pointer to an immutable C-String char array.
  const cStringPtr = Module._process_image_to_string(wasmBufferPtr, canvas.width, canvas.height);

  // Read the string directly from the WASM Memory Heap
  const rawTextFromOcr = Module.UTF8ToString(cStringPtr);

  // Clean up allocated buffer memory allocations
  Module._free_wasm_buffer(wasmBufferPtr);

  // Continue the pipeline: Go compliance masking -> Python data evaluation
  const safeText = window.maskSensitiveData(rawTextFromOcr);
  return safeText;
}

const onFileSelected = async (event) => {
  status.value = 'Processing...';
  const file = event.target.files[0];
  const img = await handleImageProcess(file);
  let imgData = getCanvasPixels(img);

  // 1. Rust Pipeline Step
  preprocess_image_channels(imgData.data, imgData.width, imgData.height);

  // 2. C++ Local Inference Execution
  const tokenizer = await AutoTokenizer.from_pretrained('microsoft/trocr-base-printed');
  const tokenIds = window.Module._process_image_to_tokens(imgData.data, imgData.width, imgData.height);
  const rawText = await tokenizer.decode(tokenIds, {skip_special_tokens: true});

  // 3. Go Compliance Scrubbing
  ocrText.value = window.maskSensitiveData(rawText);

  // 4. Python Native Analysis Evaluation
  const pyScript = await (await fetch('/python_scripts/analysis.py')).text();
  await window.pyodide.runPythonAsync(pyScript);
  const analyzeFn = window.pyodide.globals.get('analyze_text');

  metrics.value = analyzeFn(ocrText.value).toJs();
  status.value = 'Pipeline Complete.';
};
</script>

<template>
  <div class="container">
    <h1>Enterprise Polyglot WASM Pipeline (Vue 3)</h1>
    <p>Status: <span>{{ status }}</span></p>

    <input type="file" @change="onFileSelected" accept="image/*"/>

    <section v-if="ocrText">
      <h3>Sanitized Text Output:</h3>
      <p class="box">{{ ocrText }}</p>
    </section>

    <section v-if="metrics">
      <h3>Python Extraction Profile Metrics:</h3>
      <ul>
        <li>Word Metric Count: {{ metrics.word_count }}</li>
        <li>Is Invoice Detected: {{ metrics.is_invoice }}</li>
      </ul>
    </section>
  </div>
</template>

```

---

## Interlude — Paradigm Shift: Bypassing the DOM with UI Runtimes (Qt/QML)

PolyGlyph above is the **hybrid layout model** that almost every WASM tutorial assumes: a normal HTML/CSS/JS app, with computational hotspots offloaded to WebAssembly modules. The browser still owns layout, styling, accessibility, and event dispatch; WASM is a co-processor.

There is a second, very different model that's worth knowing about — especially if you come from a desktop, embedded, or games background. Frameworks like [**Qt for WebAssembly**](https://doc.qt.io/qt-6/wasm.html) (using QML) treat the browser tab as a **bare graphics runtime** and bypass the DOM entirely.

```
Traditional Web WASM (PolyGlyph):
  [React / Vue components + DOM] ◄──(JS ↔ WASM boundary)──► [WASM modules]

Qt / QML App Runtime WASM:
  [QML declarative UI + C++ core] ──► compiled to WASM ──► paints directly to
                                                          <canvas> via WebGL / WebGPU
```

When you compile a declarative QML application to WebAssembly, the browser sees **one `<canvas>` and one big `.wasm` blob**. Inside that blob runs a native C++ engine — Qt's own scene graph, layout, animation, and input subsystems — that maintains its own widget tree and rasterizes pixels straight into the canvas via WebGL (today) or WebGPU (increasingly). The browser's DOM layout/styling/reflow pipeline is never touched.

A trivial QML example to make the shape of it concrete:

```qml
import QtQuick
import QtQuick.Controls

ApplicationWindow {
    width: 400; height: 350
    visible: true; color: "#121212"

    Column {
        anchors.centerIn: parent
        spacing: 25

        Rectangle {
            id: customButton
            width: 220; height: 60; radius: 30
            color: buttonMouseArea.pressed ? "#deff9a" : "#1e1e1e"
            border.color: "#deff9a"; border.width: 2

            Behavior on color { ColorAnimation { duration: 150 } }

            Text {
                anchors.centerIn: parent
                text: "Run Native C++ Hash"
                font.bold: true; color: "#ffffff"
            }

            MouseArea {
                id: buttonMouseArea
                anchors.fill: parent
                onClicked: {
                    // Direct UI-to-native call inside the same compiled sandbox binary.
                    // No JS↔WASM marshaling, no DOM event round-trip.
                    let result = backend.computeHeavyHash("SecureWasmInputData_2026");
                    hashDisplay.text = result;
                }
            }
        }

        Text {
            id: hashDisplay
            text: "Click button to compute SHA-256"
            color: "#daffde"
            font.family: "Monospace"
        }
    }
}
```

### Why an architect should care

- **True cross-platform code reuse.** The same QML + C++ codebase compiles natively for Windows, macOS, Linux, iOS, Android — *and* the browser, via the same source. No second frontend.
- **Elimination of JS-framework churn.** The UI is rendered by a highly tuned C++ scene graph running at 60+ FPS, with no CSS specificity wars, no hydration mismatches, and no per-browser DOM layout-engine quirks.
- **Single-process, single-boundary call cost.** The "UI" and the "backend C++" live in the *same* WASM instance and the *same* linear memory — that QML `onClicked` calling `backend.computeHeavyHash(...)` is an in-process C++ call, not a JS↔WASM boundary crossing. Contrast with PolyGlyph, where every Rust/C++/Go call pays the boundary tax (see §2.2).

### Why most teams should still hesitate

The trade-offs are not subtle, and they're the inverse of every benefit:

- **Binary size is brutal.** A non-trivial Qt/WASM app is typically **10–40 MB** of `.wasm` + glue before your own code, dominated by the Qt runtime and any used modules (`QtQuick`, `QtQuickControls`, `QtCharts`). Compare with a Vue + tiny Rust module at <500 KB.
- **You opt out of the web platform.** No real DOM means: accessibility tools (screen readers) see one opaque `<canvas>`; browser text selection, find-in-page, translate, password managers, and SEO crawlers all stop working. Qt has been improving its a11y bridge, but it is **not** at parity with HTML.
- **No `SharedArrayBuffer` headers, no threads.** Qt/WASM threaded builds need COOP/COEP just like §2.3 — and the threaded build is significantly larger again.
- **Mobile and low-end devices suffer.** A 20 MB download + WebGL context + canvas-driven scroll is a different performance profile than an HTML page; expect issues on cheap Android devices.
- **It is a paradigm choice, not a feature flag.** You cannot incrementally mix Qt/QML and React/Vue in the same page in any sane way. Pick one.

### When this model actually wins

- You're porting an **existing Qt desktop product** to the browser and want one codebase across desktop/mobile/web (CAD tools, industrial HMIs, scientific instruments, trading terminals).
- The UI is **graphics-heavy and non-document-like** — timelines, node graphs, 3D viewports, DAW-style editors — where the DOM would fight you anyway.
- You control the deployment context (kiosks, internal apps, embedded WebViews) where bundle size, SEO, and accessibility constraints are relaxed.

For the typical "I have a SaaS dashboard" case, this is the wrong tool. But it's worth knowing the model exists, because it reframes the question: **WASM is not just a co-processor for the DOM — it is also a delivery vehicle for entire native application runtimes.** Figma (their own C++ renderer on canvas), Google Earth, and Photoshop Web sit on a spectrum with Qt/QML; understanding both ends of that spectrum is what separates senior architectural judgement from "just add a WASM module".

---

## Part 4 — Server-Side WASM (the other half of the story)

Everything we've looked at so far — the PolyGlyph polyglot pipeline in Part 3, and the Qt/QML canvas-runtime model in the Interlude — lives in the **browser**. Two very different shapes (co-processor vs. full native runtime), but the same host: a tab, a JS engine, a DOM (or a `<canvas>` standing in for one).

Flip the deployment target and almost every constraint inverts. There is no DOM to bypass, no `SharedArrayBuffer` headers to negotiate, no 10 MB Pyodide download to amortize over a user's session. What you *do* get is the property the Hykes quote was really about: a **portable, sandboxed, capability-scoped process** that starts in microseconds and runs the same binary on a laptop, an edge POP, and a Kubernetes node. That is where the most interesting WASM activity in 2024–2025 is happening, and it's the half of the story that browser-centric tutorials almost always skip.

The server-side runtime landscape today:

| Runtime | Use case | Notable property |
|---|---|---|
| [Wasmtime](https://wasmtime.dev/) | General-purpose embed (Bytecode Alliance reference) | First-class Component Model + WASI 0.2 support |
| [Wasmer](https://wasmer.io/) | General-purpose, multi-engine | Multiple backends (Cranelift, LLVM, Singlepass) |
| [WasmEdge](https://wasmedge.org/) | Edge / Kubernetes (CNCF) | AOT compile, gRPC, HTTPS, AI extensions |
| [Fermyon Spin](https://www.fermyon.com/spin) | HTTP microservices in WASM | Sub-millisecond cold start; one component per request |
| [wasmCloud](https://wasmcloud.com/) | Distributed actors with WIT contracts | Hot-swappable components, capability providers |
| Cloudflare Workers / Fastly Compute@Edge | Edge functions | V8 isolates + WASM (CF); Wasmtime-derived (Fastly) |

**Why this matters:** Spin starts a WASM HTTP handler in **~1 ms** vs ~100–500 ms for a typical container cold start ([Fermyon benchmarks](https://www.fermyon.com/blog/cold-start-time)). That changes the economics of "scale-to-zero" workloads — function-per-request becomes viable where containers were too slow. The trade-off: no threads on the server side until you adopt `wasi-threads`, no raw sockets, and a smaller library ecosystem than Node/Python.

If your team is evaluating "serverless v2", this is the technology to benchmark against AWS Lambda / Cloud Run. The cost model (per-millisecond billing on microsecond starts) is materially different.

---

## Part 5 — Debugging, Security, and the Stuff Nobody Demos

### 5.1 Debugging WASM in 2025

The old answer ("Chrome shows you hex offsets, good luck") is finally outdated.

- **DWARF source maps** — compile with `-g` (Emscripten), `--debug` (`wasm-pack`), or `-g` (TinyGo). Chrome DevTools' [C/C++ debugging extension](https://chromewebstore.google.com/detail/cc++-devtools-support-dwa/pdcpmagijalfljmkmjngeonclgbbannb) gives you stepping, breakpoints, and variable inspection in *original C/C++/Rust source*.
- **`console.log` from inside WASM** — import a logger from JS; Rust's `web-sys` and Emscripten's `printf` both do this for you.
- **Stack traces** — enable name-section preservation (`--emit-symbol-map` / debug build) so traces show function names, not `wasm-function[1234]`.
- **Server side:** Wasmtime ships `--profile=guest` and integrates with `perf`. Fermyon Spin has an OpenTelemetry exporter.

### 5.2 Security — what the sandbox does and doesn't give you

What it **does** give you:

- Memory safety *for the host*: the guest cannot read host memory or escape its linear memory.
- Capability-based syscalls (WASI): no ambient filesystem, no ambient network. You must explicitly grant a preopen directory or socket.
- Deterministic execution (no instruction-level non-determinism), which makes WASM attractive for blockchains and reproducible builds.

What it **does not** give you:

- **Memory safety inside the guest.** A C++ buffer overflow inside a WASM module is still a buffer overflow — it just can't escape into the host. CVEs in WASM-compiled libraries are real (e.g., ImageMagick-wasm inheriting upstream CVEs).
- **Side-channel resistance.** Spectre/Meltdown-class timing leaks still exist; that's why browsers gate `SharedArrayBuffer` behind COOP/COEP.
- **Resource exhaustion protection by default.** A guest can `malloc` until linear memory exhausts (limit is 4 GiB pre-`memory64`). Runtimes need explicit fuel/memory limits.
- **Supply-chain protection.** A malicious `.wasm` published as a Vite plugin is no safer than a malicious JS package. Audit your binaries.

---

## Part 6 — Numbers, Not Vibes

Pulled from public benchmarks. Use as order-of-magnitude reference, not promises — measure your own workload.

| Metric | Typical value | Source |
|---|---|---|
| WASM execution vs. native (CPU-bound) | 0.5×–0.95× | [PLDI '19 "Not so fast"](https://dl.acm.org/doi/10.1145/3314221.3314623); various Bytecode Alliance benchmarks |
| Wasmtime cold start (function instantiate) | 5–50 µs (AOT, pre-instantiated) | [Wasmtime perf docs](https://docs.wasmtime.dev/) |
| Spin HTTP handler cold start | ~1 ms | [Fermyon](https://www.fermyon.com/blog/cold-start-time) |
| AWS Lambda (Node) cold start | 100–800 ms | AWS public data |
| Container cold start (k8s scheduling) | 1–10 s | Industry consensus |
| Rust `wasm-pack` "hello world" `.wasm` | ~2 KB (after `wasm-opt -Oz`) | rustwasm book |
| TinyGo "hello" `.wasm` | ~10 KB | TinyGo docs |
| Stdlib Go `.wasm` (`GOOS=js`) | ~2–10 MB | Go release notes |
| Pyodide core runtime | ~6.5 MB brotli | [Pyodide constraints](https://pyodide.org/en/stable/usage/wasm-constraints.html) |
| `onnxruntime-web` (threaded) | ~10 MB | onnxruntime releases |
| JS↔WASM call overhead (numeric args) | ~5–20 ns | V8 / SpiderMonkey blog posts |

The takeaway: WASM is "near-native" for **CPU-bound** code. The moment your workload is dominated by **boundary crossings** or **large data marshaling**, the language and ABI you chose matter far more than "WASM is fast".

---

## Part 7 — When to Choose WASM (decision section)

This replaces the old benefits/drawbacks/matrix triad with a single guide.

**🟢 Reach for WASM when:**

- You need **ephemeral, scale-to-zero** workloads where container cold start is the bottleneck (edge functions, per-request isolation, multi-tenant plugin hosts).
- You have a **legacy native codebase** (C/C++/Rust) that's expensive to port to JS or to keep behind a network call.
- You need a **safe, sandboxed plugin runtime** for untrusted code (Envoy filters, Shopify Functions, database UDFs, IDE extensions).
- Your **data is sensitive** and shouldn't leave the user's device (on-device ML, client-side crypto, PII processing).
- You're building a **portable component** that must run identically in browser, edge, and server.

**🐳 Stick with containers / Node / native processes when:**

- The workload is **long-running and warm** — cold start doesn't matter; throughput and library ecosystem do.
- You are **OS-kernel-coupled** (raw sockets, eBPF, deep POSIX, kernel-bypass NICs, GPUs without WebGPU).
- Your team is **two people and you don't have toolchain bandwidth.** Polyglot WASM has a real ops cost; one well-written Node or Go service is often the right answer.
- You need **mature observability** today (APM, profilers, tracing). The WASM tooling is improving fast but is still behind containers.

**🤔 The honest "it depends":**

- Polyglot-in-one-tab (the PolyGlyph pattern). Powerful, but the surface area is large. Until the Component Model is fully shipped end-to-end, expect to hand-write glue and absorb ~10–20 MB of runtimes if Python or threaded ONNX is in the mix.

---

### Wrap-up

WASM didn't come to kill JavaScript — it came to give the entire industry a **portable, sandboxed ABI** that JavaScript happens to host first. For mid-to-senior engineers, the productive shift in 2025 is to stop asking "is WASM faster?" and start asking:

- *Where does my data cross the boundary, and how many times?*
- *Who owns the linear memory at each step?*
- *What's the size budget — and which runtime am I implicitly shipping?*
- *Will the Component Model + WASI 0.2 let me delete half this glue in 12 months?*

If you can answer those four questions for your system, you're already ahead of most teams adopting WASM today.

### Further reading

- [WebAssembly Specification (W3C)](https://www.w3.org/TR/wasm-core-2/)
- [Component Model Book — Bytecode Alliance](https://component-model.bytecodealliance.org/)
- [WASI 0.2 (Preview 2) overview](https://bytecodealliance.org/articles/WASI-0.2)
- [`wasm-bindgen` guide](https://rustwasm.github.io/wasm-bindgen/)
- [Emscripten docs](https://emscripten.org/)
- [TinyGo WebAssembly](https://tinygo.org/docs/guides/webassembly/)
- [Pyodide](https://pyodide.org/)
- [Fermyon Spin](https://www.fermyon.com/spin), [wasmCloud](https://wasmcloud.com/), [Wasmtime](https://wasmtime.dev/)
- Lin Clark, *"WebAssembly: building a new kind of ecosystem"* — the canonical talk on the Component Model.

### Thanks for reading

I hope this changed how you think about the JS↔WASM boundary.
