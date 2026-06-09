# What the WASM
A short introduction to WebAssembly (WASM) and its potential impact on the web development landscape.
> "If WASM+WASI existed in 2008, we wouldn't have needed to create Docker. That's how important it is."
>
> — Solomon Hykes, Co-founder of Docker

## What is WebAssembly (WASM) in a nutshell
WebAssembly (WASM) is a binary instruction format for a stack-based virtual machine like Assembly. 
It was originally designed as a portable compilation target for high-level languages like C and C++, 
but has since expanded to support other languages and platforms, like C#, Rust, Go, Python, and more. 
The primary goal of WASM is to enable efficient and secure execution of code in web browsers, reducing the need for plugins and improving performance.
It always had the completion of JavaScript in mind, as it was designed to run alongside JavaScript in the browser, but it has since evolved to become a standalone technology with its own ecosystem and tooling.

## A short history on WebAssembly (WASM)
- 2015: The Announcement
  - Mozilla, Google, Microsoft, and Apple announcing a joint collaboration to create a new binary code format for the web.
- 2017: The MVP Launches
  - MVP was officially completed
  - supported in all four major browsers (Firefox, Chrome, Safari, and Edge)
  - Developers could now compile C, C++, and Rust to run in the browser at near-native speed.
- 2019: The Great Escape (W3C & WASI)
  - Official Standard: The W3C declared WebAssembly the fourth official language of the web (joining HTML, CSS, and JavaScript).
  - WebAssembly System Interface (WASI) was launched
  - This allows WASM to run outside the browser, directly on servers, edge networks, and IoT devices.
- 2023–Present: Maturity & The Component Model
  - Native Garbage Collection (WasmGC): Added support for languages like Kotlin, Java, and Dart to run efficiently without shipping their own massive runtimes
  - Component Model: Allowed different languages to talk to each other seamlessly in a single WASM application

## WASM today
WASM has become a versatile and powerful technology, with a growing ecosystem of tools, compilers, and runtimes. It is no longer limited to the browser, but can now run on a wide range of platforms, from servers to edge networks and IoT devices. WASM's ability to run multiple languages seamlessly in a single application has opened up new possibilities for developers, enabling them to build more complex and efficient systems. As WASM continues to evolve, it is likely to play an increasingly important role in the future of computing.

Nowadays, we can see that a lot of heavy Desktop apps are moving into to the web with WASM. 
Some examples are Figma, AutoCAD, Adobe Photoshop Web, Google Earth, and Google Docs. These applications are now able to run in the browser, providing a more seamless and efficient user experience. WASM's ability to run multiple languages seamlessly in a single application has also made it possible for developers to build more complex and efficient systems, such as the popular game engine Unity, which now supports WASM as a target platform.

Further, it enables also Edge and Serverless computing for example on Cloudflare, Fastly, and AWS which enabling developers to build and deploy applications without worrying about the underlying infrastructure. 

## What is the general WASM compilation and loading Workflow?
In general, we can say that creating a WASM binary follows these steps.
1. We have some Source Code from some high-level coding language (Rust, Go, Python, C++, etc...)
2. Compilation, with wasm-pack or emscripten and a defined target we compile our code base
3. the output from step 2. it will give us a *.wasm and a *.js file where the *.js file contains our glue between our binary and a JavaScript environment
4. after we upload our files and make it statically available, the browser can fetch, compile, and instantiate the binary alongside JavaScript within its own runtime 

## How can we use WASM?
As developers we have multiple options to use WASM. Either directly together with plain JavaScript or one of the many frameworks,  
or we could use QT with QML which enables use to develop for Desktop (any Platform), Android, and Web simultaneously.
Within QT/QML you can use C++ and Python (official supported), Qt Quick and QML will then create a WASM binary which bypasses the DOM and draws direct to a canvas via WebGL.

```QML 
import QtQuick
import QtQuicj.Controls

ApplicationWindow {
  width: 400;
  height: 350;
  
  ...
  
  MouseArea {
    id: buttonMouseArea
    anchors.fill: parent
    onClicked: {
      let rawData = "SomeDataFile";
      let result = backend.computeHeavyHash(rawData);
      
      hashDisplay.text = result;
    }
  }
  
  Text {
    id: hashDisplay
    text: ""
    ...
  }
}
```

```CPP
// Backend.hpp
class Backend: public QObject {
  Q_OBJECT
  
public:
  explizit Backend(QObject* parent = nullptr);
  Q_INVOKABLE QString computeHeavyHash(const QString& rawData);
};

// Backend.cpp
Backend::Backend(QObject* parent) : QObject(parent) {}

QString Backend::computeHeavyHash(const QString& rawData) {
  QByteArray hash = QCrpytographicHash::hash(rawData.toUtf8)
  return QString(hash.toHex());
}
```

