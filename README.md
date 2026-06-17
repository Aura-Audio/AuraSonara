# AuraSonara

# 🎛️ **SONARA** — The Self-Hosted, Browser-Based Digital Audio Workstation

*A full product concept, competitive landscape, and market analysis.*

---

## 1. Brand & Naming

### **Project Name: SONARA**
> *From "sonar" (sound navigation) + "aura" (sonic atmosphere). A name that evokes spatial audio, presence, and craft.*

**Tagline:** *"Your studio. Your server. Your sound."*

**Logo Concept:** A minimalist waveform rendered as a rising helix, with a subtle server rack silhouette in the negative space.

**Domain/Handle Candidates:** `sonara.audio`, `sonara.studio`, `getsonara.com`, `@sonarada`

---

## 2. Vision & Positioning

**The Problem:**
Today's DAWs fall into two rigid camps:
- **Desktop DAWs** (Ableton, Logic, FL Studio): Powerful but platform-locked, expensive ($200-$999 upfront), and collaboration-unfriendly.
- **Browser DAWs** (Soundtrap, BandLab): Accessible and collaborative but **fully cloud-hosted SaaS** — your audio lives on someone else's servers, you can't run it offline, and you're subject to account bans, pricing changes, or platform shutdown.

**Sonara's Position:**
A **fully self-hosted, browser-based DAW** you deploy on your own laptop, homelab, NAS, or VPS. Access your studio from any browser on any device. Keep 100% ownership of your audio, plugins, and projects. Zero dependency on SaaS platforms.

**Target Persona:**
- **Prosumers & indie producers** who want DAW power with privacy.
- **Homelab & self-hosting enthusiasts** (the "r/selfhosted" crowd) who currently run Nextcloud, Jellyfin, Immich.
- **Bands & collectives** who want a shared studio without paying per-seat SaaS fees.
- **Podcasters & audio educators** who need on-premise compliance.

---

## 3. Product Architecture

### High-Level Stack

```
┌─────────────────────────────────────────────────┐
│                  Browser Client                  │
│  React + TypeScript + Web Audio API + WASM DSP  │
└──────────────────────┬──────────────────────────┘
                       │ WebSocket / HTTP
┌──────────────────────▼──────────────────────────┐
│              Sonara Core Server                  │
│        Go (high-perf) or Node.js backend         │
│  ┌─────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │ Auth    │ │ Session  │ │ Render Engine    │  │
│  │ (OAuth/ │ │ Manager  │ │ (FFmpeg + libs)  │  │
│  │  OIDC)  │ │          │ │                  │  │
│  └─────────┘ └──────────┘ └──────────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│              Storage Layer                       │
│   Local FS │ MinIO (S3-compat) │ Postgres/SQLite│
└─────────────────────────────────────────────────┘
```

### Deployment

```yaml
# docker-compose.yml (one-command deploy)
version: '3.8'
services:
  sonara:
    image: sonara/studio:latest
    ports:
      - "8080:8080"
    volumes:
      - ./projects:/data/projects
      - ./samples:/data/samples
      - ./plugins:/data/plugins
    environment:
      - SONARA_AUTH=selfhosted  # or ldap, oidc
```

One `docker compose up` and the user hits `http://localhost:8080`. Helm chart available for Kubernetes. Deployable on GCP/AWS EC2, DigitalOcean, Hetzner, Raspberry Pi, old laptop — anywhere Linux runs.

---

## 4. Feature Set

### 🟢 Tier 1 — Core (Free, Open-Source, AGPLv3)

| Category | Features |
|---|---|
| **Timeline & Arrangement** | Unlimited tracks (audio, MIDI, instrument, bus), non-linear clip editing, elastic audio (time-stretch), grid/snap, markers |
| **Mixer** | Channel strips with insert chains, sends/returns, VCA groups, stereo/mono, pan, mute/solo, metering (LUFS, RMS, peak) |
| **Recording** | Browser mic/line-in capture via `getUserMedia`, multi-channel support, punch-in/out, loop recording |
| **Editing** | Waveform zoom, split, trim, crossfade, comping, batch normalize, destructive & non-destructive |
| **Virtual Instruments** | Built-in WASM synths (subtractive, FM, wavetable), sampler (SFZ/AKAI import), drum machine |
| **Effects** | 30+ stock plugins (EQ, compressor, limiter, reverb, delay, chorus, flanger, bitcrusher, gate) — all WASM-compiled for zero-latency |
| **Export** | WAV, FLAC, MP3, OGG. Full mixdown or stems. Background render queue |
| **Project Management** | Local project files, automatic versioning, import/export (.sonara format + AAF/XML for interop) |
| **Sample Library** | Drag-drop import, tag-based browser, bundled CC0 starter loops (~2GB) |
| **MIDI** | Web MIDI API support, hardware controller mapping, piano roll, drum editor, notation view |

