# ROADMAP — Bridge OS

> **Living document.** Updated after every session. Grounded in what is proven.
> §6.1: Documentation is generated from proof, not written as aspiration.

**Current version:** v1.3.0
**Last updated:** 2026-05-03
**Author:** James Brooks (Erosmancer)

---

## State of the System Right Now

### Proven and Passing

| Component | Evidence |
|-----------|----------|
| Boot sequence — all 10 phases | Live on Windows 11 24H2 — boot log in README |
| BLE adapter detection | `USB\VID_0BDA&PID_0852` powers on, state `poweredOn` |
| BLE chunking/reassembly | 93/93 tests passing — `test-ble-full.js` |
| BLE MTU probe | 247 bytes max, 244 effective payload |
| BLE /ble/status route | Returns `{ state: ready, backend: bridge-winrt }` |
| Transport fallback cascade | TransportManager invariants T-01 through T-05 proven |
| Port registry | Dynamic range 49152-65535, no infinite loop |
| HTTP API routes | All transport routes return correct responses |

### Known Limitations (Documented, Not Hidden)

| Limitation | Root Cause | Fix Path |
|------------|-----------|---------|
| BLE advertisement scanning | Windows 11 24H2 blocks WinRT events from non-UWP processes | `dotnet SDK` + `ble-host/build.ps1` |
| RTL8852BE peripheral role | Chip firmware limitation — central only | Update driver via Gigabyte App Center |
| ws package not installed | Not in package.json | `npm install ws` |
| Announced host empty | No `NEXUS_ANNOUNCED_HOST` env var | Set env var or configure |

---

## Build Order (§3.1 — bottom up only)

Nothing starts until the thing below it has passing tests. This is the law.

```
CURRENT ──────────────────────────────────────────────── v1.3.0

  ✓  bridge-identity     Ed25519, UUID, keystore
  ✓  bridge-core         SISO bus, registries
  ✓  bridge-IME          behavioral memory, trust scoring
  ✓  bridge-sngate       three-state gate
  ✓  bridge-causal       1M ring buffer, CQL
  ✓  bridge-transport    HTTP + BLE (WinRT) + relay cascade
  ✓  bridge-onion        multi-hop ECDH
  ✓  bridge-shaper       traffic normalization
  ✓  bridge-steg         steganographic channels
  ✓  bridge-mesh         ECDH peers, DHT
  ✓  bridge-ollama       local LLM driver
  ✓  bridge-bayesian     belief engine

NEXT ─────────────────────────────────────────────────── v1.4.0

  🔨 bridge-threat       GATE: all above passing

       Threat delta scoring system.
       Reads: IME profiles, sngate decisions, causal sigma,
              heartbeat BPM, mesh peer count, DHT health.
       Produces: score 0-100, regime CALM/ELEVATED/HIGH/CRITICAL/BLACKOUT.
       Emits: threat:score, threat:regime on bus.
       Exit criteria: score computes correctly from live bus events.
                      Regime transitions emit correct bus events.
                      100% test coverage on scoring logic.
                      No stubs. No mocks in production.

  →  bridge-conductor    GATE: bridge-threat passing      v1.5.0

       Threat-aware orchestration.
       Reads threat regime, activates/deactivates modules automatically.
       In CALM: HTTP preferred.
       In ELEVATED: enables BLE fallback.
       In HIGH: activates shaper + steg.
       In CRITICAL: forces onion routing, BLE relay.
       In BLACKOUT: radio only, kills all IP transport.
       Operator configures nothing. System adapts.

  →  bridge-relay        GATE: bridge-conductor passing   v1.6.0

       BLE → phone → LTE relay chain.
       Node connects to phone over BLE.
       Phone carries traffic over mobile data.
       Traffic appears to originate from phone's carrier.
       Local machine: zero visible network activity.
       IME score threshold per regime governs relay trust.

  →  bridge-wcc          GATE: bridge-relay passing       v1.7.0

       Weighted Compute Credit economy.
       Non-transferable. Interaction-based decay.
       Sybil resistance via behavioral cluster cap.
       Ed25519 dual-sig receipts on Layer 1.
       Free floor: qwen2:0.5b local.
       Tiers: 5/10/15 WCC per request.

  →  bridge-mind         GATE: bridge-wcc passing         v1.8.0

       Distributed LLM across mesh.
       phi4-mini default (2.5GB, ~12 t/s).
       qwen2.5:7b for threat analysis.
       WCC governs who pays for what.
       A $5 dongle queries a 70B model via WCC if warranted.
       bridge-mind also selects which modules to load
       based on detected environment — self-configuring.

  →  bridge-radio        GATE: bridge-conductor passing   v1.9.0

       Sovereign radio transport.
       Hardware: ESP32-S2 + nRF24L01+ = $5.
       Protocol: 32-byte fixed packets, AES-128-GCM, FHSS 125 channels.
       Hop schedule: HMAC-SHA256(shared_key, unix_minute)[i % 125].
       Same TransportAdapter interface as BLE/HTTP.
       Buildable at home in 20 minutes.
       Conductor picks in CRITICAL/BLACKOUT.

  →  bridge-ghost        GATE: bridge-radio passing       v2.0.0

       Journalist dongle.
       Hardware: ESP32-S2 + nRF24L01+ + HID clone + shell = $5.60.
       Enumerates as Logitech Unifying Receiver (VID:046D PID:C52B).
       Mouse actually works. Passes functional inspection.
       Cryptographic HID activation sequence.
       Silent at rest. No RF without activation.
       Steg-over-HID fallback: ~100bps, zero RF.
       Pre-keyed before distribution — plug in, done.
```

