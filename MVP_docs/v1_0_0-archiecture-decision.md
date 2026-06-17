# 🎛️ **Architecture Decision: FAUST vs. Rust/WASM for DSP**

**The Short Answer:**
For the SONARA MVP, **a hybrid approach (Combination)** is not just beneficial—it is the **optimal architectural choice**.

Using **Rust/WASM** for the Audio Engine (transport, graph routing, file I/O) and **FAUST/WASM** for the DSP algorithms (EQ, compression, reverb) gives you the "best of both worlds." Attempting to write complex, musical DSP algorithms from scratch in raw Rust will severely bottleneck your MVP timeline.

Here is the deep-dive technical breakdown of why this combination wins, and how to implement it.

---

## 1. The Roles: Conductor vs. Musician

To understand why the hybrid approach works, we must separate the DAW into two distinct domains:

### 🦀 **Rust/WASM: The Conductor (The Engine)**
Rust is a general-purpose systems language. It is phenomenal at memory management, concurrency, and complex logic.
*   **Use it for:** Clip scheduling, timeline playback, mixing bus routing, WAV file decoding, state management, and bridging the UI (React) to the AudioWorklet.
*   **Why not DSP?** Writing a mathematically stable, alias-free Parametric EQ or a lush Algorithmic Reverb from scratch in Rust requires deep DSP engineering knowledge and hundreds of lines of complex math.

### 🎻 **FAUST/WASM: The Musician (The DSP)**
FAUST (Functional Audio Stream) is a Domain-Specific Language (DSL) built *exclusively* for audio signal processing. It compiles down to highly optimized C++ and then to WebAssembly.
*   **Use it for:** The actual audio processing nodes (Synths, EQs, Compressors, Delays, Reverbs).
*   **Why FAUST?** A 3-band parametric EQ in Rust might take 300 lines of complex boilerplate and filter design. In FAUST, it is literally **10 lines of code**. FAUST handles the complex calculus, anti-aliasing, and filter stability automatically.

---

## 2. The Hybrid Architecture for SONARA

Here is how Rust and FAUST will coexist inside the browser's `AudioWorklet`:

```text
┌────────────────────────────────────────────────────────┐
│                 Browser AudioWorklet                   │
│                                                        │
│  ┌────────────────────┐    ┌───────────────────────┐   │
│  │   Rust WASM Core   │    │    FAUST WASM Nodes   │   │
│  │                    │    │                       │   │
│  │ • Clip Scheduler   │    │ • 3-Band EQ           │   │
│  │ • Transport/Tempo  │    │ • FET Compressor      │   │
│  │ • Graph Routing    │    │ • Freeverb Reverb     │   │
│  │ • WAV Decoder      │    │ • Tape Delay          │   │
│  └─────────┬──────────┘    └───────────┬───────────┘   │
│            │                           │               │
│            └───────── Audio Bus ───────┘               │
│                    (SharedArrayBuffer)                 │
└────────────────────────────────────────────────────────┘
```

### How they communicate:
1.  **The FAUST Compiler:** You write your effects in `.dsp` (FAUST code). The FAUST compiler outputs C++ code, which is then compiled to WebAssembly (using `emscripten` or the official `faustwasm` toolchain).
2.  **The Rust Host:** Your Rust WASM module acts as the "Host" (like a VST host). It manages the audio graph. When a user adds an EQ to Track 1, Rust instantiates the FAUST WASM module for that EQ and routes the audio buffer through it.
3.  **Alternative (Easier for MVP):** Use the official `faustwasm` library's `FaustWorklet` for the effects, and use a separate Rust WASM module purely for the offline rendering/export engine and complex UI state management.

---

## 3. Impact on the MVP Timeline (Phase 4)

In the original 12-month roadmap, **Phase 4 (DSP Effects)** was allocated 8 weeks. If you use raw Rust, 8 weeks is barely enough to build *one* good compressor and EQ.

