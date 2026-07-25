# OSI Layer 1 — Physical Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

Layer 1 doesn't really have "protocols" in the message-exchange sense — it defines how raw **bits** become electrical voltages, light pulses, or radio waves on a medium. What people call "Layer 1 protocols" are really **physical/electrical standards**. There's no header, no addressing, no logic — just signal encoding.

## Why this layer exists as its own thing

Before standardization, every vendor built proprietary cabling, connectors, and signaling schemes — a NIC from one manufacturer literally couldn't talk to another's, even ignoring software. Layer 1 standards exist purely to guarantee **electrical/optical interoperability**, independent of what's actually being communicated. This separation is what let Ethernet's MAC-layer logic (Layer 2) stay stable for 40+ years while the physical medium evolved from thick coax to fiber to twisted pair to Wi-Fi radio underneath it.

## Popular Layer 1 standards

### Ethernet PHY standards (IEEE 802.3 family)

- **Problem solved:** Early LANs (ARCNET, Token Ring, coax-based 10BASE5 "thicknet") were expensive, fragile (a single cable break took down the whole shared bus), and hard to install through buildings.
- **How it works:** Defines voltage levels, encoding (Manchester encoding for 10BASE-T, then more efficient encodings like 4B5B/8B10B/64B66B for faster speeds), cable types (Cat3 → Cat5e → Cat6a → Cat8), and max segment lengths.
- **Evolution:** 10BASE-T (10 Mbps, 1990) → 100BASE-TX "Fast Ethernet" (1995) → 1000BASE-T "Gigabit" (1999) → 10GBASE-T → up to 400GbE today for datacenter backbones.
- **Advantages:** Twisted-pair star topology (via a hub/switch) meant one bad cable only takes down one device, not the whole segment; massive backward compatibility (auto-negotiation lets a Gigabit NIC talk to a 100Mbps switch).
- **Disadvantages:** Copper has distance limits (100m for most twisted-pair standards) — fiber needed for longer runs.
- **Owner:** IEEE 802.3 Working Group.
- **Implementations:** NIC chipsets (Intel I350, Realtek RTL8111), switch ASICs (Broadcom Trident/Tomahawk), structured cabling (Cat5e/6/6a/8).

### Fiber optic standards (SMF/MMF, SFP/SFP+/QSFP)

- **Problem solved:** Copper's distance and EMI (electromagnetic interference) limits made it useless for long-haul or noisy industrial links.
- **How it works:** Encodes bits as light pulses (LED for multi-mode/MMF short runs, laser for single-mode/SMF long-haul) through glass fiber. Physical transceivers (SFP, SFP+, QSFP28) are hot-swappable modules that plug into switch/router ports, decoupling the electrical switch ASIC from the optical medium.
- **Advantages:** Immune to EMI, far longer distances (SMF: tens of km), much higher bandwidth ceiling.
- **Disadvantages:** More expensive, more fragile physically (fiber bend radius), requires precision alignment/cleaving during installation.
- **Owner:** ITU-T (fiber specs), IEEE 802.3 (Ethernet-over-fiber variants like 10GBASE-LR).
- **Implementations:** Datacenter spine-leaf fabrics, submarine cables, ISP backbone links.

### Wi-Fi PHY (IEEE 802.11 a/b/g/n/ac/ax/be)

- **Problem solved:** Needing connectivity without running cable at all — mobility.
- **How it works:** Radio transmission in unlicensed ISM bands (2.4GHz, 5GHz, 6GHz for Wi-Fi 6E/7), using modulation schemes that got progressively more spectrally efficient: DSSS (802.11b) → OFDM (802.11a/g) → MIMO/OFDM (802.11n/ac) → OFDMA (802.11ax "Wi-Fi 6", which multiplexes multiple clients per transmission the way cellular networks do).
- **Advantages:** No cabling, roaming mobility.
- **Disadvantages:** Shared, contended medium (see CSMA/CA in [[OSI Layer 2 - Data Link Layer Protocols and Technologies]]) — throughput degrades sharply with more clients or interference; less reliable/secure than wired without care.
- **Owner:** IEEE 802.11 Working Group, certified/branded by the Wi-Fi Alliance.
- **Implementations:** Broadcom/Qualcomm Wi-Fi chipsets, access points, virtually every phone/laptop radio.

### RS-232 / Serial

- **Problem solved:** One of the earliest standardized ways to connect a terminal to a computer/modem — predates Ethernet entirely (1960s).
- **How it works:** Simple voltage-level asynchronous signaling, one bit at a time, no inherent addressing (point-to-point only).
- **Relevance today:** Console/management ports on routers, switches, and server BMCs still use RS-232-derived serial consoles (via USB-to-serial adapters now) — this is exactly the kind of port you'd use for out-of-band access to a router if the network itself is down.
- **Owner:** Originally EIA (Electronic Industries Alliance), now TIA.

### Cellular PHY (4G LTE, 5G NR)

- **Problem solved:** Wide-area mobile connectivity beyond a single building's Wi-Fi range.
- **How it works:** Licensed spectrum, OFDMA (downlink) with cell-tower-managed scheduling; 5G NR adds massive MIMO and millimeter-wave bands for much higher throughput at shorter range.
- **Owner:** 3GPP (a consortium of telecom standards bodies), spectrum licensing by national regulators (FCC in the US).

### DOCSIS (cable modem PHY)

- **Problem solved:** Reusing existing cable-TV coax infrastructure for internet, instead of running new wiring to every home.
- **How it works:** Frequency-division multiplexing — internet data shares the coax cable with TV channels on different frequency bands.
- **Owner:** CableLabs.

---
Related concepts: [[Ethernet]] · [[Fiber Optics]] · [[Wi-Fi]] · [[RS-232]] · [[DOCSIS]]
