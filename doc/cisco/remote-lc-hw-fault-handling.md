# Remote LC Hardware Fault Detection and Handling

## 1. Executive Summary

This document describes the high-level design for detecting and handling hardware faults on remote Line Cards (LCs) through the P2PM interrupt mechanism. The feature enables the RP to receive and process hardware failure events from remote LCs, providing visibility into critical faults such as CAT_ERR that can only be detected through this interrupt-based mechanism.

---

## 2. Problem Statement

### 2.1 Background

In a distributed chassis architecture with multiple Line Cards (LCs) and a Route Processor (RP), hardware faults occurring on remote LCs need to be detected and reported to the RP for proper fault management and system health monitoring.

### 2.2 Current Limitations

- **Limited Visibility:** The RP has no direct mechanism to detect hardware faults occurring on remote LCs
- **Critical Fault Detection:** Certain critical hardware faults (e.g., CPU CAT_ERR) can only be detected through interrupt mechanisms
- **Delayed Response:** Without interrupt-based notification, fault detection relies on polling or reactive discovery, leading to delayed responses
- **Incomplete Monitoring:** Power zone faults, CPU readiness changes, and other hardware events on LCs remain invisible to the RP

### 2.3 Requirements

1. **Real-time Fault Detection:** Detect hardware faults on remote LCs in real-time
2. **Fault Classification:** Support multiple interrupt types (8 general-purpose interrupts) for different fault categories
3. **Source Identification:** Identify which specific LC (LC0-LC17), FC (FC0-FC7), FT (FT0-FT3), or other module raised the interrupt
4. **Fault Granularity:** Support detailed fault information including power zone faults, CPU errors, and state changes

## 3. Architecture Overview

### 3.1 System Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Route Processor (RP)                          |
│                                                                      |
│  ┌─────────────────────────────────────────────────────────────────┐ |
│  │                    User Space Applications (Software)           │ │
│  │                      ┌──────────────────┐                       │ │
│  │                      │  Fault Manager   │                       | |
│  │                      │       (FM)       │                       | |
│  │                      └────────┬─────────┘                       | |
│  │                               │                                 | |
│  │  ┌────────────────────────────┴───────────────────────────────┐ │ |
│  │  │     LC Fault Monitor Agent (NEW COMPONENT)                 │ │ │
│  │  │  ┌──────────────────────────────────────────────────────┐  │ │ │
│  │  │  │ - Registers callbacks with  P2PM driver              │  │ │ │
│  │  │  │ - Processes GP interrupt notifications               │  │ │ │
│  │  │  │ - Reads rmt_gp_intr[n] to identify source cards      │  │ │ │
│  │  │  │ - Reads LC intSts registers via P2PM                 │  │ │ │
│  │  │  │ - Decodes faults based on card type                  │  │ │ │
│  │  │  │ - Interfaces with Fault Manager                      │  │ │ │
│  │  │  │ - Manages fault history and statistics               │  │ │ │
│  │  │  └──────────────────────────────────────────────────────┘  │ │ │
│  │  └────────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────┘ │
|  ╔═════════════════════════════════════════════════════════════════╗ │
│  ║                   KERNEL SPACE (Software Drivers)               ║ │
│  ╚═════════════════════════════════════════════════════════════════╝ │                                         
│                          │                                           │
│  ┌───────────────────────┴───────────────────────────────────────┐   │
│  │  ┌──────────────────────────────────────────────────────────┐ │   │
│  │  │  P2PM Master Driver (cisco-fpga-p2pm-m.c )               │ │   │
│  │  │  - Local interrupt enable/disable/status/clear           │ │   │
│  │  │  - Manages intr_stat, intr_en, intr_dis registers        | |   |
│  │  │  - ISR handler, Callback dispatch                        │ │   │
│  │  └──────────────────────────────────────────────────────────┘ │   │
│  |  ┌──────────────────────────────────────────────────────────┐ │   |
│  │  |  misc_intrsToRP IP block driver                          │ │   |
│  │  |  - Manages intSts0-7 status registers                    │ │   |
│  │  └──────────────────────────────────────────────────────────┘ |   │
│  └───────────────────────────────────────────────────────────────┘   │
│  ╔════════════════════════════════════════════════════════════════╗  │
│  ║                         HARDWARE                               ║  │
│  ╚════════════════════════════════════════════════════════════════╝  |
|                                                                      |
│                  P2PM Master Hardware Block                          │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Interrupt Control Registers                                    │ │
│  │  - intr_stat  (@0x1809c) - Shows which GP interrupt fired       │ │
│  │  - intr_en    (@0x180a4) - Enable interrupts                    │ │
│  │  - intr_dis   (@0x180a8) - Disable interrupts                   │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Remote GP Interrupt Source Registers                           │ │
│  │  - rmt_gp_intr0 (@0x18260) - Which card raised GP intr 0        │ │
│  │  - rmt_gp_intr1 (@0x18264) - Which card raised GP intr 1        │ │
│  │  - ... (rmt_gp_intr2-7)                                         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└────────────────────┬─────────────────────────────────────────────────┘
                     │ P2PM Link
                     │
