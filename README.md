# SENTINEL-SECURITY-FOR-V2V-COMMS
THIS WAS A PERSONAL PROJECT AIMED AT CREATING A SECURITY LAYER FOR V2V COMMUNICATION AGAINST POTENTIAL ATTACKERS  
# VehNet: Transport-Agnostic Security Pipeline for Vehicular Networks

**A cryptographically-enforced security system for vehicular ad-hoc networks with ML-based behavioral anomaly detection.**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://python.org)
[![Hardware Validated](https://img.shields.io/badge/hardware-ESP32%20tested-green.svg)](#hardware-validation)

---

## Problem Statement

Vehicular ad-hoc networks (VANETs) face fundamental security challenges:

1. **No pre-shared infrastructure** — vehicles must authenticate each other without centralized PKI
2. **Broadcast medium vulnerability** — attackers can replay, spoof, or flood the network
3. **Real-time constraints** — security decisions must complete in <10ms
4. **Resource-constrained nodes** — ESP32-class microcontrollers with limited memory

Existing solutions either sacrifice security for performance or assume infrastructure availability.

**VehNet provides cryptographically-enforced security with zero infrastructure assumptions.**

---

## Threat Model

### Attacker Capabilities
- **Passive eavesdropping** — attacker can observe all network traffic
- **Active replay** — attacker can capture and retransmit valid packets
- **Identity spoofing** — attacker can forge sender MAC addresses
- **Sybil attacks** — attacker can create multiple fake identities
- **Message injection** — attacker can craft malicious packets
- **Jamming** — attacker can degrade RF channel quality

### Security Guarantees (AUTHORITATIVE)

| Attack Vector | Defense Mechanism | Result |
|--------------|-------------------|--------|
| Replay (timestamp) | ±1s window enforcement | **REJECT** |
| Replay (nonce) | Per-sender nonce tracking | **REJECT** |
| Replay (sequence) | Monotonic sequence enforcement | **REJECT** |
| Replay (hash) | Packet hash deduplication | **REJECT** |
| Invalid signature | Ed25519 verification | **REJECT** |
| Unknown identity | TOFU enforcement | **REJECT** |
| Protocol violation | Magic/version/CRC checks | **REJECT** |

### ML-Based Detection (OBSERVATIONAL)

| Threat Pattern | Detection Method | Result |
|----------------|------------------|--------|
| Sybil correlation | Cross-identity RF profiling | **FLAG** (not REJECT) |
| Behavioral anomaly | Statistical outlier detection | **FLAG** (not REJECT) |
| Jamming indicators | RSSI/SNR correlation | **ALERT** (logged) |

**Critical Design Principle:**
- **Security decisions (REJECT) are deterministic and cryptographically enforced**
- **ML analysis (FLAG) is probabilistic and advisory only**
- **ML NEVER overrides security — Phase 1 is absolute authority**

### Out of Scope
- **Physical layer attacks** (RF jamming countermeasures)
- **Side-channel attacks** (timing, power analysis)
- **Compromised nodes** (assumed honest-but-curious)
- **GPS spoofing** (location integrity not enforced)

---

## Architecture

### Pipeline Flow

┌─────────────────────────────────────────────────────────────┐
│                     PACKET RECEIVED                         │
└────────────────────────┬────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1: HARD SECURITY GATE (ABSOLUTE AUTHORITY)           │
│                                                              │
│  1. Protocol Validation  → Deserialize, check magic/version │
│  2. Identity Resolution  → TOFU lookup (HELLO-first)        │
│  3. Crypto Verification  → Ed25519 signature verify         │
│  4. Replay Protection    → timestamp/nonce/sequence/hash    │
│                                                              │
│  ANY FAILURE → IMMEDIATE REJECT (no ML inference)           │
└────────────────────────┬────────────────────────────────────┘
│
✓ PASSED
│
▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2: DETERMINISTIC POLICY (RESERVED)                   │
│  Future: rate limiting, priority enforcement                │
└────────────────────────┬────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3: ML INFERENCE (ADVISORY ONLY)                      │
│                                                              │
│  1. Feature Extraction → Behavioral profiling               │
│  2. ML Inference       → RandomForest classification        │
│  3. Sybil Detection    → Cross-identity correlation         │
│  4. Anomaly Scoring    → Statistical outlier detection      │
│                                                              │
│  ML CAN: Upgrade ACCEPT → FLAG (suspicious)                 │
│  ML CANNOT: Downgrade ACCEPT → REJECT                       │
│  ML CANNOT: Override Phase 1 REJECT                         │
└────────────────────────┬────────────────────────────────────┘
│
▼
DECISION OUTPUT
(ACCEPT | REJECT | FLAG)



### Decision Authority Hierarchy

**Phase 1 > Phase 2 > Phase 3**

- **Phase 1 (Security):** Cryptographically absolute — REJECT cannot be overridden
- **Phase 2 (Policy):** Deterministic rules — future expansion for QoS
- **Phase 3 (ML):** Probabilistic advisory — enriches ACCEPT with suspicion flags

**Why This Ordering?**
1. **Security first** — compromised packets must never reach application logic
2. **Performance** — ML inference skipped entirely for rejected packets (~50% savings)
3. **Fail-safe** — ML failures return ACCEPT (fail-open for ML, not security)

---

## Transport-Agnostic Design

VehNet is **intentionally transport-agnostic**:

- **Current hardware validation:** ESP-NOW (ESP32)
- **Protocol support:** Any broadcast medium (LoRa, Zigbee, BLE, etc.)
- **Not included in this release:** LoRa firmware (design permits, not implemented)

**Why ESP-NOW?**
- Low latency (<10ms)
- Hardware availability (ESP32 dev boards)
- Broadcast native support

**Why NOT LoRa yet?**
- Requires different RF stack (SX127x drivers)
- Longer air time (100-500ms per packet)
- Hardware procurement delay

**The packet format and security pipeline are radio-agnostic.**

---

## Hardware Validation

**Platform:** ESP32-WROOM-32 (Espressif)  
**Firmware:** Custom C++ sender (Ed25519 signing on-device)  
**Python Runtime:** Raspberry Pi 4 receiver  
**Test Duration:** 1000+ packets at 50 pkt/sec  

### Validation Results

| Metric | Result |
|--------|--------|
| Valid packet acceptance | 100% |
| Replay attack rejection | 100% |
| Invalid signature rejection | 100% |
| Unknown identity rejection | 100% |
| Avg validation latency | 0.8ms |
| Max validation latency | 4.2ms |
| ML inference overhead | 0.3ms |

**Critical finding:** Replay violations were **detected and logged** correctly, with decisions properly enforced after fixing control-flow bug (see `CRITICAL_BUG_FIX_REPORT.md`).

---

## Installation

### Requirements
- Python 3.10+
- Dependencies: `cryptography`, `numpy`, `scikit-learn`

```bash
git clone https://github.com/yourusername/vehnet.git
cd vehnet
pip install -r requirements.txt
Quick Start

from vehnet.core.identity import IdentityRegistry
from vehnet.core.replay import ReplayProtector
from vehnet.core.packet_validator import PacketValidator

# Initialize security components
self_mac = b"\xff\xff\xff\xff\xff\xff"
identity_registry = IdentityRegistry(self_mac)
replay_protector = ReplayProtector(window_seconds=60)
validator = PacketValidator(identity_registry, replay_protector)

# Validate a packet
result = validator.validate_packet(packet_bytes)

if result.decision == DecisionType.REJECT:
    print(f"REJECTED: {result.rejection_reason}")
elif result.decision == DecisionType.FLAG:
    print(f"ACCEPTED but flagged: {result.flag_reason.value}")
else:
    print("ACCEPTED")
Testing
Run the security verification suite:


cd tests/
python3 -m pytest test_crypto.py          # Crypto layer tests
python3 -m pytest test_replay.py          # Replay protection tests
python3 -m pytest test_packet_validator.py # Integration tests
Expected: All tests pass. Any failures indicate security regression.

Project Status
✅ Crypto layer: Ed25519 signing/verification (production-ready)
✅ Replay protection: 4-layer defense (production-ready)
✅ Identity management: TOFU with consistency checks (production-ready)
✅ ML integration: Sybil detection, anomaly flagging (observational)
✅ Hardware validation: ESP32 tested (1000+ packets)
⚠️ GUI: Terminal dashboard exists but excluded from release (unpolished)
❌ LoRa support: Protocol permits, not implemented
