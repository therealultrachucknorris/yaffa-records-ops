# Maccabi House Plugins — How We Build VSTs with Claude

**Status:** Research and plan, draft v1 (25 Nov 2026)
**Owner:** Mosko (Yaffa Records)
**Partner:** Adam Ten / Maccabi House (approval given, terms still to be written down; see §9)

---

## 1. What we're making

A small line of **Maccabi House–branded audio plugins** that producers load in their DAW (Ableton, Logic, FL, Bitwig, Cubase). They should carry the Maccabi House sound, so a producer gets that vibe in one or two knobs.

"VST" in this doc means any plugin format:

| Format | Where it runs | Needed? |
|---|---|---|
| **VST3** | Ableton, FL, Cubase, Bitwig, Reaper (Win + Mac) | **Yes**, the main format |
| **AU** (Audio Unit) | Logic, GarageBand, and Ableton on Mac | **Yes**, most house producers on Mac use Logic or Ableton |
| **CLAP** | Bitwig, Reaper, FL (newer versions) | Nice to have, nearly free with the stack below |
| **AAX** | Pro Tools | **No** (for now). Needs an Avid dev account and PACE signing, and few house producers use Pro Tools |

---

## 2. TL;DR: the recommendation

1. **Stack:** **JUCE 8 (C++) + CMake**, starting from the open-source **Pamplejuce** template, which comes with CMake, tests, and GitHub Actions builds for Mac and Windows. Add **clap-juce-extensions** for CLAP.
2. **Claude's role:** Claude Code writes almost all of the code: DSP, parameters, UI, tests, the build pipeline, and installers. **We** set the sound direction, listen, and decide. Claude can't hear audio, so our ears, and Adam's, are the quality gate.
3. **Where builds happen:** GitHub Actions builds signed Mac and Windows installers on every release. Day-to-day listening and tweaking happen on a Mac running Claude Code locally with a DAW open.
4. **First product:** one **simple effect** (e.g. a one-knob "Maccabi" character/saturation or a groove filter), free or cheap, to prove the whole pipeline: code → sound → signed installer → shop → customer. Then build the bigger products.
5. **Time:** about 2–4 weeks for the first free plugin end to end, and about 2–3 months for a first paid bundle. This assumes a few focused hours a week.

---

## 3. How a plugin works (the 2-minute version)

Every plugin has the same four parts, whatever the framework:

1. **Audio processor (DSP).** A function the DAW calls hundreds of times a second with a small block of audio (e.g. 256 samples). It changes the samples and hands them back. **Hard rule:** this code must never block, allocate memory, or touch files or the network. If it does, the user hears clicks and dropouts.
2. **Parameters.** The knobs (e.g. "Drive", "Mix"). The DAW can automate them and saves them with the project.
3. **State and presets.** Save and recall of all settings, plus factory presets. This is a great place for "Adam Ten presets".
4. **Editor (GUI).** The window with the knobs. It's optional for the sound, but it carries the brand.

There are two types of plugin:
- **Effect:** audio in → audio out (EQ, saturation, delay, compressor, filter).
- **Instrument:** MIDI in → audio out (synth, drum machine, sampler). Instruments are much more work: voices, envelopes, presets, and often sample libraries.

**Start with effects.**

---

## 4. Framework options

| Option | Language | Licence / cost | Claude-friendliness | Verdict |
|---|---|---|---|---|
| **JUCE 8** | C++ | Free **Starter** tier (no splash screen) up to a revenue/funding cap. **Indie** reported at ~$40/mo or ~$800 perpetual, **Pro** ~$175/mo. Check juce.com for current numbers | ★★★★★ The industry standard, with a huge amount of public code. Claude knows it very well | **Use this** |
| **VST3 SDK** (raw) | C++ | **MIT since v3.8 (Oct 2025)**: free, closed-source OK, no Steinberg agreement needed | ★★★ Verbose and low-level. JUCE wraps it for us | Comes inside JUCE |
| **nice-plug** (fork of nih-plug) | Rust | ISC, but its VST3 bindings are **GPLv3**, which is a problem for closed-source commercial VST3 | ★★★★ | Skip for commercial |
| **iPlug2** | C++ | Permissive (zlib-style) | ★★★ Smaller community | Backup option |
| **HISE** | Visual + scripting | GPL or commercial licence | ★★ Mostly GUI-driven, so less of a fit for Claude | Good for sample instruments, not our first choice |
| **Cmajor / Faust** | DSP languages | Various | ★★★ | Useful for prototyping DSP ideas only |

**Why JUCE:** it builds VST3, AU and standalone from one codebase (CLAP via an extension). It has ready DSP blocks (filters, oversampling, convolution, compressors). And it's what Claude has seen the most of, which means fewer hallucinated APIs and faster progress.