## A Practical Example: PolyGlyph
For a better understanding on how WASM can help us with our architecture when it comes to complex computations we imagine a OCR related task and our Team isn't sure how to start solving it. 
Only a few Team members now Docker/Kubernetes and how to use them to deploy our application. Also the knowledge of Programmimg languages is quite shared, and everyone tries to force the usage of its favorite.

### A possible solution on how we share the workload among our Team preferred languages
- Rust => for memory safe preprocessing (grayscale/contrast) 
- C++ => Machine Learning Core (ONNX runtime interface)
- Python => with Pyodide data processing (saving output to custom dataframe)
- GO => Enterprise compliance checks (Regex PII data masking)
- Vue => UI/UX

### Our project structure
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

### A look into the Rust Preprocessor
Rust has a dedicated crate for WASM called `wasm-bindgen` which is easy to use and provides a simple way to expose Rust functions to JavaScript. Which means we need to write less glue code and can reduce potential overhead. 
This makes it perfect to process raw data safely without risks of buffer overflows.
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
More than this is not needed to enable communication between JavaScript and Rust.

### A look into the C++ Machine learning core
C++ as one of the widely used programming languages, especially when it comes to performance and legacy code reuse.
With C++ we have a highly optimized native codebase which we can see after we compile it via emscripten cause this will create nearly bare-metal assembly instruction code.
Also it enables us to load and manipulate quite heavy models and data structures directly within Webassembly.

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
        auto output_shape = output_tensors Qu0].GetTensorTypeAndShapeInfo().GetShape();
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

### A look into our Go module
With TinyGo we're able to compile existing libraries, routing rules and other components into Webassembly. 
This enables us to create a fully native Webassembly module that can be used in the browser without any additional dependencies or plugins. 
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

### A look into the Python (Pyodide) module
With this we can have Client-Side Data Science cause Pyodide enables us to have a full Python runtime compiled to WASM.
This enables us to use existing Pandas or Numpy libraries/scripts directly.
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

### How can we bundle this
For the frontend developers it is important to have multiple ways to build the wasm files. 
For this we need to adapt the `package.json`.
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
To enabled it within vue we need to register the wasm modules within our `vite.config.js`.
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

### Vue frontend
Now the frontend Team can directly use our modules instead of installing heavy frontend npm packages.
With this structure we have all running the same web worker/browser tab thread which makes it simple to transfer data between the different modules (languages)
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

    <section vxl-if="ocrText">
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

### What are the benefits of this architecture?
- It will help us save costs upfront cause we don't need to spinn up additional clusters with expensive GPUs cause the workload is completely offloaded to the user's local machine for free.
- Also, we gain a huge strength for our security and privacy because our wasm modules are running fully client-side in a secure WASM sandbox, this means that no sensitive data leaves the user's machine unententinely.
Overall, we can achieve a high HIPAA, GDPR compliance with less effort compared to the traditional approach with several containers and communication over the Network protocols.
- Our team members can now be split up based on their specific programming language knowledge to tackle different tasks with the least amount of effort.

### Draw backs of this architecture
Besides all the benefits we have a few draw backs especially when it comes to the toolchain. Cause sometimes we have to pay a high tax to make our C++/Rust/Go/Python code compilable to WASM.
C++ can require a complex CMake file while not all libraries are portable to WASM cause of POSIX threads are OS file system calls which are not available in WASM (cause it is sandboxed and only has a virtual file system).
Some Rust crates can use legacy C libraries under the hood which are not/not easy portable to WASM.
We also need to consider that languages like Go and Python require their own compiler and runtime environment which can add complexity to the development process when no prober tools like TinyGo or Pyodide are used we can bloat our wasm binary unnecessarily.
Debuggin can become a nightmare especially when not prober logging is used, because inside the console of the browser we only see the unreadable obfuscated memory addresses of a wasm module but not the actual code nor any other helpfull information. 

## WASM drawbacks
Every architecture/tool has its drawbacks, and WASM is no exception.

| 32-bit architecture                                                 | JS interop overhead                                          | binary bloat                                                                           |
|---------------------------------------------------------------------|--------------------------------------------------------------|----------------------------------------------------------------------------------------|
| for standard WASM                                                   | Passing objects between requires serialization and data copy | Languages which ship their own standard library or runtimes lead to bigger .wasm files |
| Limits linear memory to 4GB (can be a hurdle for data-heavy C++apps | Can negate the performance when used too frequent            | Can slow down initial page loading compared to lightweight JS                          |                                                                                     


## Summary
WASM didn't come to kill JavaScript it came to give it a superpower.
It started as a tool to make web browsers faster, but it has evolved into a secure, language-agnostic,
ultra-fast sandbox for cloud and edge computing
When creating additional microservices consider using WASM as optional choice


### Thanks for reading
I hope you enjoyed this article. And that you learned something new about WASM and its potential impact on the web development landscape.