### 🟡 Tier 2 — Sonara+ (Commercial, $79 one-time or $49/yr)

| Feature | Description |
|---|---|
| **Real-time Collaboration** | WebRTC-based, multiple users editing the same session simultaneously with cursor presence |
| **VST/CLAP Hosting** | Run Windows/Mac VST3 and Linux CLAP plugins on the server, stream audio to browser via low-latency protocol |
| **AI Mastering Engine** | On-premise AI (no cloud upload) for mastering suggestions, loudness targeting, stem separation (Demucs WASM) |
| **Video-to-Audio** | Import video files, score to picture, ADR workflow |
| **Advanced Automation** | Parametric curves, LFO generators, MIDI CC mapping |
| **Hardware Control Surfaces** | Native support for Mackie Control, OSC, HUI protocols |

### 🔴 Tier 3 — Enterprise (Custom pricing)

- Multi-tenant workspaces with RBAC
- LDAP/SSO integration
- Audit logs, asset watermarking
- Air-gapped deployment (airlines, defense, film studios)
- White-label option for music schools

---

## 5. Technical Deep-Dive: Why This Was Hard Until Now

The reason no browser DAW has been truly self-hosted until 2026:

1. **Web Audio API latency** was too high pre-2023. With `AudioWorklet` + `SharedArrayBuffer` now broadly supported, sub-10ms round-trip is achievable.
2. **DSP in JavaScript** was slow. Compiling C++ DSP libraries (like JUCE, FAUST, or custom) to **WebAssembly** now delivers near-native performance.
3. **WASM multithreading** now enables real-time plugin chains in the browser without blocking the UI.
4. **WebCodecs + WebTransport** allow low-latency streaming of server-rendered audio (for heavy VSTs) to the browser client.

---

## 6. Competitive Landscape

### 6.1 Direct Competitors — Browser-Based DAWs

| Product | Hosting | Pricing | Collaboration | Privacy/Self-Host |
|---|---|---|---|---|
| **Soundtrap** (Spotify) | Cloud SaaS | Free → $14.95/mo [[28]] | ✅ Real-time | ❌ Spotify servers |
| **BandLab** | Cloud SaaS | Free + $14.99/mo tier [[41]] | ✅ Social | ❌ BandLab servers |
| **Soundation** | Cloud SaaS | Free → $9.99/mo | ✅ Async | ❌ Cloud only |
| **Amped Studio** | Cloud SaaS | Free → $11.99/mo | ✅ Real-time | ❌ Cloud only |
| **Audiotool** | Cloud SaaS | Free | ✅ Social | ❌ Cloud only |
| **Veena Studio** | Cloud SaaS | Free + paid | ✅ | ❌ Cloud only |
| **LA Studio** | Cloud SaaS | Free + paid | ✅ | ❌ Cloud only |
| **🟦 Sonara** | **Self-hosted** | **Free (OSS) + paid tiers** | **✅ Real-time** | **✅ Full ownership** |

### 6.2 Desktop DAWs (Indirect Competitors)

| Product | Price | Collaboration | Self-Host |
|---|---|---|---|
| Ableton Live | $99-$749 | ❌ Limited | ✅ (local) |
| FL Studio | $99-$899 | ❌ | ✅ |
| Logic Pro | $199 (Mac only) | ❌ | ✅ |
| Pro Tools | $29.99/mo | ✅ | ✅ (with HDX) |
| Reaper | $60-$225 | ❌ | ✅ |
| Bitwig | $99-$499 | ❌ | ✅ |

### 6.3 Open-Source & Self-Hosted Audio Ecosystem

| Product | Type | Limitation |
|---|---|---|
| **Audacity** | Audio editor | Not a DAW — no MIDI, instruments, or mixer [[25]] |
| **Ardour** | Open-source DAW | Desktop only, steep learning curve, no browser |
| **LMMS** | Open-source DAW | Desktop only, dated UI |
| **Airsonic / Navidrome / Ampache** | Self-hosted streaming | Playback only — no creation [[21]] |
| **Jamulus / JackTrip** | Real-time jamming | No recording/production features |
| **Sonic Pi** | Live coding music | Not a DAW |

### 🎯 The Gap
**Sonara is the first product to combine browser-based UX + full DAW capability + true self-hosting.** The "Nextcloud of audio production."

---

## 7. Market Sizing

### Total Addressable Market (TAM)

- The **global DAW market** was valued at approximately **$3.2–$4.4 billion in 2025** [[1]][[10]].
- Projected to reach **$6.1–$9.1 billion by 2032–2034** at a CAGR of ~6.7–9.1% [[1]][[6]].
- The **self-hosting market** in the US alone is worth **$4.62 billion** with a CAGR of 16.2% [[45]].
- The **self-hosted cloud platform market** globally hit **$18.48 billion in 2025**, projected to reach **$46.10 billion by 2033** [[48]].

### Serviceable Addressable Market (SAM)

