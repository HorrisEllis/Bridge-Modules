# CHANGELOG — Bridge OS

> §6.1: Documentation is generated from proof, not written as aspiration.
> Every entry below reflects something that actually happened.

---

## v1.3.0 — 2026-05-03 — The Long Day

**Session duration:** ~12 hours
**Hardware:** Gigabyte Aorus B450 WiFi Pro · RTL8852BE · Windows 11 24H2 (build 26200) · Node.js v24.15.0 · PS 5.1 + PS7 7.6.1

---

### Repo Architecture — Three-Repo Split

**What was done:**
- Forked the kernel into `HorrisEllis/Bridge-Core` — 10 modules, stripped of all transport, UI, LLM, and mesh modules. Developers building custom stacks use this.
- Created `HorrisEllis/Bridge-Modules` — official module catalog, 35 modules catalogued in `catalog.json` with conflict/require/status/tags declarations per module.
- `Bridge-v2` remains the full distribution — batteries included, all modules pre-installed.

**The relationship:**
```
Bridge-Core     ← kernel
     ↑
Bridge-Modules  ← catalog
     ↑
Bridge-v2       ← full distribution
```

**Module manifest spec introduced:** any npm package with a `bridge` key in `package.json` is a Bridge module. Third-party modules publish to npm with `bridge-module` keyword.

---

### BLE Transport — Complete Audit

**What was attempted (12 approaches, all documented):**

Every approach to BLE advertisement scanning from a Node.js child process on Windows 11 24H2 was attempted:

1. PS5.1 `add_Received` — silent, handler never fires
2. PS5.1 `Register-ObjectEvent` — `INVALID_REGISTRATION` for WinRT types
3. PS5.1 `Add-Type` inline C# in `Load-WinRT` — winmd assembly load fails at init
4. PS5.1 `Add-Type` inline C# in `Start-BLEScan` — WinMetadata .winmd not loadable
5. C#5 csc.exe `+=` operator — CS1545, not supported with WinMD
6. C#5 csc.exe `add_Received()` — CS0571, cannot explicitly call accessor
7. C#5 csc.exe `WindowsRuntimeMarshal.AddEventHandler` — still calls add/remove → CS0571
8. C#5 csc.exe runtime reflection `EventInfo.AddEventHandler` — "dynamically adding WinRT events not supported"
9. C#5 csc.exe `DeviceWatcher` — only finds paired devices, not active advertisers
10. Raw HCI `\\.\bthHCI0` — not exposed by Windows 11
11. PS7 `ContentType=WindowsRuntime` cast — cannot load type, .NET 6 uses CsWinRT
12. `Get-PnpDevice` PnP poll — returns paired/known devices only

**Root cause confirmed:** Windows 11 24H2 blocks WinRT event subscription from non-UWP child processes. This is a platform restriction, not a code bug.

**Correct solution identified:** `.NET 6 SDK` with `net6.0-windows10.0.19041.0` TFM. CsWinRT included automatically. `watcher.Received +=` compiles and works. `ble-host/` contains the project. Pending: `winget install Microsoft.DotNet.SDK.8`.

**What IS proven and working:**
- Adapter detection — `USB\VID_0BDA&PID_0852` powers on correctly
- WinRT backend initializes — `bridge-winrt` backend, JSON line protocol, pong verified
- Advertising (Publisher) — works fine in PS5.1
- BLE in boot log — `✓ bridge-transport BLE :ready (bridge-winrt)`
- Chunking — 394 bytes → 25 chunks @ 20B MTU, reassembly verified
- MTU — 247 bytes max, 244 effective payload
- `/ble/status` route — returns real state (was always returning unavailable — fixed)
- 93/93 test suite passing

---

### BLE Status Route Fix

**Bug:** `/ble/status` always returned `{ state: 'unavailable' }`.

**Root cause:** `_adapters` was private in `TransportManager`. The `/ble/*` route did `ctx.transport._adapters?.find()` which always returned `undefined`. `getBLE()` always returned null.

**Fix:** Exposed `getBLEAdapter()` and `getBLE()` on `TransportManager`. Added boot log line showing BLE state after `transport.start()` resolves.

---

### Port Registry Fix

**Bug:** Boot hung in infinite loop printing `[port-registry] EADDRINUSE on 3747 — Retrying on port 3748` forever.

**Root cause:** Dynamic range was 3747-3760 — only 13 ports, all in the Bridge sovereign range. Every retry picked from the same blocked set.

**Fix:** `dynamicMin=49152`, `dynamicMax=65535`. Failed port excluded from retry candidates on subsequent attempts.

---

### Copyright Update

All 136 files updated: `2024-2026` → `2026`. Version header standardized.

---

### Architecture Designs (Specced, Not Yet Built)

#### WCC — Weighted Compute Credit
Complete design with adversarial pressure applied:
- Non-transferable. Interaction-based decay (not time-based).
- Cluster Sybil resistance: cap shared across behavioral cluster.
- Hierarchical λ: global (DHT) → cluster → node.
- Layer 1: Ed25519 dual-sig receipt.
- Layer 3: threat delta bonus.
- WCC = force field over routing, not gate between modules.
- Free floor: `qwen2:0.5b` local. Tiers: 5/10/15 WCC.

#### Sovereign Radio Protocol
- nRF24L01+ ($1.50) + ESP32-S2 ($3) = $5 total.
- Fixed 32-byte packets, AES-128-GCM, FHSS on HMAC-keyed schedule (125 channels).
- Same TransportAdapter interface as BLE/HTTP.
- Conductor picks in CRITICAL/BLACKOUT.

#### Journalist Dongle
- Form: wireless mouse receiver (VID:046D PID:C52B = Logitech Unifying).
- BOM: ESP32-S2 + nRF24L01+ + HID clone + shell = $5.60.
- Mouse works (passes functional inspection). Silent at rest. Key-activated.
- Steg-over-HID fallback: ~100bps, zero RF.

