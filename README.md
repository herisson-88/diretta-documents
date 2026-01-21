
## The purpose of this document is to present Diretta, its implementations, and in particular the latest one: the Diretta UPnP renderer

### Diretta is proprietary licensed software by Yu Harada. The Diretta UPnP Renderer is an open‑source Host implementation built using the Diretta Host SDK, which is also owned by Yu Harada.

### https://www.diretta.link
### https://github.com/cometdom/DirettaRendererUPnP

# DIRETTA - Hi-Fi Audio Protocol

## Philosophy and Fundamental Principle

Diretta is an audio transmission protocol over Ethernet developed in Japan by Yu Harada. Its goal is to improve sound quality by maintaining a constant load on the receiver, with the transmitting side as **Host** and the receiving side as **Target**.

### The Problem Solved

Variations in power consumption of digital blocks generate noise that affects audio quality. These low-frequency disturbances pass through traditional low-pass filters and are detectable in the audio band.

### The Diretta Solution

Packets are transmitted as frequently as possible at short, constant intervals to smooth processing. Transmission is controlled by predicting the Target's buffers. This way, power consumption fluctuations of the player (Target) are averaged to the maximum extent.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DIRETTA PRINCIPLE - FLOW SMOOTHING                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   CONVENTIONAL TRANSMISSION (UPnP, HTTP...)                                 │
│   ══════════════════════════════════════                                    │
│                                                                             │
│   Target CPU/Network Activity:                                              │
│                                                                             │
│   █████          ████████████               ███████                         │
│   █████          ████████████               ███████                         │
│   █████          ████████████               ███████                         │
│   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│   Irregular bursts → Current variations → Power supply noise                │
│                                                                             │
│   ─────────────────────────────────────────────────────────────────────     │
│                                                                             │
│   DIRETTA TRANSMISSION                                                      │
│   ════════════════════                                                      │
│                                                                             │
│   Target CPU/Network Activity:                                              │
│                                                                             │
│   ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██   │
│   ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██   │
│   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│   Regular packets → Constant consumption → Stable power supply              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

- The Host synchronizes with the Target (not the reverse)
- No buffer/flow control like asynchronous USB
- Isolatable Ethernet connection (fiber optic possible)
- No complex processing like UPnP file range requests
- No multi-device synchronization like AES67

---

