# Technical Evaluation Report: Refactoring Artemis Android from Java + C to Kotlin + Rust

**Author:** Artemis Android Engineering Team
**Status:** Architecture Proposal & Feasibility Assessment
**Date:** May 2024

---

## Executive Summary

This document presents a comprehensive feasibility assessment, technical evaluation, trade-off analysis, and step-by-step roadmap for refactoring the **Artemis Android** codebase (a high-performance GameStream / Apollo / Sunshine client) from its current **Java + C** architecture to a modern **Kotlin + Rust** stack.

### Key Takeaways
* **Is it worth it?** **Yes, conditionally.**
  * Migrating **Java to Kotlin** yields immediate benefits in null-safety, async control flow (Coroutines/Flow), UI responsiveness, and code maintainability with zero runtime latency overhead.
  * Rewriting **C (`moonlight-common-c`) to Rust** provides massive improvements in memory safety, thread safety (preventing data races in real-time streaming queues), cross-platform code reuse, and modern crate ecosystem access (e.g., safe async network I/O, modern crypto). However, because `moonlight-common-c` is highly optimized C code honed over a decade, a full Rust rewrite carries **high engineering effort** (~6 to 9 person-months) and requires meticulous bench testing to ensure zero degradation in glass-to-glass streaming latency.
* **Overall Recommendation:** Adopt a **phased, incremental migration strategy**. Begin with Java-to-Kotlin migration and Rust JNI bridge integration, followed by rewriting modular components (RTSP client, packet parsing, FEC, crypto) into Rust crates while maintaining full binary compatibility.

---

## 1. Current Architecture vs. Target Architecture

```
+-----------------------------------------------------------------------------------+
|                                CURRENT ARCHITECTURE                               |
+-----------------------------------------------------------------------------------+
|  [Android UI & Logic]                                                             |
|   - Java (Activities, Fragments, Views, Preferences, Discovery, Handlers)         |
|   - JNI Bridge (MoonBridge.java -> simplejni.c)                                   |
+-----------------------------------------------------------------------------------+
|  [Native Core Stream Engine]                                                      |
|   - C (moonlight-common-c: RTSP, ENet/UDP, RTP, FEC Reed-Solomon, Crypto, Opus)   |
|   - C++ (cpuaffinity.cpp, evdev_reader)                                           |
|   - C Libraries (libopus, OpenSSL, FidelityFX FSR1)                               |
+-----------------------------------------------------------------------------------+

                                        ||
                                        \/

+-----------------------------------------------------------------------------------+
|                                TARGET ARCHITECTURE                                |
+-----------------------------------------------------------------------------------+
|  [Modern Android App Layer]                                                       |
|   - 100% Kotlin (Coroutines, Flow, Jetpack Compose / ViewBinding, ViewModel)       |
|   - Idiomatic Rust-Kotlin Interop (UniFFI / jni-rs type-safe bindings)            |
+-----------------------------------------------------------------------------------+
|  [Rust Stream Engine Crate (`artemis-core`)]                                      |
|   - Rust (Async Tokio/mio RTSP client, Safe RTP depacketizer, Safe FEC)           |
|   - Safe C FFI bindings for hardware decoders (MediaCodec NDK / FFmpeg / Opus)    |
|   - Multiplatform ready (Android, iOS, Desktop, WebAssembly)                      |
+-----------------------------------------------------------------------------------+
```

---

## 2. Component Feasibility & Complexity Breakdown