┌────────────────────┴───────────────────────────────────────────────────┐
│              Line Card (LC) - Vanguard/Lancer                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │      Misc Interrupts Block (misc_intrsToRP @ 0xfa000)            │  │
│  │  - intSts0 (@0xfa020) - Power Zone & CPU Ready Events            │  │
│  │  - intSts1 (@0xfa030) - CPU CAT_ERR                              │  │
│  │  - intSts2-7 (@0xfa040-0xfa090) - Other faults (card-specific)   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘

```

## 4. Detailed Design

### 4.1 RP P2PM Master Block

#### 4.1.1 Interrupt Status Register (intr_stat @ 0x1809c)

**Purpose:** Top-level interrupt status showing which general-purpose interrupt was raised by remote cards.

**Register Layout:**

| Bit | Name | Access | Description |
|-----|------|--------|-------------|
| 24 | rmt_gp_intr7 | R/WOCLR | Remote GP interrupt 7 asserted |
| 23 | rmt_gp_intr6 | R/WOCLR | Remote GP interrupt 6 asserted |
| 22 | rmt_gp_intr5 | R/WOCLR | Remote GP interrupt 5 asserted |
| 21 | rmt_gp_intr4 | R/WOCLR | Remote GP interrupt 4 asserted |
| 20 | rmt_gp_intr3 | R/WOCLR | Remote GP interrupt 3 asserted |
| 19 | rmt_gp_intr2 | R/WOCLR | Remote GP interrupt 2 asserted |
| 18 | rmt_gp_intr1 | R/WOCLR | Remote GP interrupt 1 asserted |
| 17 | rmt_gp_intr0 | R/WOCLR | Remote GP interrupt 0 asserted |

**Characteristics:**
- Sticky bits (remain set until cleared by software)
- Write-one-clear (R/WOCLR) - writing 1 clears the bit
- Each bit maps to one of 8 general-purpose interrupt channels

#### 4.1.2 Remote GP Interrupt Source Registers

**Purpose:** Identify which specific card/module raised the interrupt.

**Register:** rmt_gp_intr0 @ 0x18260 (Power Error Interrupts)

| Bits | Description |
|------|-------------|
| 17:0 | LC17 ~ LC0 (18 Line Cards) |
| 25:18 | FC7 ~ FC0 (8 Fabric Cards) |
| 29:26 | FT3 ~ FT0 (4 Fan Trays) |
| 30 | Peer RP |
| 31 | Local BMC FPGA (Kingman) |

**Registers:** rmt_gp_intr1-7 @ 0x18264-0x1827c

Similar bit mapping for different interrupt types (mapping defined per HWFS).

### 4.2 LC Misc Interrupts Block (Vanguard/Lancer)

#### 4.2.1 Interrupt Status Registers (intSts0-7)

**Base Address:** misc_intrsToRP @ 0xfa000

**Register:** intSts0 @ 0xfa020 (Power Zone and CPU Ready Events)

| Bits | Name | Description |
|------|------|-------------|
| 30 | intrpt30 | CPU ready rising edge |
| 29 | intrpt29 | CPU ready falling edge |
| 14 | intrpt14 | ZONE3 power cycling |
| 13 | intrpt13 | ZONE3 powering off |
| 12 | intrpt12 | ZONE3 powered off |
| 11 | intrpt11 | ZONE3 powering on |
| 10 | intrpt10 | ZONE3 powered on |
| 9 | intrpt9 | ZONE3 fault1 |
| 8 | intrpt8 | ZONE3 fault0 |
| 7 | intrpt7 | ZONE12 power cycling |
| 6 | intrpt6 | ZONE12 powering off |
| 5 | intrpt5 | ZONE12 powered off |
| 4 | intrpt4 | ZONE12 powering on |
| 3 | intrpt3 | ZONE12 powered on |
| 2 | intrpt2 | ZONE12 fault1 |
| 1 | intrpt1 | ZONE12 fault0 |
| 0 | intrpt0 | ZONE0 fault0 |

**Register:** intSts1 @ 0xfa030 (CPU Catastrophic Error)

| Bits | Name | Description |
|------|------|-------------|
| 0 | intrpt0 | CPU CAT_ERR (Catastrophic Error) |
| 1-31 | intrpt1-31 | Reserved/TBD |

**Registers:** intSts2-7 @ 0xfa040-0xfa090

## 5. Software Design

### 5.1 Interrupt Flow

1. **Fault Generation:** Hardware fault occurs on LC (e.g., CPU CAT_ERR, power zone fault)
2. **Local Registration:** LC's misc_intrsToRP block registers the fault in appropriate intSts register
3. **P2PM Transmission:** LC sends interrupt to RP via P2PM slave-to-master communication
4. **RP Reception:** RP's P2PM master block receives interrupt, sets corresponding bit in intr_stat
5. **ISR Processing:** Reads intr_stat register to see which interrupts are active (bits 17-24)
6. **Callback Dispatch:** ISR calls registered callback (LC Fault Monitor Agent)
7. **Agent Processing:**
    - Reads rmt_gp_intr[n] register to identify which cards raised the interrupt
    - Reads LC's interrupt status registers to determine specific fault 
    - Report to Fault Manager for logging, generates alarms, triggers recovery
8. **Fault Handling:** Agent reports faults to FM for system-wide fault correlation (logging, recovery, alarm generation)


```
┌─────────────────────────────────────────────────────────┐
│  RP Interrupt Handler Entry                             │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Read intr_stat register (@0x1809c)                     │
│    - Check bits 17-24 for active GP interrupts          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  For each active interrupt bit                          │
│     - Call registered callback (LC Fault Monitor)       │
└────────────────────┬────────────────────────────────────┘
                     │ Callback execution
                     ▼