Targeting the intersection of:
- Prosumer & indie musicians willing to pay for tools
- Self-hosting enthusiasts
- Small studios and educators

Estimated SAM: **~$600M** (browser + self-hosted + prosumer subset of DAW market)

### Serviceable Obtainable Market (SOM)

Realistic 3–5 year capture:

| Year | Assumptions | ARR Target |
|---|---|---|
| Y1 | 5,000 free users, 500 paid ($79) | $39K |
| Y2 | 25,000 free, 3,500 paid | $276K |
| Y3 | 100,000 free, 15,000 paid + 20 enterprise deals | $1.8M |
| Y5 | 500,000 free, 60,000 paid + 200 enterprise | **$7.2M** |

**SOM at maturity ≈ 1.5% of SAM = $9M ARR**, achievable with strong open-source community growth and word-of-mouth in the homelab/self-hosted ecosystem.

---

## 8. Revenue Model

| Stream | Description | Pricing |
|---|---|---|
| **Open Core** | AGPLv3 community edition | Free forever |
| **Sonara+ License** | Commercial unlock (VST hosting, AI mastering, collab) | $79 one-time (personal) / $199 (commercial) |
| **Annual Subscription** | Same features + updates + support | $49/yr personal / $149/yr commercial |
| **Enterprise** | Multi-seat, SSO, SLAs, white-label | $299/seat/yr |
| **Marketplace** | 30% commission on user-submitted sample packs, plugins, presets | Revenue share |
| **Cloud Hosting (optional)** | For users who want managed Sonara on a VPS | $15/mo per instance |

*Note: Unlike BandLab, which currently doesn't directly make money [[36]], Sonara has a sustainable direct-revenue model.*

---

## 9. Go-to-Market Strategy

### Phase 1 — Open-Source Launch (Months 1–6)
- **GitHub release** with polished README, demo videos, one-click installers
- **Reddit**: r/selfhosted, r/WeAreTheMusicMakers, r/BedroomBands, r/homelab
- **Hacker News**: Show HN launch
- **Lobsters & Product Hunt**: Secondary channels
- **Seed influencers**: Send pre-release to self-hosting YouTubers (e.g., NetworkChuck, Techno Tim, Craft Computing) and indie music producers

### Phase 2 — Community Flywheel (Months 6–18)
- Launch **Sonara Hub** — plugin/sample marketplace
- Host **monthly remix challenges** (CC0 stems, prizes)
- Build **integrations**: Spotify for Artists, DistroKid, SoundCloud upload
- Create **education tier**: free licenses for universities

### Phase 3 — Commercial Scale (Months 18–36)
- Enterprise sales for music schools, post-production houses
- OEM licensing for hardware manufacturers (audio interfaces)
- Conference presence (NAMM, Superbooth, Self-Hosted Conf)

---

## 10. Three-Year Product Roadmap

| Quarter | Milestone |
|---|---|
| **Q1–Q2 2026** | Core DAW (audio tracks, mixer, 15 effects, export) — open-source beta |
| **Q3 2026** | MIDI + piano roll, built-in synths, sampler engine |
| **Q4 2026** | Real-time collaboration via WebRTC |
| **Q1 2027** | VST3/CLAP hosting (server-side), marketplace launch |
| **Q2 2027** | AI stem separation + mastering (Demucs-based, local) |
| **Q3 2027** | iOS/Android companion apps for remote control |
| **Q4 2027** | Video sync & scoring, ADR mode |
| **Q1 2028** | Enterprise tier, multi-tenant, white-label |
| **Q2 2028** | Plugin SDK — third-party WASM plugin development |
| **2028 H2** | Spatial audio (Dolby Atmos) mixing in-browser |

---

## 11. Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Browser performance ceiling** | Heavy DSP offloaded to server via WASM + WebTransport streaming |
| **Web Audio API fragmentation** | Polyfills + audio worklet abstraction layer |
| **Plugin licensing** | Start with open-source/C++ DSP; add VST licensing via Steinberg SDK agreements |
| **Open-source competitors forking** | Strong community governance, CLA, commercial license for embedders |
| **Browser DAWs adding self-host** | First-mover advantage + deep self-hosting DNA (LDAP, MinIO, K8s) |

---

## 12. Why Now?

Three technological waves have converged in 2025–2026:

1. **WebAssembly maturity** — near-native DSP performance in browsers
2. **Self-hosting renaissance** — the privacy-first movement is mainstream (Nextcloud, Immich, Jellyfin all 100K+ stars)
3. **Remote collaboration demand** — post-pandemic, producers need to work together across timezones

**Sonara sits at the intersection of all three.** It is the DAW that the self-hosting generation has been waiting for — the missing piece in the homelab audio stack where streaming (Navidrome) and library management (Beets) exist, but *creation* has always been missing.

---

*"Your music. Your machine. Your masterpiece."* — **Sonara**