| Layer / Component | Current Implementation | Target Stack | Complexity | Estimated Effort | Key Risks & Considerations |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **Android UI & Navigation** | Java (Activities, XML Views, PreferenceActivity) | Kotlin + Jetpack Compose / ViewModel | **Medium** | 4–6 weeks | Preserving touch/controller focus, custom overlays, and foldable/multi-window behaviors. |
| **Async & Threading** | Handlers, Runnable queues, Executors | Kotlin Coroutines + Flow | **Low** | 2–3 weeks | Eliminating memory leaks and race conditions in UI event loops. |
| **JNI Bridge Layer** | `MoonBridge.java` + `simplejni.c` (Raw JNI) | `jni-rs` or `UniFFI` | **Medium** | 3–4 weeks | JNI overhead with high-frequency callbacks (audio PCM, video decode units, rumble/touch events). |
| **RTSP & Control Stream** | C (`RtspConnection.c`, `RtspParser.c`) | Rust (`tokio` / `reqwest` / custom async state machine) | **Medium** | 3–4 weeks | Handling custom Sunshine/Apollo/GameStream RTSP extensions and handshake edge cases. |
| **RTP & Video Depacketizer** | C (`RtpVideoQueue.c`, `VideoDepacketizer.c`) | Rust safe queues & slice manipulation | **High** | 6–8 weeks | Real-time jitter buffer, out-of-order packet reordering, zero-copy pointer transfers to MediaCodec. |
| **FEC / Reed-Solomon** | C (`reedsolomon/` C library) | Rust (`reed-solomon-erasure` or custom C SIMD bindings) | **Medium** | 2–3 weeks | Ensuring parity computation speed matches SIMD C performance (AVX/NEON). |
| **Crypto & Key Exchange** | C (`PlatformCrypto.c` using OpenSSL C API) | Rust (`ring` / `rustls` / `aes` crates) | **Low** | 1–2 weeks | AES-GCM and SHA256 performance verification against hardware-accelerated OpenSSL. |
| **Audio Decoding & Stream** | C (`AudioStream.c`, `libopus`) | Rust (`opus` bindings + AudioTrack JNI) | **Low** | 2 weeks | Maintaining low audio buffer latency (<10ms). |
| **Evdev Direct Reader** | C++ (`evdev_reader`) | Rust (`evdev` crate) | **Low** | 1–2 weeks | Root permissions and Linux event system integration. |

---

## 3. Comprehensive Trade-Off Analysis ("Is It Worth It?")

### A. Latency & Streaming Performance
* **C (Current):** Hand-optimized C code with direct socket polling and zero-copy byte buffers. Extremely low latency (~1–3ms network pipeline latency on local Wi-Fi 6/7).
* **Rust (Target):** Rust compiles to identical LLVM machine code as C/C++. Safe slice indexing and ring buffers in Rust incur **zero additional runtime overhead** when using release mode optimizations (`O3`, `lto = "fat"`, `codegen-units = 1`).
* **Verdict:** Neutral to slight improvement. Rust allows deterministic, allocation-free streaming queues and lock-free data structures (`crossbeam-channel`), preventing intermittent JNI / GC or mutex contention pauses.

### B. Memory Safety & Crash Reduction
* **C (Current):** Buffer overflows, dangling pointers, race conditions during ungraceful stream disconnects, and thread synchronization bugs are common risks in `moonlight-common-c`.
* **Rust (Target):** Rust guarantees **100% memory safety and data-race freedom** at compile time in safe code blocks. Buffer overruns, double-frees, and use-after-free crashes are completely eliminated.
* **Verdict:** **Major Advantage for Rust.** Drastically reduces native crash rates reported in Google Play Console.

### C. Developer Ergonomics & Maintainability
* **Java (Current):** Verbose boilerplate code, null-pointer exception risks, legacy callbacks.
* **Kotlin (Target):** Concise code (~30-40% reduction in LOC), null safety, pattern matching, extension functions, Coroutines for smooth UI & background state handling.
* **Verdict:** **Major Advantage for Kotlin.** Developer velocity increases significantly once migrated.

### D. Build System & CI/CD Complexity
* **Current:** Android Studio + Gradle + `ndk-build` / CMake with C/C++ toolchains.
* **Target:** Gradle + `cargo-ndk` / `uniffi-bindgen`.
* **Verdict:** **Slight Initial Complexity Increase.** Requires configuring Rust targets (`aarch64-linux-android`, `armv7-linux-androideabi`, `x86_64-linux-android`, `i686-linux-android`) in CI pipelines and cross-compilation toolchains.