---

## Parallel Tracks

These can happen alongside the main build order:

### BLE Scanning (Windows)
```
1. Test FindAllAsync: powershell -File bridge-transport\tests\test-find-all.ps1
2. If returns devices: wire into bridge-winrt-ble.ps1 as scan mechanism
3. If blocked: install dotnet SDK: winget install Microsoft.DotNet.SDK.8
4. Build: pwsh -File bridge-transport\ble-host\build.ps1
5. Test: node bridge-transport/tests/test-ble-rtl8852be.js --scan-ms=20000 --verbose
6. Verify devices appear with iPhone nearby + Settings → Bluetooth open
```

### Open Platform Modules
Any of these can be built independently once bridge-threat exists:

```
bridge-pipeline   — automation and pipeline (Tasker-like trigger → action)
bridge-audio      — BLE speaker AI interface (Whisper + Ollama + TTS)
bridge-sms        — SMS/call transport adapter
bridge-rdp        — RDP control surface
bridge-sftp       — SFTP file transport
bridge-acoustic   — data over ultrasound (zero RF covert channel)
bridge-optical    — data over IR/visible light
bridge-powerline  — data over building electrical wiring
bridge-lora       — LoRa 10km+ radio transport
bridge-satellite  — Iridium/Globalstar SMS fallback
bridge-beacon     — minimal 8-byte position+condition packet (POW/field ops)
bridge-deadman    — proof-of-life protocol
bridge-sigint     — passive RF intelligence
```

---

## Exit Criteria by Phase

Every phase has explicit exit criteria. "It seems to work" is not exit criteria (§3.1).

### bridge-threat (v1.4.0)
- [ ] Threat score computes correctly from simulated bus events (100% unit test coverage)
- [ ] All 5 regime transitions emit correct `threat:regime` bus events
- [ ] Score is 0 when all inputs are nominal
- [ ] Score reaches 81+ when: mesh peer count drops to 0, sngate deny rate > 80%, heartbeat flatlines
- [ ] Score does not oscillate — hysteresis applied at transitions
- [ ] No external dependencies
- [ ] Integrates with boot sequence — loads at phase 9a level

### bridge-conductor (v1.5.0)
- [ ] In CALM: shaper/steg/onion are OFF
- [ ] In HIGH: shaper and steg activate automatically (no operator action)
- [ ] In CRITICAL: onion routing activates, BLE relay engaged
- [ ] In BLACKOUT: all HTTP transport disabled, radio only
- [ ] Transitions are reversible — BLACKOUT → CALM when threat score falls
- [ ] Transition events logged in causal ring

### bridge-relay (v1.6.0)
- [ ] Node connects to phone via BLE
- [ ] Traffic routes through phone's mobile data
- [ ] Remote Bridge node receives message correctly
- [ ] End-to-end latency documented
- [ ] Relay trust requires IME score above threshold per regime

---

## Module Conflict Registry

All conflicts are declared in `package.json` under the `bridge` key.
The module registry enforces at load time. These are the known conflicts:

| Module | Conflicts With | Reason |
|--------|---------------|--------|
| bridge-beacon | bridge-shaper | Beacon needs silent channel; shaper pads traffic |
| bridge-beacon | bridge-radio-active | Can't transmit + maintain radio silence |
| bridge-sigint | bridge-radio-active | Passive listening conflicts with active transmission |
| bridge-jam | all radio modules | Jamming conflicts with all other radio use |
| bridge-acoustic | bridge-radio | Different 2.4GHz interference |
| bridge-lora | bridge-radio | Same frequency band conflict |

---

## Hardware Reference

### Validated Hardware (§1.1 — proven)
- **Realtek RTL8852BE** — `USB\VID_0BDA&PID_0852` — Gigabyte Aorus B450 WiFi Pro
  - Central role: ✓ scan, discover, connect
  - Peripheral role: ✗ not supported by firmware
  - Driver: Windows 11 built-in (no install required)

### Specced Hardware (not yet built)
- **ESP32-S2** — $3 — native USB, dual-core 240MHz, SPI
- **nRF24L01+** — $1.50 — 2.4GHz ISM, 125 channels, 2Mbps
- **nRF24L01+ PA+LNA** — $3 — 1km+ range
- **18650 cell** — standalone node, 72hr runtime

---

## Version History

| Version | Date | What Shipped |
|---------|------|-------------|
| v1.3.0 | 2026-05-03 | BLE transport, RTL8852BE test harness, WCC design, three-repo split, 35-module catalog, phasemap sections 1-14, sovereign radio spec, journalist dongle spec, port fix, BLE status route fix |
| v1.2.0 | 2026-04-xx | Relay cascade, transport fallback, BLE integration |
| v1.1.0 | 2026-04-xx | DHT, bayesian engine, onion routing |
| v1.0.0 | 2026-04-xx | Core kernel, identity, IME, sngate, causal |

---

## Next Session Checklist

Start every session with this. §8.1 — begin with context.

```
□ node index.js — all phases green?
□ git status — clean?
□ powershell -File bridge-transport\tests\test-find-all.ps1 — any devices?
□ If dotnet available: pwsh -File bridge-transport\ble-host\build.ps1
□ node bridge-transport/tests/test-ble-rtl8852be.js --scan-ms=20000 --verbose
□ Are we on bridge-threat yet? If yes: start with the scoring model.
□ Nothing starts until the thing below it passes. Check the order.
```