┌─────────────────────────────────────────────────────────┐
│  LC Fault Monitor Agent Processing (USER SPACE)         │
├─────────────────────────────────────────────────────────|
│  1. Read rmt_gp_intr[n] register (@0x18260 + n*4)       │
│     - Decode card bitmap (bits indicate which cards)    │
│       * bits [17:0]  = LC0-LC17                         │
│       * bits [25:18] = FC0-FC7                          │
│       * bits [29:26] = FT0-FT3                          │
│       * bit  [30]    = Peer RP                          │
│       * bit  [31]    = Local BMC FPGA                   │
│  2. For identified LC:                                  │
│     - Read LC's misc_intrsToRP base + intSts[0-7]       │
│     - Parse active interrupt bits                       │
│     - Map to specific fault types                       │
│  3. Process Faults:                                     │
│     - Log fault details                                 │
│     - Report to Fault Manager                           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  FM infra                                               │
│    - Fault handling and recovery based on fault type    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Clear Interrupts:                                      │
│     - Write 1 to clear LC's intSts register bits        │
│     - Write 1 to clear RP's rmt_gp_intr[n] bits         │
│     - Write 1 to clear RP's intr_stat bits              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Return from Interrupt Handler                          │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Fault Handling

#### 5.2.1 Fault Types and Actions

| Fault Type | Description | Action |
|------------|-------------|--------|
| CPU CAT_ERR | CPU catastrophic error | Log, alarm, consider LC reset/isolation |
| Power Zone Fault | Power delivery failure | Log, alarm |

Additional interrupt status registers for other fault types

**Characteristics:**
- All bits are R/WOCLR (write-one-clear)
- Sticky bits that remain set until cleared
- 32-bit registers providing detailed fault information

### 5.3 Card-Specific Handling

Supporting LC- Vanguard, Lancer for now

## 6. Testing Strategy
Unit tests and fault injection tests to cover the following:
- Specific Fault Simulation 
- Detection Verification 
- Logging and Error Recovery 