### E. Cross-Platform Code Reuse Potential
* **Current:** C codebase (`moonlight-common-c`) is shared across Moonlight Android, Moonlight iOS, and Moonlight Embedded.
* **Target:** A clean Rust core crate (`artemis-core`) can be compiled natively into:
  * Android (`.so` via `jni-rs` / `UniFFI`)
  * iOS (`.a` framework via Swift UniFFI)
  * Desktop (Windows, macOS, Linux via C-FFI or direct Rust app)
  * Web (WebAssembly / WASM)
* **Verdict:** **Massive long-term value** if Artemis expands into a multi-platform client ecosystem for Apollo.

---

## 4. Phase-by-Phase Migration Roadmap

```
+-----------------------------------------------------------------------------------+
|  PHASE 0: Preparation & Infrastructure Setup                                      |
|  - Add Kotlin support to Gradle & enable Coroutines / ViewBinding                 |
|  - Integrate `cargo-ndk` plugin & set up multi-target Rust build pipeline         |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
|  PHASE 1: Java -> Kotlin UI & Logic Migration                                     |
|  - Convert UI Activities, Fragments, Preferences & Adapters to Kotlin             |
|  - Refactor background discovery (mDNS, HTTP) to Kotlin Coroutines & Flow          |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
|  PHASE 2: Rust JNI Bridge & Modular C Integration                                 |
|  - Create `artemis-core` Rust crate exposing JNI via `jni-rs` / UniFFI             |
|  - Replace `simplejni.c` with Rust safety wrapper around `moonlight-common-c`    |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
|  PHASE 3: Incremental Native Engine Rewrite (C -> Rust)                           |
|  - Step 3.1: Rewrite Crypto & Key Exchange in Rust (`ring`/`aes`)                  |
|  - Step 3.2: Rewrite RTSP Handshake & SDP Generator in Rust                        |
|  - Step 3.3: Rewrite Control & Input Stream handling in Rust                      |
|  - Step 3.4: Rewrite RTP Audio/Video Depacketizer & FEC Reed-Solomon in Rust      |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
|  PHASE 4: Optimization, Benchmarking & Hardening                                  |
|  - Latency profiling & side-by-side glass-to-glass comparison                     |
|  - Automated NDK memory leak checks & full unit/integration test coverage          |
+-----------------------------------------------------------------------------------+
```

---

## 5. Risk Assessment & Mitigation Matrix

| Risk | Likelihood | Impact | Mitigation Strategy |
| :--- | :---: | :---: | :--- |
| **Increased streaming latency or frame jitter** | Medium | High | Profile Rust depacketizer against C implementation using `Perfetto` and `nanos` timestamp logging. Keep allocation-free lockless channels. |
| **Regressions in custom Artemis/Apollo features** | Medium | High | Implement comprehensive unit tests in Kotlin and Rust for server command protocol, virtual touchpad, and 3D SBS features. |
| **Increased APK size** | Low | Low | Use `cargo build --release` with `panic = "abort"`, `opt-level = "z"`, and `strip = true` to keep Rust `.so` library footprint under 2MB. |
| **Build pipeline failure across NDK ABIs** | Low | Medium | Use containerized CI/CD (GitHub Actions) with `cargo-ndk` pre-installed for arm64-v8a, armeabi-v7a, x86, and x86_64. |

---

## Conclusion & Final Recommendation

Refactoring Artemis Android to **Kotlin + Rust** is a **highly viable and strategically sound decision**.

* **Kotlin Migration** should be executed first; it brings immediate productivity gains, reduces nullability bugs, and modernizes the UI codebase with minimal risk.
* **Rust Migration** should follow an **incremental bottom-up modular strategy**. Rather than a high-risk full rewrite at once, wrapping `moonlight-common-c` in Rust JNI bindings first, then iteratively replacing C modules (Crypto -> RTSP -> Input -> RTP/FEC) guarantees continuous app stability and zero downtime for users.