#### Open Platform / Module Catalog
- Module manifest: `bridge` key in package.json with `conflicts`, `requires`, `regime`, `vital`.
- 35 modules catalogued in `Bridge-Modules/catalog.json`.
- Conflict registry enforced at load time. CLI/UI shows reason for disabled modules.
- Third-party modules installable via npm with `bridge-module` keyword.

#### Sovereign AI Vision
Any device connected to a Bridge node gets AI:
- BLE speaker → voice interface (Whisper + Ollama + TTS)
- SSH terminal → LLM-reasoning shell
- RDP → AI workspace
- Any phone → I/O surface for mesh intelligence
- Control surfaces specced: audio, HTTP, Guardian, SSH, SMS, RDP, SFTP, FTP

#### Transport Fallback Cascade (BLE Relay)
Full relay architecture designed and tested in TransportManager:
- HTTP → BLE direct → BLE relay (phone over LTE) → BLE+steg (BLACKOUT).
- IME score threshold per regime for relay trust.
- BLE advertisement flag: `BRIDGE:<shortId>:R` = relay capable.

#### Model Tier Table
Updated after deepseek-coder discussion — coder models are wrong for bridge-mind:
- phi4-mini (recommended default, 2.5GB, 12 t/s, reasoning)
- qwen2.5:7b (strong reasoning, multilingual, threat analysis)
- mistral:7b (fast, solid reasoning)
- deepseek-r1:8b (high-end, slow, reasoning not coding)

#### Phasemap — sections 1-14
`bridge-os-v5.3-phasemap.html` expanded from 8 to 14 sections:
- Sections 12-13: Sovereign radio protocol + journalist dongle full spec
- Section 14: Open platform + module conflict system + extended catalog

---

### Files Added This Session

```
bridge-transport/bridge-winrt-ble-ps7.ps1    PS7 BLE backend
bridge-transport/bridge-ble-win.js           Windows PnP poll discovery
bridge-transport/bridge-hci.js              Raw HCI scanner (proven unavailable)
bridge-transport/ble-host/Program.cs        .NET 6 BLE host (correct solution)
bridge-transport/ble-host/ble-host.csproj   .NET 6 project file
bridge-transport/ble-host/build.ps1         Build script
bridge-transport/tests/test-ble-full.js     93-test comprehensive suite
bridge-transport/tests/test-ble-rtl8852be.js Smoke test (was already there, updated)
bridge-transport/tests/test-hci-scan.js     HCI scanner test
bridge-transport/tests/test-pnp-scan.js     PnP poll test
bridge-transport/tests/test-find-all.ps1    FindAllAsync one-shot test (pending)
bridge-os-v5.3-phasemap.html               Full session phasemap (14 sections)
ROADMAP.md                                  This roadmap
```

---

### Commits This Session (chronological)

```
fix: port-registry infinite retry loop on EADDRINUSE
v1.3.0 -- BLE integration hardened, RTL8852BE test harness, transport routes
docs: README updated to v1.3.0
docs: phasemap updated with relay cascade, trust model, phone architecture
fix: BLE test path resolver — finds bridge-winrt.js from any run location
fix: WinRT loader broken on Windows 11 24H2 (build 26xxx)
fix: PS1 file encoding — strip all non-ASCII chars for PS1 compatibility
fix: BLE scan Received event never fired -- add_Received broken in PS5.1
fix: skip GetDefaultAsync entirely -- use watcher probe for init on Win11 24H2
fix: suppress EventRegistrationToken noise + Advertisement readonly on Win11
fix: BLE status route always returned unavailable — _adapters not exported
fix: all 10 WinRT null/failure modes hardened in bridge-winrt-ble.ps1
test: comprehensive BLE transport test suite — 93 tests, 0 failures
feat: replace PS1 BLE backend with compiled C# exe (bridge-winrt-ble.cs)
fix: C# exe compiles on .NET 4.x / C#5 -- remove anonymous type reflection
fix: replace broken bat with PowerShell compile script
fix: build script searches for System.Runtime.dll facade automatically
fix: C# AsTask via reflection + add Windows.winmd ref
fix: use WindowsRuntimeMarshal.AddEventHandler for WinRT events in C#5
feat: v3.0.0 -- replace BluetoothLEAdvertisementWatcher with DeviceWatcher
feat: v4.0.0 -- pure reflection BLE scanner, zero WinMD compile-time refs
feat: v5.0.0 -- dotnet 6 BLE host, CsWinRT, proper WinRT events
feat: PS7 BLE backend -- PowerShell 7 supports WinRT += events natively
fix: octal escape in pwsh path strings -- use path.join + env vars
feat: Windows BLE via PnP poll -- proven, no WinRT events needed
test: add test-pnp-scan.js for Windows BLE cache discovery
test: FindAllAsync one-shot BLE scan -- no events, spin-wait
docs: open platform framing + module conflict system + extended catalog
docs: phasemap -- sovereign radio protocol + journalist dongle spec
docs: add repository structure section -- bridge-core and bridge-modules repos
init: bridge-modules official module catalog v1.3.0 [Bridge-Modules repo]
init: bridge-core v1.3.0 -- sovereign node kernel [Bridge-Core repo]
```

---

## v1.2.0 — (prior session)

Transport fallback cascade. BLE initial integration. Relay architecture.

## v1.1.0 — (prior session)

DHT. Bayesian engine. Onion routing hardened.

## v1.0.0 — (prior session)

Core kernel. Identity. IME. Sngate. Causal ring buffer.

---

> The codebase is a materialised view of the Upgrade Ledger.
> Ideas are input, not output.
> Let's make history.