**Licence note:** the VST3 SDK becoming MIT in Oct 2025 removed the old Steinberg licence-agreement step. We can ship closed-source commercial VST3s without signing anything. Using the **VST logo** is optional, and if we use it we must follow Steinberg's trademark guidelines.

---

## 5. How we work with Claude

### 5.1 What Claude is good and bad at here

| Claude does well | We must do |
|---|---|
| JUCE boilerplate, parameters, presets, state save/load | Decide what it should **sound** like (reference tracks!) |
| Standard DSP: filters, saturation curves, delays, compressors, oversampling | **Listen** to every change in a real DAW, A/B against references |
| GUI layout from a mockup or description | Provide or approve the **visual design** (artwork, knob style, logo) |
| CMake, GitHub Actions, code signing scripts, installers | Own the Apple/Windows signing accounts and the secrets |
| Unit tests, running `pluginval`, fixing crashes it reports | Test in each target DAW (Ableton, Logic, FL) on real machines |
| Writing the manual, product page copy, changelogs | Final sign-off (us + Adam) |

**Golden rule:** Claude can measure audio with tests (e.g. "output never exceeds 0 dBFS", "no NaNs", "a 1 kHz sine at Drive=0 comes out unchanged") but **can't judge whether it sounds good**. Every feedback loop needs a human listening.

### 5.2 Setup