## Basic Architecture: Host + Target

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DIRETTA 2-BOX ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    ┌──────────────────────────┐         ┌──────────────────────────┐        │
│    │         HOST             │         │         TARGET           │        │
│    │  ════════════════════    │         │  ════════════════════    │        │
│    │                          │         │                          │        │
│    │  ┌──────────────────┐    │         │  ┌──────────────────┐    │        │
│    │  │   Audio Player   │    │         │  │  Diretta Engine  │    │        │
│    │  │  (Roon, Audirvana│    │         │  │    (Minimal)     │    │        │
│    │  │   HQPlayer...)   │    │         │  │                  │    │        │
│    │  └────────┬─────────┘    │         │  └────────┬─────────┘    │        │
│    │           │              │         │           │              │        │
│    │           ▼              │         │           ▼              │        │
│    │  ┌──────────────────┐    │         │  ┌──────────────────┐    │        │
│    │  │  Diretta Driver  │    │  IPv6   │  │   USB Audio      │    │        │
│    │  │  (ASIO or ALSA)  │────┼─────────┼──│   Driver         │    │        │
│    │  │                  │    │ Ethernet│  │                  │    │        │
│    │  └──────────────────┘    │         │  └────────┬─────────┘    │        │
│    │                          │         │           │              │        │
│    │  Complex processing      │         │           │ USB          │        │
│    │  Storage, decoding       │         │           ▼              │        │
│    │  Standard network        │         │       ┌───────┐          │        │
│    │                          │         │       │  DAC  │          │        │
│    └──────────────────────────┘         │       └───────┘          │        │
│                                         │  Minimal processing      │        │
│                                         │  Close to DAC            │        │
│                                         └──────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation 1: Windows Host + Target

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               WINDOWS HOST + DIRETTA TARGET CONFIGURATION                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│         Windows PC                            Diretta Target                │
│    ┌─────────────────────┐             ┌─────────────────────┐              │
│    │                     │             │                     │              │
│    │  ┌───────────────┐  │             │  ┌───────────────┐  │              │
│    │  │ Roon          │  │             │  │ GentooPlayer  │  │              │
│    │  │ Audirvana     │  │             │  │ AudioLinux    │  │              │
│    │  │ foobar2000    │  │             │  │ Eunhasu       │  │              │
│    │  │ JRiver        │  │             │  │               │  │              │
│    │  │ HQPlayer      │  │             │  │  (Light Linux)│  │              │
│    │  └───────┬───────┘  │             │  └───────┬───────┘  │              │
│    │          │          │             │          │          │              │
│    │          ▼          │             │          ▼          │              │
│    │  ┌───────────────┐  │   Diretta   │  ┌───────────────┐  │              │
│    │  │ ASIO Driver   │  │   Protocol  │  │ Diretta Recv  │  │              │
│    │  │ Diretta       │═══════════════>│  │               │  │              │
│    │  │               │  │    IPv6     │  │               │  │              │
│    │  └───────────────┘  │   Ethernet  │  └───────┬───────┘  │              │
│    │                     │             │          │ USB      │              │
│    │  Win 10/11          │             │          ▼          │              │
│    │  Win Server 2019    │             │      ┌───────┐      │              │
│    └─────────────────────┘             │      │  DAC  │      │              │
│                                        │      └───────┘      │              │
│                                        └─────────────────────┘              │
│                                                                             │
│    Supported Target Hardware:                                               │
│    • Raspberry Pi 4/5                                                       │
│    • x86 Mini PC (NUC)                                                      │
│    • Dedicated hardware (see http://www.diretta.link)                       │
│    • Ian Canada boards                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation 2: Linux Host + Target

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 LINUX HOST + DIRETTA TARGET CONFIGURATION                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│         Linux PC (Host)                      Diretta Target                 │
│    ┌─────────────────────┐             ┌─────────────────────┐              │
│    │                     │             │                     │              │
│    │  ┌───────────────┐  │             │                     │              │
│    │  │ Roon Server   │  │             │  ┌───────────────┐  │              │
│    │  │ HQPlayer Emb. │  │             │  │ Diretta       │  │              │
│    │  │ MPD           │  │             │  │ Target        │  │              │
│    │  │ Squeezelite   │  │             │  │ Engine        │  │              │
│    │  │ Audirvana     │  │             │  └───────┬───────┘  │              │
│    │  └───────┬───────┘  │             │          │          │              │
│    │          │          │             │          │ USB      │              │
│    │          ▼          │             │          ▼          │              │
│    │  ┌───────────────┐  │   Diretta   │      ┌───────┐      │              │
│    │  │ ALSA Driver   │  │   Protocol  │      │  DAC  │      │              │
│    │  │ Diretta       │═══════════════>│      └───────┘      │              │
│    │  │               │  │    IPv6     │                     │              │
│    │  └───────────────┘  │   Ethernet  └─────────────────────┘              │
│    │                     │                                                  │
│    │  AudioLinux         │                                                  │
│    │  GentooPlayer       │                                                  │
│    │  Arch Linux         │                                                  │
│    └─────────────────────┘                                                  │
│                                                                             │
│    Linux Advantage: Fine system control, CPU isolation, RT kernel           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation 3: 3-Box Configuration (UPnP)

This configuration separates user control from the Diretta Host.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    3-BOX CONFIGURATION WITH UPnP                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   BOX 1: Control          BOX 2: Diretta Host     BOX 3: Target             │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐       │
│   │                 │     │                 │     │                 │       │
│   │  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │       │
│   │  │ Control   │  │     │  │ MPD       │  │     │  │ Diretta   │  │       │
│   │  │ Point     │  │     │  │    +      │  │     │  │ Target    │  │       │
│   │  │ (App)     │  │     │  │ upmpdcli  │  │     │  │           │  │       │
│   │  └─────┬─────┘  │     │  └─────┬─────┘  │     │  └─────┬─────┘  │       │
│   │        │        │     │        │        │     │        │        │       │
│   │        │ UPnP   │     │        │        │     │        │ USB    │       │
│   │        ▼        │     │        ▼        │     │        ▼        │       │
│   │  Tablet/        │     │  ┌───────────┐  │     │    ┌───────┐    │       │
│   │  Smartphone     │     │  │ Diretta   │  │     │    │  DAC  │    │       │
│   │  (BubbleUPnP,   │     │  │ ALSA Host │═══════>|    └───────┘    │       │
│   │   Kazoo)        │     │  │           │  │ IPv6│                 │       │
│   └─────────────────┘     │  └───────────┘  │     └─────────────────┘       │
│                           │                 │                               │
│   Standard LAN Network    │  Dedicated Linux│     Close to DAC              │
│                           └─────────────────┘                               │
│                                                                             │
│   Flow: Control Point ──UPnP──> MPD/upmpdcli ──Diretta──> Target ──> DAC    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    3-BOX CONFIGURATION WITH ROON                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   BOX 1: Roon Core        BOX 2: Roon Bridge      BOX 3: Target             │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐       │
│   │                 │     │                 │     │                 │       │
│   │  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │       │
│   │  │ Roon Core │  │     │  │ Roon      │  │     │  │ Diretta   │  │       │
│   │  │ Server    │  │     │  │ Bridge    │  │     │  │ Target    │  │       │
│   │  │           │  │     │  │           │  │     │  │           │  │       │
│   │  └─────┬─────┘  │     │  └─────┬─────┘  │     │  └─────┬─────┘  │       │
│   │        │        │     │        │        │     │        │        │       │
│   │        │ RAAT   │     │        │        │     │        │ USB    │       │
│   │        │        │     │        ▼        │     │        ▼        │       │
│   │        │        │     │  ┌───────────┐  │     │    ┌───────┐    │       │
│   │        └────────┼────>│  │ Diretta   │═══════>│    │  DAC  │    │       │
│   │                 │     │  │ ALSA Host │  │ IPv6│    └───────┘    │       │
│   │  NAS/Server     │     │  └───────────┘  │     │                 │       │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘       │
│                                                                             │
│   The Roon Bridge sees the Target as a local sound card (ALSA Diretta)      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Memory Play: System Layer Bypass

The DPDKMemoryPlay system (based on DPDK - Data Plane Development Kit) is a proprietary embedded Linux system that prevents the PC from being used for any purpose other than running Diretta. Sound improvements with Memory Play (which bypasses Linux's ALSA audio system) are maximized compared to the standard ALSA configuration.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              DIRETTA MEMORY PLAY - COMPLETE SYSTEM BYPASS                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│     CLASSIC LINUX ARCHITECTURE          MEMORY PLAY ARCHITECTURE            │
│                                                                             │
│     ┌─────────────────────┐               ┌─────────────────────┐           │
│     │    Application      │               │  Memory Play        │           │
│     │    (Player)         │               │  Controller (Win)   │           │
│     └──────────┬──────────┘               └──────────┬──────────┘           │
│                │                                     │ Network              │
│                ▼                                     ▼                      │
│     ┌─────────────────────┐               ┌─────────────────────┐           │
│     │    PulseAudio/      │               │  DPDK Memory Play   │           │
│     │    PipeWire         │               │  Host               │           │
│     └──────────┬──────────┘               │  ═════════════════  │           │
│                │                          │                     │           │
│                ▼                          │  • Files loaded     │           │
│     ┌─────────────────────┐               │    in RAM           │           │
│     │       ALSA          │               │  • Direct hardware  │           │
│     │    (User Space)     │               │    access (DPDK)    │           │
│     └──────────┬──────────┘               │  • No syscalls      │           │
│                │                          │  • No ALSA          │           │
│                ▼                          │  • No scheduler     │           │
│     ┌─────────────────────┐               └──────────┬──────────┘           │
│     │   Kernel Drivers    │                          │ Diretta              │
│     └──────────┬──────────┘                          ▼                      │
│                │                          ┌─────────────────────┐           │
│                ▼                          │    Diretta Target   │           │
│           [Hardware]                      └──────────┬──────────┘           │
│                                                      │ USB                  │
│                                                      ▼                      │
│     Multiple layers                             [   DAC   ]                 │
│     = Multiple interrupts                                                   │
│     = System jitter                            Minimal path                 │
│                                                = Minimal jitter             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE MEMORY PLAY CONFIGURATION                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    Windows PC                  Dedicated DPDK Host        Target            │
│   (Control)                    (Playback)                 (Output)          │
│   ┌───────────────┐           ┌───────────────┐         ┌───────────────┐   │
│   │               │           │               │         │               │   │
│   │ MemoryPlay    │   Network │  Linux DPDK   │ Diretta │  Diretta      │   │
│   │ Controller    │══════════>│  embedded    ══════════>│  Target       │   │
│   │ GUI           │   (Config)│               │  IPv6   │               │   │
│   │               │           │  Files        │         │               │   │
│   │ • File        │           │  in RAM       │         └───────┬───────┘   │
│   │   selection   │           │               │                 │ USB       │
│   │ • Playback    │           │  No standard  │                 ▼           │
│   │   control     │           │  OS           │             ┌───────┐       │
│   │               │           │               │             │  DAC  │       │
│   └───────────────┘           └───────────────┘             └───────┘       │
│                                                                             │
│    Limitations:                                                             │
│    • Supported formats: FLAC, WAV, DSF, DFF, AIFF only                      │
│    • No streaming (local files only)                                        │
│    • Minimalist interface                                                   │
│    • All files in a batch must have the same format                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diretta Renderer UPnP: Integrated Host SDK

The Diretta Renderer UPnP integrates the Host SDK directly into a UPnP renderer, combining the advantages of UPnP control with Diretta optimization. The spirit is similar to Memory Play: minimizing intermediate layers.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DIRETTA RENDERER UPnP                                  │
│              (Host SDK Integration in the Renderer)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    Control Point              Diretta Renderer UPnP         Target + DAC    │
│   ┌───────────────┐          ┌─────────────────────┐       ┌────────────┐   │
│   │               │          │                     │       │            │   │
│   │  mconnect     │          │  ┌───────────────┐  │       │  Diretta   │   │
│   │  JPlay        │   UPnP   │  │ UPnP Renderer │  │       │  Target    │   │
│   │  BubbleUPnP   │═════════>│  │ (OpenHome)    │  │       │            │   │
│   │               │          │  └───────┬───────┘  │       └──────┬─────┘   │
│   │               │          │          │          │              │         │
│   └───────────────┘          │          ▼          │              │ USB     │
│                              │  ┌───────────────┐  │              ▼         │
│                              │  │ Audio         │  │          ┌───────┐     │
│                              │  │ Decoder       │  │          │  DAC  │     │
│                              │  │ (FFmpeg)      │  │          └───────┘     │
│                              │  └───────┬───────┘  │                        │
│                              │          │          │                        │
│                              │          ▼          │                        │
│                              │  ┌───────────────┐  │  Diretta               │
│                              │  │ DIRETTA SDK   │═>│                        │
│                              │  │ HOST          │  │  IPv6                  │
│                              │  │ (Integrated)  │  │  Smoothed              │
│                              │  └───────────────┘  │                        │
│                              │                     │                        │
│                              │  All-in-one:        │                        │
│                              │  • UPnP reception   │                        │
│                              │  • Decoding         │                        │
│                              │  • Diretta Host     │                        │
│                              └─────────────────────┘                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Roon Integration via Squeeze Bridge UPnP

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              ROON + SQUEEZE BRIDGE UPnP + DIRETTA RENDERER                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐  │
│   │              │    │              │    │              │    │          │  │
│   │    ROON      │    │   SQUEEZE    │    │   DIRETTA    │    │ DIRETTA  │  │
│   │    CORE      │───>│   BRIDGE     │───>│   RENDERER   │───>│ TARGET   │  │
│   │              │    │   UPnP       │    │   UPnP       │    │          │  │
│   │              │    │              │    │              │    │          │  │
│   └──────────────┘    └──────────────┘    └──────────────┘    └────┬─────┘  │
│                                                                    │ USB    │
│        RAAT             Squeezebox            Diretta              ▼        │
│      Protocol            → UPnP              Protocol          ┌───────┐    │
│                                                                │  DAC  │    │
│                                                                └───────┘    │
│                                                                             │
│   • Roon sees the renderer as a Squeezebox endpoint                         │
│   • Squeeze Bridge translates the protocol to UPnP/OpenHome                 │
│   • The Diretta Renderer decodes and transmits via Host SDK                 │
│   • The Target receives an optimized smoothed stream                        │
│                                                                             │
│   Advantages:                                                               │
│   ─────────────                                                             │
│   • Complete Roon interface (DSP, library, metadata)                        │
│   • Optimized Diretta transmission to the Target                            │
│   • Unified architecture for UPnP and Roon                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│        COMPARISON: MPD+upmpdcli vs Diretta Renderer UPnP                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    CLASSIC SOLUTION MPD + upmpdcli + Diretta ALSA                           │
│    ═════════════════════════════════════════════════                        │
│                                                                             │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌────────┐   │
│    │upmpdcli │───>│   MPD   │───>│  ALSA   │───>│ Diretta │───>│ Target │   │
│    │(UPnP)   │    │(Player) │    │(Driver) │    │  ALSA   │    │        │   │
│    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └────────┘   │
│                                                                             │
│    5 components = 5 potential latency/jitter points                         │
│                                                                             │
│    ─────────────────────────────────────────────────────────────────────    │
│                                                                             │
│    DIRETTA RENDERER UPnP SOLUTION (Integrated SDK)                          │
│    ═══════════════════════════════════════════════                          │
│                                                                             │
│    ┌─────────────────────────────────────┐              ┌────────┐          │
│    │  Diretta Renderer UPnP              │   Diretta    │ Target │          │
│    │  (UPnP + Decode + Host SDK)         │─────────────>│        │          │
│    └─────────────────────────────────────┘              └────────┘          │
│                                                                             │
│    2 components = Simplified architecture, direct flow control              │
│    "Memory Play" spirit: minimize intermediate layers                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    DIRETTA SUMMARY TABLE                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Config        │ Host OS    │ Driver     │ Players/Control  │ Complexity   │
│  ══════════════│════════════│════════════│══════════════════│══════════════│
│                │            │            │                  │              │
│  2-Box Windows │ Windows    │ ASIO       │ Roon, Audirvana  │ **           │
│                │ 10/11      │ Diretta    │ JRiver, foobar   │              │
│                │            │            │ HQPlayer         │              │
│  ───────────────────────────────────────────────────────────────────────── │
│                │            │            │                  │              │
│  2-Box Linux   │ Linux      │ ALSA       │ Roon Server      │ ****         │
│                │ AudioLinux │ Diretta    │ HQP Embedded     │              │
│                │ GentooPlayer│           │ MPD, Squeezelite │              │
│  ───────────────────────────────────────────────────────────────────────── │
│                │            │            │                  │              │
│  3-Box UPnP    │ Linux      │ ALSA       │ MPD + upmpdcli   │ *****        │
│                │            │ Diretta    │ Control Points   │              │
│  ───────────────────────────────────────────────────────────────────────── │
│                │            │            │                  │              │
│  3-Box Roon    │ Linux      │ ALSA       │ Roon Bridge      │ ****         │
│                │            │ Diretta    │ + Roon Core      │              │
│  ───────────────────────────────────────────────────────────────────────── │
│                │            │            │                  │              │
│  Memory Play   │ Linux      │ Complete   │ Memory Play Ctrl │ *****        │
│  (PIC)         │            │ Bypass     │ (files only)     │              │
│  ───────────────────────────────────────────────────────────────────────── │
│                │            │            │ JPlay, MConnect  │              │             
│  Renderer UPnP │ Linux      │ Integrated │ Roon (via        │ ***          │
│  (Integrated   │            │ Host SDK   │ Squeeze Bridge)  │              │
│   SDK)         │            │            │                  │              │
└────────────────────────────────────────────────────────────────────────────┘
```

---

### Diretta DDS (Diretta Direct Stream): The Non-IPv6 Mode
Diretta DDS represents a major evolution of the protocol by completely eliminating the IP layer from audio transmission. Unlike the standard mode that uses IPv6 Link-Local and Multicast, DDS operates directly at the Ethernet level (Layer 2) via a dedicated EtherType (0x88B6). The initial connection is briefly established on UDP port 19642 for authentication, then the audio stream switches to raw Ethernet transmission, without any IP encapsulation. This approach offers several decisive advantages: complete elimination of the IP/TCP/UDP stack drastically reduces latency and CPU processing, audio data is aligned on 64-bit boundaries for maximum efficiency, and a dedicated Linux driver enables exclusive handling of communications. DDS mode pushes the Diretta philosophy to its peak by removing all superfluous protocol layers between the Host and Target

## Price

Diretta is licensed software, Diretta is not free, see https://www.diretta.link, contact Diretta or his partners.
The extraordinary quality of the development and the improvement in sound comes at a cost; nevertheless, the increase in quality is exceptional given the very reasonable price of the investment.

## Compatible Hardware

### Commercial Targets & Hosts

- See the website https://www.diretta.link

### DIY Targets & Hosts

With partners like Audiolinux or Gentooplayer or with Diretta firmware (https://www.diretta.link)

- Raspberry Pi 4/5
- x64 NUC or PC
- Ian Canada boards

### Sound Quality

In terms of sound quality, the different Diretta implementations are not equivalent. The Diretta Renderer UPnP with integrated Host SDK configuration offers very good results in terms of sound quality. Like Memory Play, the UPnP Renderer completely bypasses the ALSA system layer for direct hardware access, but it goes even further: it combines this complete bypass with streaming compatibility and online services (Roon via Squeeze Bridge, Qobuz, Tidal). This solution combines the advantages of the shortest audio path with modern usage flexibility.

---

## Conclusion

The Diretta philosophy remains constant across all implementations: **minimize influence on analog blocks and prioritize sound quality above all else**. Whether through a simple 2-box configuration, a 3-box architecture with role separation, the complete Memory Play bypass, or SDK integration in a UPnP Renderer, the goal is always to smooth system activity to reduce electrical disturbances that affect audio reproduction.

