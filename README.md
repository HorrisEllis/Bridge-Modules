# bridge-modules — Official Module Catalog

> The official module catalog for [Bridge OS](https://github.com/HorrisEllis/Bridge-v2)

**Author:** James Brooks (Erosmancer) · [rheon.world](https://rheon.world)  
**Core runtime:** [bridge-core](https://github.com/HorrisEllis/bridge-core)  
**Full distribution:** [Bridge-v2](https://github.com/HorrisEllis/Bridge-v2)

---

## What This Is

Every official Bridge OS module, individually installable. Each module declares its dependencies, conflicts, and required threat regime. The module registry in bridge-core enforces all constraints at load time.

## Installing Modules

```bash
# From within a bridge-core or Bridge-v2 installation
nexus module install bridge-transport
nexus module install bridge-onion bridge-shaper bridge-steg
nexus module install bridge-radio        # requires hardware
nexus module install bridge-ghost        # journalist dongle firmware
```

## Module Index

### Transport Layer
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-transport | HTTP + BLE (WinRT native) + cellular | — |
| bridge-radio | nRF24L01+ sovereign radio protocol | bridge-jam |
| bridge-ghost | Journalist dongle firmware (ESP32+nRF24) | — |

### Security Layer
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-onion | Multi-hop ECDH onion routing | — |
| bridge-shaper | Traffic padding + jitter + mimicry | bridge-beacon |
| bridge-steg | Steganographic channels (JSON + HTTP) | — |
| bridge-proxy | HTTP/HTTPS reverse proxy + SNI | — |

### Network Layer
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-mesh | ECDH peer channels + trust-mesh | — |
| bridge-dht | Kademlia DHT + Ed25519 records | — |
| bridge-routing | TCP mesh router :3749 | — |
| bridge-gateway | HTTP API gateway :3748 | — |
| bridge-ddns | DDNS client + custom DNS server | — |

### Intelligence Layer
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-threat | Threat delta scoring · regimes | — |
| bridge-conductor | Threat-aware orchestration | — |
| bridge-wcc | Weighted Compute Credit ledger | — |
| bridge-mind | Distributed LLM (phi4-mini / qwen2.5) | — |
| bridge-ollama | Local Ollama LLM driver | — |
| bridge-bayesian | Beta belief engine | — |

### Field Operations (CRITICAL/BLACKOUT regimes)
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-beacon | Minimal mode: 8-byte position+condition, 24hr burst | bridge-shaper, bridge-radio-active |
| bridge-deadman | Proof-of-life protocol · mesh alert on silence | — |
| bridge-cache | Encrypted flash dead-drop store | — |
| bridge-burst | Transmission window governor | — |
| bridge-sigint | Passive RF signals intelligence · feeds threat score | bridge-radio-active |

### Covert Channels (speculative)
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-acoustic | Data over ultrasound | bridge-radio |
| bridge-optical | Data over IR/visible light | — |
| bridge-powerline | Data over building electrical wiring | — |
| bridge-lora | LoRa radio transport (10km+) | bridge-radio |
| bridge-satellite | Iridium/Globalstar SMS transport | — |

### Platform
| Module | Description | Conflicts |
|--------|-------------|-----------|
| bridge-mobile | Pi/Android power + watchdog | — |
| bridge-ipfs | CIDv0 content-addressed storage | — |
| bridge-guardian | Browser extension endpoint | — |
| bridge-plugin | WebSocket gateway | — |
| bridge-magnet | nexus:// URI resolution | — |
| bridge-ssh | SSH session management | — |
| bridge-cfr | Physics mesh visualizer | — |

---

## Writing A Module

```json
{
  "name": "bridge-mymodule",
  "version": "1.0.0",
  "description": "What this module does",
  "bridge": {
    "conflicts": ["bridge-other-module"],
    "requires":  ["bridge-identity", "bridge-core"],
    "vital":     false,
    "regime":    "CALM",
    "tags":      ["transport", "covert"]
  }
}
```

Publish to npm with the `bridge-module` keyword. The catalog picks it up automatically.

---

## License

Copyright © 2026 James Brooks (Erosmancer). All rights reserved.