- **Claude Code on a Mac (local)** is the main dev environment, because only a Mac can build and test AU in Logic. Install Xcode, CMake, Ninja, a DAW, and `pluginval` (Tracktion's free plugin validator).
- **Windows:** GitHub Actions builds it for us. Someone with a Windows PC tests it in FL and Ableton before each release.
- **Claude Code in the cloud** (like this session) runs on Linux. It's fine for writing code, the docs, and the CI pipeline, and it can build a Linux VST3 to run tests. It **cannot** produce Mac/Win builds or play audio. CI handles builds.
- **One repo per product line**, e.g. `maccabi-house-plugins` (monorepo with one folder per plugin and a shared `MaccabiLookAndFeel` for the brand UI). This would be a separate repo, not this ops repo.

### 5.3 The workflow, step by step

1. **Spec first (no code yet).** For each plugin, write a one-page spec: purpose, 3–5 reference tracks from the Maccabi House catalog, the controls (name, range, default), presets, and what "done" sounds like. Claude can draft this from a voice note or a chat with Adam.
2. **`CLAUDE.md` in the repo.** This file tells every Claude session the rules: JUCE version, real-time safety rules (no allocations or locks in `processBlock`), naming, "run tests + pluginval before saying done", and the brand UI guidelines.
3. **Scaffold.** Ask Claude to set up Pamplejuce with the plugin name, company "Maccabi House", formats VST3/AU/CLAP/Standalone, and our bundle IDs.
4. **DSP in small steps.** Have Claude implement one block at a time (e.g. "add a tilt EQ before the saturator"), each with a unit test. Build, load in the DAW, listen, then give feedback in **plain words** ("too harsh above 8k", "needs more low-end weight at higher drive").
5. **Parameters and presets.** Once the sound is right, lock parameter IDs (changing them later breaks users' saved projects) and build the factory presets with Adam.
6. **GUI.** Give Claude the artwork (Figma export or PNGs), and it builds the editor. Keep it resizable and HiDPI-sharp.
7. **Validate.** Run `pluginval` at strictness 10, test in Ableton, Logic and FL on Mac and Win, check CPU use, test automation, and test save/reload.
8. **Ship.** Tag a version, CI builds signed installers, and we upload them to the shop.

### 5.4 Helpful Claude add-ons (review before use)

- **Community Claude Code skills** for JUCE exist, e.g. `juce-plugin` (on skills.sh) and a `cross-platform-builds` skill covering CMake, signing, notarization and CI. They're useful as a starting point, but read them first: they're third-party.
- **Audio Plugin Coder (APC)** by Noizefield (launched Feb 2026) is an open-source JUCE 8 framework built for "vibe-coding" plugins with Claude Code. It has `/dream → /plan → /design → /implement → /ship` commands. It's worth a test run for a prototype, but I haven't verified its quality, so we shouldn't depend on it for the commercial product.
- Our own **project skill** later: once the first plugin ships, we capture our exact workflow (spec template, test checklist, release steps) as a skill in the plugins repo.

---

## 6. Product ideas (to decide with Adam)

These are starting points, not decisions. The best ideas will come from how Adam actually makes his records.

| Idea | Type | Difficulty | Why it fits |
|---|---|---|---|
| **Maccabi Drive**: one-knob character saturation + tilt EQ | Effect | ★ Easy | Perfect first product. Fast to build, easy to give away as a lead magnet |
| **Maccabi Filter**: musical resonant filter with LFO/groove sync | Effect | ★★ | Filter sweeps are central to house builds |
| **Maccabi Groove**: swing/shuffle + humanize MIDI tool | MIDI effect | ★★ | Captures a producer's "feel". VST3 MIDI effects are fiddly in some DAWs, so needs testing |
| **Maccabi Space**: delay + reverb tuned for the Maccabi sound | Effect | ★★★ | Bigger product, needs a lot of listening work |
| **Maccabi Bus**: drum-bus glue (compressor + clipper + punch) | Effect | ★★★ | High value for producers. Hard to make sound *better* than the competition |
| **Maccabi Drums / Keys**: sample-based instrument | Instrument | ★★★★ | Needs recorded/licensed samples, and Adam's sounds are the moat. Do it later |

**Suggested order:** Drive (free) → Filter (paid) → bundle with Groove → bigger products once we know what sells.

---

## 7. Distribution and selling

### 7.1 Code signing (mandatory, or users see scary warnings)
- **macOS:** Apple Developer Program (**$99/year**). Sign with a *Developer ID* certificate and **notarize** with Apple, or macOS blocks the plugin. CI does this automatically once the secrets are set.
- **Windows:** an Authenticode code-signing certificate, or Microsoft's cloud signing service (cheaper and works in CI). Without it, SmartScreen warns users.

### 7.2 Installers
- **Mac:** a signed `.pkg` that puts AU/VST3/CLAP in the right folders (built with `pkgbuild`/`productbuild` in CI).
- **Windows:** an Inno Setup `.exe` installer.
- Pamplejuce's CI already does most of this. Claude adapts it.

### 7.3 Licensing / copy protection
Options, from simplest to heaviest:
1. **None or honor-system** for free plugins. Collect the email at download.
2. **Serial or licence key** via a service like Moonbase or Keyzy. Plugin-focused, with JUCE integrations.
3. **iLok/PACE**: expensive and overkill for us.

**Recommendation:** start with free + email gate, then Moonbase-style keys for paid products.

### 7.4 Where to sell
- **Own shop:** Yaffa Records / Maccabi House site (e.g. Shopify digital products), plus the **Brevo email list we already run**. Hypeddit download gates already work for free releases, and could work for the free plugin too.
- **Marketplaces later:** Plugin Boutique, KVR Marketplace, ADSR. Good reach, but they take a cut.

---

## 8. Roadmap

| Phase | What | Output | Rough time |
|---|---|---|---|
| **0. Agreement** | Write down terms with Adam (see §9). Pick the first product | Signed one-pager | 1 week |
| **1. Setup** | Plugins repo, Mac dev machine, Apple dev account, Pamplejuce scaffold, CI green | Empty plugin builds and installs on Mac + Win | 1 week |
| **2. Maccabi Drive** | Spec → DSP → listening rounds with Adam → GUI → presets | v1.0 free plugin | 2–3 weeks |
| **3. Launch** | Signed installers, download page, email gate, launch content | Public release + email list growth | 1 week |
| **4. Paid product** | Maccabi Filter (+ Groove), licensing, shop | First revenue | 4–8 weeks |
| **5. Learn and expand** | Feedback, updates, bundle, marketplaces | Product line | Ongoing |

---

## 9. Business and legal checklist (before we sell anything)

Adam's approval is the start. Before launch, put it **in writing** (not legal advice; have a lawyer look at the final version):

- [ ] **Brand use:** permission to use "Maccabi House" name/logo and Adam Ten's name, plus approval rights over product, artwork and marketing
- [ ] **Revenue split:** % to each side, on gross or net, after which costs (store fees, signing, JUCE licence)
- [ ] **Ownership:** who owns the code (suggest: Yaffa), and who owns presets and samples Adam provides
- [ ] **Term and exit:** what happens to sold copies and updates if the partnership ends
- [ ] **Support:** who answers customer emails and handles refunds
- [ ] **JUCE licence tier:** stay within the Starter limits, or budget for Indie once revenue or funding grows
- [ ] **Trademarks:** check "Maccabi House" for plugin/software use. Follow the Steinberg VST and Apple AU guidelines if we show their logos

---

## 10. Costs (year one, rough)

| Item | Cost |
|---|---|
| JUCE Starter | $0 (until the revenue cap; then Indie ~$40/mo or ~$800 one-time) |
| VST3 SDK | $0 (MIT) |
| Apple Developer Program | $99 / year |
| Windows code signing | ~$10/mo cloud signing, or ~$200–400/yr for a certificate |
| Claude (Pro/Max plan for Claude Code) | Our existing plan, or a Max plan if we code a lot |
| Licensing service (Moonbase etc.) | Free tier → % of sales |
| GitHub Actions | Free tier is likely enough; Mac runners eat minutes faster |
| Artwork / UI design | $0 if in-house; otherwise a designer for the brand UI kit |
| **Total to launch the free plugin** | **≈ $100–300** plus our time |

---

## 11. Risks and how we handle them

| Risk | Mitigation |
|---|---|
| Plugin sounds "generic" | Reference tracks + Adam in every listening round. Ship nothing he hasn't signed off |
| Crashes a user's DAW (reputation hit) | `pluginval` in CI, real-time-safety rules in `CLAUDE.md`, beta testers before launch |
| Mac/Win build hell | Start from Pamplejuce. CI from day one |
| Claude produces confident but wrong DSP | Unit tests on every DSP block + listening. Keep changes small and reviewable |
| Scope creep (instrument first) | Effects first. One plugin shipped beats three half-done |
| Partnership misunderstanding | §9 checklist signed before launch |

---

## 12. Decisions (updated 27 Nov 2026)

| # | Question | Decision |
|---|---|---|
| 1 | First plugin | **Maccabi Drive**, leaning yes. We're approved to use the artist's name (Adam Ten) on products. **Omer's feedback pending.** |
| 2 | Dev and listening machines | **Mosko's and Omer's Macs** |
| 3 | Visual design | **Mosko + Claude together.** Rule: **the plugin UI is written in code and rendered as vectors**, never bitmaps. Higgsfield is OK for mood boards and marketing visuals only |
| 4 | Split and ownership with Adam | **Open**: draft the one-pager (§9) |
| 5 | Plugins repo | **Yes**: `maccabi-house-plugins` |

### Status: `maccabi-house-plugins` live (private repo, PR #1, builds pass on macOS, Windows and Linux)
- JUCE 9.0.6 + CMake. Builds **VST3, CLAP, Standalone** (AU on Mac).
- **Maccabi Drive v0.1**: hero Drive knob plus Tone (tilt), Mix, Output. 4× oversampling, latency-aligned dry path, smoothed parameters.
- **Vector UI kit** (`shared/ui/`): "Night in Jaffa" palette, sunset emblem, wordmark, custom knobs, all drawn in code. A `RenderUI` tool renders the UI to PNG without a DAW.
- **11 unit tests pass**: silence, bounded output, tone tilt, smoothing, latency alignment, bypass, state recall, parameter IDs.
- GitHub Actions for Linux, macOS and Windows (+ pluginval and auval). Signing and notarization are TODOs.
- `CLAUDE.md` with the rules (real-time safety, parameter IDs are forever, vectors only, "Claude can't hear").

- Design v2: embedded Barlow fonts, preset bar with 7 factory presets (placeholders until we've listened and Adam signs off), heat meter, hover states.

**Next:** Omer's go/no-go → first listening round on both Macs → tune `DriveVoicing` → Adam's sign-off.

---

## Sources

- Steinberg VST 3 licensing (MIT since 3.8): https://steinbergmedia.github.io/vst3_dev_portal/pages/VST+3+Licensing/VST3+License
- VST 3.8.0 release notes: https://steinbergmedia.github.io/vst3_dev_portal/pages/Versions/Version+3.8.0.html
- Sound on Sound, "Steinberg adopt MIT license for VST3": https://www.soundonsound.com/news/steinberg-adopt-mit-license-vst3
- Libre Arts, VST3 relicensing: https://librearts.org/2025/11/steinberg-relicenses-vst3-and-asio/
- JUCE 8 release and pricing tiers (Sound on Sound): https://www.soundonsound.com/news/juce-8-development-platform-released
- JUCE 8 EULA amendments (JUCE forum): https://forum.juce.com/t/amendments-to-the-juce-end-user-licence-agreement-for-juce-8/61265
- JUCE basic plugin tutorial: https://docs.juce.com/master/tutorial_create_projucer_basic_plugin.html
- NIH-plug (maintenance mode, GPLv3 VST3 bindings note): https://github.com/robbert-vdh/nih-plug
- Audio Plugin Coder (APC) launch: https://www.noizefield.com/news/audio-plugin-coder-apc-ai-launch?amp=1
- Attack Magazine, "you can now build your own plugins without C++ knowledge": https://www.attackmagazine.com/news/you-can-now-build-your-own-plugins-without-c-knowledge/
- `juce-plugin` Claude skill: https://www.skills.sh/shortwavlabs/agent-skills/juce-plugin
- `cross-platform-builds` Claude skill: https://claude-plugins.dev/skills/@yebot/rad-cc-plugins/cross-platform-builds

*Not checked against a live source in this round (from general knowledge; confirm before relying on it): Pamplejuce template, clap-juce-extensions, pluginval, iPlug2/HISE licensing, Moonbase/Keyzy, Windows cloud signing pricing, and current JUCE prices (confirm on juce.com).*
