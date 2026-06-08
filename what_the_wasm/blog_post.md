# What the WASM
A short introduction to WebAssembly (WASM) and its potential impact on the web development landscape.

## What is WebAssembly (WASM) in a nutshell
WebAssembly (WASM) is a binary instruction format for a stack-based virtual machine like Assembly. 
It was originally designed as a portable compilation target for high-level languages like C and C++, 
but has since expanded to support other languages and platforms, like C#, Rust, Go, Python and more. 
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