**With FAUST, Phase 4 is compressed from 8 weeks to 3 weeks:**

| Effect | Raw Rust/WASM Effort | FAUST/WASM Effort |
| :--- | :--- | :--- |
| **3-Band Parametric EQ** | ~2 weeks (Math, Biquad filter design, UI mapping) | **2 hours** (Use `fi.resonlp` from FAUST standard library) |
| **Compressor** | ~3 weeks (Envelope follower, knee math, attack/release curves) | **1 day** (Use `fi.compressor` from FAUST library) |
| **Algorithmic Reverb** | ~3 weeks (Feedback delay networks, diffusion) | **2 days** (Use `ve.freeverb` or `ve.zita_rev`) |
| **Stereo Delay** | ~1 week (Buffer management, feedback loops) | **4 hours** (Use `de.echo`) |

**Result:** You hit your MVP feature set faster, with mathematically proven, alias-free DSP algorithms.

---

## 4. The Open-Source "Flywheel" Benefit

One of SONARA's core pillars is being a community-driven open-source project. FAUST provides a massive advantage here:

*   **Lower Barrier to Entry:** Finding a Rust developer who *also* understands digital signal processing (DSP) is incredibly difficult. However, there are thousands of audio programmers, musicologists, and hobbyists who know FAUST or Max/MSP/PureData.
*   **Community Contributions:** If a user wants to add a "Vintage Tube Saturation" effect to SONARA, they don't need to learn your Rust codebase. They just write a 20-line FAUST script, compile it to WASM, and drop it into the SONARA `/plugins` folder.
*   **Documentation:** The FAUST standard library is heavily documented and widely used in academia (CCRMA, IRCAM). You can lean on existing research and code snippets.

---

## 5. Potential Pitfalls & Mitigations

While the hybrid approach is superior, it introduces complexity that must be managed:

### ⚠️ Pitfall 1: Build Pipeline Complexity
*   **The Issue:** You now have two compilation targets. Rust needs `wasm-pack`, and FAUST needs the FAUST compiler + `emscripten`/`wasm` target.
*   **Mitigation:** Create a robust `Makefile` or a `Turborepo` pipeline in Phase 0. Automate the FAUST-to-WASM compilation so that when a developer changes a `.dsp` file, it automatically rebuilds the WASM node.

### ⚠️ Pitfall 2: Parameter Automation (UI to DSP)
*   **The Issue:** Moving a knob in the React UI needs to update the parameter in the FAUST WASM node in the AudioWorklet without causing audio "zipper noise" (clicks/pops).
*   **Mitigation:** FAUST has built-in support for UI variables (e.g., `freq = hslider("freq", 1000, 20, 20000, 0.01);`). You must implement **parameter smoothing** (linear interpolation) inside the AudioWorklet so that when React sends a new value, the FAUST node glides to it over ~10ms rather than jumping instantly.

### ⚠️ Pitfall 3: Polyphony and Voices (For future synths)
*   **The Issue:** FAUST is inherently mono/sterile. Building a 16-voice polyphonic synth in FAUST requires specific architectural flags (`-nvoices`).
*   **Mitigation:** For the MVP, stick strictly to **audio effects** (which are stereo-in/stereo-out). Defer polyphonic virtual instruments to v1.1 when you have more time to architect the MIDI-to-FAUST voice allocation layer.

---

## 6. Final Verdict for the MVP

**Adopt the Hybrid Architecture.**

*   **Use Rust/WASM** to build the **DAW Engine**: The timeline, the transport, the clip launcher, the WAV importer, and the offline bounce engine.
*   **Use FAUST/WASM** to build the **Studio Rack**: The EQs, Compressors, Delays, and Reverbs.

This division of labor respects the strengths of both technologies, guarantees professional-grade audio quality, drastically shortens your time-to-market for the MVP, and sets up a highly accessible plugin ecosystem for your future open-source community.
