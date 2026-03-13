# ECN Cumulative Packet Counter

## Table of Content

- [ECN Cumulative Packet Counter](#ecn-cumulative-packet-counter)
  - [Table of Content](#table-of-content)
  - [Revision](#revision)
  - [Scope](#scope)
  - [Definitions/Abbreviations](#definitionsabbreviations)
  - [Overview](#overview)
  - [Requirements](#requirements)
  - [Architecture Design](#architecture-design)
  - [High-Level Design](#high-level-design)
  - [SAI API](#sai-api)
  - [Configuration and management](#configuration-and-management)
    - [CLI Changes](#cli-changes)
      - [CLI output when counterpoll wredqueue enable](#cli-output-when-counterpoll-wredqueue-enable)
      - [CLI output when counterpoll wredqueue disable](#cli-output-when-counterpoll-wredqueue-disable)
  - [Warmboot and Fastboot Design Impact](#warmboot-and-fastboot-design-impact)
  - [Restrictions/Limitations](#restrictionslimitations)
  - [Testing Requirements/Design](#testing-requirementsdesign)
    - [Unit Test cases](#unit-test-cases)
    - [System Test cases](#system-test-cases)

## Revision

| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 12/15/2025  |   Zhixin Zhu       | Initial version                   |

## Scope

This document provides the high level design for ECN cumulative packet counter in SONiC.

## Definitions/Abbreviations

|  Abbreviation | Description                     |
|:-------------:|---------------------------------|
| __ECN__       | Explicit Congestion Notification|
| __WRED__      | Weighted Random Early Detection |
| __CLI__       | Command Line interface          |
| __SAI__       | Switch Abstraction Interface    |

## Overview

The main goal of this feature is to provide better ECN impact visibility in SONiC by providing a mechanism to have aggregate counters for packets that are ECN-marked.

## Requirements

1. Support per-asic total ECN marked packets counters.
2. Support per-asic total WRED drop packets and bytes counters.

## Architecture Design

There are no architectural design changes as part of this design.

## High-Level Design

This section covers the high level design of the WRED and ECN aggregate counters. The following step-by-step short description provides an overview of the operations involved in this feature.

1. Orchagent fetches the platform statistics capability for WRED and ECN Statistics from SAI.
2. The stats capability will be updated to STATE_DB by orchagent
3. Based on the stats capability and CONFIG_DB status of respective statistics, Orchagent sets stat-ids to FLEX_COUNTERS_DB
    * In case, the platform is capable of WRED and ECN statistics,
        * Per-queue WRED and ECN counters will create and use the new flexcounter group WRED_ECN_QUEUE
4. Syncd has subscribed to Flex Counter DB and it will set up flex counters.
5. Flex counters periodically query platform counters and publishes data to COUNTERS_DB
6. CLI will look-up the Capability at STATE_DB
7. Only the supported statistics will be fetched and displayed on the CLI output
8. CLI for cumulative counter will add up the WRED and ECN counters per asic

## SAI API

Following SAI statistics are used in this feature,

* SAI counters list,
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_PACKETS
    * SAI_QUEUE_STAT_WRED_ECN_MARKED_BYTES
    * SAI_QUEUE_STAT_WRED_DROPPED_PACKETS
    * SAI_QUEUE_STAT_WRED_DROPPED_BYTES

* sai_query_stats_capability() will be used to identify the supported statistics

## Configuration and management

The config CLI commands ```"counterpoll wredqueue <enable | disable>"``` will enable and disable the WRED and ECN statistics by interacting with the FLEX_COUNTER_TABLE table of CONFIG_DB.

### CLI Changes

There is a new CLI introduced for this feature, it will display the aggregated WRED and ECN statistics per asic.

    * Display the statistics on the console      : ```show queue wredcounters summary -n <namespace>```

#### CLI output when counterpoll wredqueue enable

Existing CLI
```
sonic-dut:~# show queue wredcounters Ethernet160
       Port    TxQ    WredDrp/pkts    WredDrp/bytes    EcnMarked/pkts    EcnMarked/bytes
-----------  -----  --------------  ---------------  ----------------  -----------------
Ethernet160    UC0               0                0                 0                N/A
Ethernet160    UC1               0                0                 0                N/A
Ethernet160    UC2               0                0                 0                N/A
Ethernet160    UC3               0                0        3615805119                N/A
Ethernet160    UC4               0                0                 0                N/A
Ethernet160    UC5               0                0                 0                N/A
Ethernet160    UC6               0                0                 0                N/A
Ethernet160    UC7               0                0                 0                N/A
```

New CLI
```
sonic-dut:~# show queue wredcounters summary -n asic0
  TxQ    WredDrp/pkts    WredDrp/bytes    EcnMarked/pkts    EcnMarked/bytes
-----  --------------  ---------------  ----------------  -----------------
  UC0               1              120                 0                N/A
  UC1               2              240                 0                N/A
  UC2               0                0                 0                N/A
  UC3               0                0        3615805119                N/A
  UC4               0                0                 0                N/A
  UC5               0                0                 0                N/A
  UC6               0                0                 0                N/A
  UC7               0                0                 0                N/A

Total               3              360        3615805119                N/A
```

#### CLI output when counterpoll wredqueue disable

Existing CLI
```
sonic-dut:~# show queue wredcounters Ethernet120
       Port    TxQ    WredDrp/pkts    WredDrp/bytes    EcnMarked/pkts    EcnMarked/bytes
-----------  -----  --------------  ---------------  ----------------  -----------------
Ethernet120    UC0             N/A              N/A               N/A                N/A
Ethernet120    UC1             N/A              N/A               N/A                N/A
Ethernet120    UC2             N/A              N/A               N/A                N/A
Ethernet120    UC3             N/A              N/A               N/A                N/A
Ethernet120    UC4             N/A              N/A               N/A                N/A
Ethernet120    UC5             N/A              N/A               N/A                N/A
Ethernet120    UC6             N/A              N/A               N/A                N/A
Ethernet120    UC7             N/A              N/A               N/A                N/A
```

New CLI
```
sonic-dut:~# show queue wredcounters summary -n asic0
  TxQ    WredDrp/pkts    WredDrp/bytes    EcnMarked/pkts    EcnMarked/bytes
-----  --------------  ---------------  ----------------  -----------------
  UC0             N/A              N/A               N/A                N/A
  UC1             N/A              N/A               N/A                N/A
  UC2             N/A              N/A               N/A                N/A
  UC3             N/A              N/A               N/A                N/A
  UC4             N/A              N/A               N/A                N/A
  UC5             N/A              N/A               N/A                N/A
  UC6             N/A              N/A               N/A                N/A
  UC7             N/A              N/A               N/A                N/A

Total             N/A              N/A               N/A                N/A
```

## Warmboot and Fastboot Design Impact
There are no impact to warmboot or fastboot.


## Restrictions/Limitations
* SAI_QUEUE_STAT_WRED_ECN_MARKED_BYTES is not supported.

## Testing Requirements/Design

### Unit Test cases
- Disable counterpoll wredqueue, Verify the CLI doesn't display WRED and ECN Queue statistics summary.
- Enable counterpoll wredqueue
  - Verify the CLI display WRED and ECN Queue statistics summary correctly.
  - Send traffic to 2 egress ports and create congestion, verify that the summary updates the ECN marked counters correctly (the summary of the 2 egress ports).
  - Verify the CLI on both T0/T1 and T2 platforms.

### System Test cases
* New sonic-mgmt(PTF) ecn wred statistics summary testcase will be created to verify the statistics summary on supported platforms.
