# Memory and CPU Monitoring Enhancement

## Table of Contents
- [Scope](#scope)
- [Overview](#overview)
- [Requirements](#requirements)
- [High-Level Design](#high-level-design)
    - [Memory Gradual Increase Detection](#memory-gradual-increase-detection)
    - [Architecture](#architecture)
    - [State Format](#state-format)
    - [Monit Configuration](#monit-configuration)
    - [Example Flows](#example-flows)
    - [Output Format](#output-format)
    - [CPU Monitoring Enhancement](#cpu-monitoring-enhancement)
- [Testing Considerations](#testing-considerations)
- [Future Enhancements](#future-enhancements)

## Scope

This document describes the high-level design of memory and CPU monitoring enhancements in SONiC.

## Terminology

**OOM (Out of Memory):** A critical system state where available memory is exhausted. OOM can be caused by (a) memory leak, (b) misconfigured limits, or (c) insufficient resources.

**OOM Killer:** A Linux kernel mechanism that terminates processes to reclaim memory when the system reaches an OOM condition. In SONiC, OOM Killer events can crash critical containers (syncd, bgp, orchagent), leading to service disruptions.

**RSS (Resident Set Size):** The portion of a process's memory held in RAM, excluding swapped memory.

**Linear Regression:** A statistical method that fits a straight line through a series of data points to identify trends. Simple linear regression uses one independent variable (time) to predict one dependent variable (memory usage).

**Slope:** The rate of change in a linear trend, measured in MB/min for memory growth.

**R² (R-squared):** A statistical confidence metric (0.0 to 1.0) indicating how well data fits a linear trend. Higher values indicate stronger linear patterns.


## Overview

Current memory monitoring is threshold-based (triggers at 90% memory usage) and fires at late stage when system is already at high memory. Memory leaks that grow slowly over hours or days go undetected until threshold is breached, by which time the system may be near OOM.

This enhancement adds:
1. **Memory gradual increase detection** - Statistical trend-based detection using linear regression across three time scales to predict time-to-OOM
2. **CPU monitoring** - Process-level logging when existing CPU threshold is breached

## High-Level Design

### Memory Gradual Increase Detection

This enhancement uses **linear regression** to detect memory trends and predict time-to-OOM across three time scales. Each time scale operates independently:

**Time scales:**
- Short: 2 hours (24 samples, check every 5 min) - Catches fast leaks (hours to OOM)
- Medium: 24 hours (24 samples, check every 60 min) - Catches medium leaks (days to OOM)
- Long: 7 days (21 samples, check every 8 hours) - Catches slow leaks (weeks to OOM)

**Cycle:** Monit runs checks every cycle (1 cycle = 1 minute).

**Detection methodology:**
1. Collect memory samples at regular intervals into a sliding window
2. Run linear regression on the window to calculate:
   - **Slope** (MB/min growth rate)
   - **R²** (confidence in trend, 0.0-1.0)
3. Calculate **time-to-90%**: How long until memory reaches 90% threshold at current growth rate
4. Alert if:
   - Time-to-90% is less than threshold (e.g., < 6 hours for short window)
   - Memory grew significantly as % of free memory (e.g., > 5% of free mem for short window)
   - Trend is statistically significant (R² > 0.7)

This approach is **self-calibrating** (adapts to any RAM size) and **contextual** (considers available headroom when calculating urgency).

### Architecture

**Components:**

| Component | Function | Output |
|-----------|----------|--------|
| Checker | Sample collection, regression, detection | Exit code: 0 or 2 |
| Handler | Logging, process attribution | Syslog (INFO level) |
| Monit | Scheduling, exit code monitoring | Triggers handler on exit 2 |

**Monit:** Process and resource monitoring daemon. Runs checks at configured intervals, executes actions based on exit codes.

**Detection algorithm:**

**Note:** Detection triggers on system-wide memory growth (total RAM usage trending toward OOM). Per-process memory is tracked only for attribution—to identify which process caused the growth after detection fires.

1. **Data collection** (every check interval):
   - Collect system memory: `psutil.virtual_memory().used`
   - Collect processes meeting criteria: RSS >0.5% of total RAM (max 100 processes)
   - Append to sliding windows (system + filtered per-process)
   - Rationale: Self-scales to RAM size (e.g., >156 MB on 31 GB system), caps storage at ~100 processes

2. **Regression calculation** (when window is full):
   - Run linear regression on system memory samples using Python stdlib (`statistics.linear_regression`, Python 3.10+; no NumPy/SciPy)
   - Calculate system slope (MB/min) and R² (goodness of fit)
   - Store all process memory windows for handler analysis

3. **Threshold checks**:
   ```
   headroom = (total_ram * 0.9) - current_memory
   time_to_90pct = headroom / slope  (in minutes)
   
   # Growth measured as % of FREE memory (self-scaling)
   free_memory = total_ram - current_memory
   growth_mb = current_memory - baseline_memory
   percent_of_free = (growth_mb / free_memory) * 100
   
   # Detection logic: High confidence trend (R² > 0.7) AND (time urgent OR growth significant)
   time_triggered = (time_to_90pct < threshold_time)
   growth_triggered = (percent_of_free > threshold_percent)
   
   if (r_squared > 0.7 AND (time_triggered OR growth_triggered)):
       trigger alert (exit code 2)
   ```

4. **Alert thresholds per time scale**:
   - **Short (2h)**: time_to_90% < 6 hours, growth > 5% of free memory
   - **Medium (24h)**: time_to_90% < 3 days, growth > 10% of free memory
   - **Long (7d)**: time_to_90% < 21 days, growth > 10% of free memory
   
   **Growth calculation**: Growth is measured as percentage of **free memory** (not total RAM). This makes the threshold self-scaling - on a system with less free memory, smaller absolute growth triggers detection, which is appropriate since the system is closer to exhaustion.

**Process attribution:** When alert fires, handler:
1. Reads per-process and per-container memory windows from state file
2. Runs linear regression on each to calculate slope and R²
3. Collects current state:
   - Process memory via `psutil` (top 15 by RSS)
   - Container memory via `docker stats --no-stream --format` (top 15, 10s timeout; falls back to process-only on failure)
4. Filters contributors using criteria:
   - Contribution threshold: Growth ≥ 10% of system growth (self-scaling)
   - Consistent trend: R² > 0.5 (filters spikes/oscillations)
5. Builds output lists (5-10 items each):
   - **Processes**: Growth contributors first (with growth stats + R²), then top consumers by RSS
   - **Containers**: Growth contributors first (with growth stats + R²), then top consumers
6. Reports each process on separate line with PID, name, memory, cmdline
7. Reports all containers on one line (semicolon-separated)

**Process restart handling:** Processes are tracked by name (not PID). If a process restarts during the window, its memory pattern shows a drop then recovery (non-linear), resulting in low R². The R² > 0.5 filter naturally excludes restarted processes from being reported as leak contributors, preventing false attribution. Only processes with consistent linear growth are identified as contributors.

**Resource overhead:** Measured on Cisco 8000 (30.6 GB RAM, ~865 processes):

| Operation | Time | Frequency |
|-----------|------|-----------|
| System memory (`virtual_memory()`) | ~0.1 ms | Every sample |
| Process enumeration + filter | ~140 ms | Every sample |
| **CPU impact (short, every 5 min)** | **0.047%** | Highest frequency |
| CPU impact (medium, every 60 min) | 0.004% | |
| CPU impact (long, every 8 hours) | Negligible | |
| State file I/O | ~8 KB (typical) | Every sample |

Per-process collection is done at every sample (not deferred to handler) because attribution requires historical windows to calculate per-process slope and R². A single snapshot at alert time cannot distinguish a process that leaked +1200 MB from one that always uses 1200 MB.

### State Format

**File:** `/var/run/memory_gradual_{short,medium,long}.json`

**Storage location:** `/var/run` (tmpfs, RAM-backed). No flash wear from frequent writes. Typical total ~25 KB across all three files (negligible relative to system RAM).

**Purpose:** State file stores sliding windows of memory samples, bridging the checker and handler components:
- **Checker (writer)**: Collects and appends system memory + per-process/per-container samples at each check interval
- **Handler (reader)**: When detection fires, reads windows to run per-process/per-container regression and identify growth contributors
- **Detection vs Attribution**: `system_memory_window` triggers alerts (system-wide trend), `tracked_processes`/`tracked_containers` identify culprits (attribution)

```json
{
  "window_type": "short",
  "window_size": 24,
  "sample_count": 18,
  
  "system_memory_window": [4400.0, 4450.0, 4500.0, ..., 5200.0],
  
  "tracked_processes": {
    "syncd": [1200.0, 1210.0, 1220.0, ..., 1320.0],
    "bgp": [850.0, 852.0, 855.0, ..., 880.0],
    "orchagent": [320.0, 322.0, 325.0, ..., 350.0],
    "swss": [260.0, 261.0, 262.0, ..., 270.0],
    "pmon": [235.0, 235.0, 236.0, ..., 238.0],
    "...additional processes...": []
  },
  
  "tracked_containers": {
    "syncd": [1250.0, 1260.0, 1270.0, ..., 1370.0],
    "bgp": [200.0, 201.0, 202.0, ..., 210.0],
    "pmon": [450.0, 451.0, 452.0, ..., 460.0],
    "database": [110.0, 111.0, 111.0, ..., 115.0],
    "...additional containers...": []
  },
  
  "baseline_snapshot": {
    "timestamp": "2026-02-06T10:00:00.123456",
    "memory_mb": 4400.0,
    "total_ram_mb": 20000.0
  },
  
  "last_regression": {
    "slope_mb_per_min": 3.42,
    "r_squared": 0.87,
    "calculated_at_sample": 18
  }
}
```

**Field Descriptions:**
- `window_type`: Time scale identifier ("short", "medium", or "long")
- `window_size`: Maximum number of samples to store (24/24/21 for short/medium/long)
- `sample_count`: Current number of samples collected (1 to window_size)
- `system_memory_window`: Array of system-wide memory readings (MB). Used for system-level regression.
- `tracked_processes`: Per-process memory windows for processes >0.5% of total RAM (max 100). Each process has array of memory readings (MB) over the window. Process list may change between samples as processes cross thresholds. Used by handler to calculate per-process regression and filter false positives.
- `tracked_containers`: Per-container memory windows collected via `docker stats`. Each container has array of memory readings (MB) over the window. Used by handler to calculate per-container regression and identify leaking containers.
- `baseline_snapshot`: Metadata captured at window start (sample 1)
  - `timestamp`: When window started
  - `memory_mb`: System memory at baseline
  - `total_ram_mb`: Total system RAM (for calculations)
- `last_regression`: Cached system-level regression results (updated every 5-10 samples to reduce CPU)
  - `slope_mb_per_min`: System memory growth rate from linear regression
  - `r_squared`: Confidence metric (0.0-1.0) for system trend
  - `calculated_at_sample`: Which sample number regression was last calculated

**Storage size (measured on Cisco 8000):**
- Typical (8 processes, 15 containers): ~8 KB per file, **~25 KB total**
- Worst case (100 processes, 20 containers): ~40 KB per file, ~120 KB total
- File is reset when window fills and detection fires

**Lifecycle:** Created on first run, updated at each check interval, reset when window fills and detection fires.

### Monit Configuration

Monit configuration in `/etc/monit/conf.d/sonic-host`:

```
check program memory_gradual_short with path "/usr/bin/memory_gradual_check.py --scale short"
    every 5 cycles
    if status == 2 for 1 cycles then exec "/usr/bin/memory_gradual_handler.py --scale short"

check program memory_gradual_medium with path "/usr/bin/memory_gradual_check.py --scale medium"
    every 60 cycles
    if status == 2 for 1 cycles then exec "/usr/bin/memory_gradual_handler.py --scale medium"

check program memory_gradual_long with path "/usr/bin/memory_gradual_check.py --scale long"
    every 480 cycles
    if status == 2 for 1 cycles then exec "/usr/bin/memory_gradual_handler.py --scale long"
```

**Explanation:**
- `every N cycles`: Check interval per-check (1 cycle = 1 minute from `set daemon 60` in monitrc). The checker script (Python + psutil) only launches at the configured interval—not every cycle. No wasted overhead between samples.
  - Short: Every 5 minutes (24 samples = 2 hour window)
  - Medium: Every 60 minutes (24 samples = 24 hour window)
  - Long: Every 480 minutes (8 hours; 21 samples = 7 day window)
- `if status == 2`: Triggers when checker returns exit code 2 (detection). Exit code 0 = normal.
- `then exec`: Runs handler script to log details
- Three independent checks for three time scales

**Time scale summary:**

| Scale | Check Interval | Window Size | Samples | Target Leaks |
|-------|---------------|-------------|---------|-------------|
| Short | 5 min | 2 hours | 24 | Fast leaks (hours to OOM) |
| Medium | 60 min | 24 hours | 24 | Medium leaks (days to OOM) |
| Long | 480 min (8h) | 7 days | 21 | Slow leaks (weeks to OOM) |

### Example Flows

**Flow 1: Fast Memory Leak Detection (Short Window)**

```
System: 20 GB RAM, currently at 4000 MB (20%)
Window: Short (2 hours, 24 samples every 5 min)

Sample 1:  Memory = 4000 MB → Baseline captured, window = [4000]
Sample 5:  Memory = 4400 MB → window = [4000, 4100, 4200, 4300, 4400]
Sample 10: Memory = 4900 MB → window = [4000...4900]
Sample 24: Memory = 6000 MB → Window full [4000...6000]

  Regression:
    Slope = 17.4 MB/min (+1044 MB/hour)
    R² = 0.92 (high confidence)
  
  Thresholds check:
    Headroom = 18000 - 6000 = 12000 MB
    Time to 90% = 12000 / 17.4 = 690 min = 11.5 hours
    
    Free memory = 20000 - 6000 = 14000 MB
    Growth = 6000 - 4000 = 2000 MB
    Growth as % of free = (2000 / 14000) * 100 = 14.3%
    
    R² > 0.7? YES (0.92)
    Time triggered: 11.5 hours < 6 hours? NO
    Growth triggered: 14.3% > 5% (short threshold)? YES
    
    R² > 0.7 AND (time OR growth)? YES (0.92 > 0.7 AND growth triggered)
    → Exit code 2 → Handler logs fast leak detected
  
  Process attribution (contribution-based filtering):
    System growth: +2000 MB
    Min contribution threshold: 2000 × 10% = 200 MB
    
    orchagent: 320 → 1200 MB (+880 MB, R² = 0.91) ✓ Reports (880 > 200, high R²)
    syncd:     1200 → 1850 MB (+650 MB, R² = 0.88) ✓ Reports (650 > 200, high R²)
    bgp:       850 → 970 MB (+120 MB, R² = 0.35) ✗ Filtered (120 < 200)
    swss:      260 → 340 MB (+80 MB, R² = 0.72) ✗ Filtered (80 < 200)
```
**Result:** Fast leak (1 GB/hour) caught in 2 hours

**Flow 2: Slow Leak Detection (Long Window)**

```
System: 20 GB RAM, currently at 4000 MB (20%)
Window: Long (7 days, 21 samples every 8 hours)

Sample 1:  Memory = 4000 MB (Day 0)
Sample 7:  Memory = 4500 MB (Day 2)
Sample 14: Memory = 5000 MB (Day 4)
Sample 21: Memory = 6000 MB (Day 7) → Window full

  Regression:
    Slope = 0.2 MB/min (+12 MB/hour, +286 MB/day)
    R² = 0.94 (very high confidence - clear linear trend)
  
  Thresholds check:
    Headroom = 18000 - 6000 = 12000 MB
    Time to 90% = 12000 / 0.2 = 60,000 min = 41.7 days
    
    Free memory = 20000 - 6000 = 14000 MB
    Growth = 6000 - 4000 = 2000 MB
    Growth as % of free = (2000 / 14000) * 100 = 14.3%
    
    R² > 0.7? YES (0.94)
    Time triggered: 41.7 days < 21 days? NO
    Growth triggered: 14.3% > 10% (long threshold)? YES
    
    R² > 0.7 AND (time OR growth)? YES (0.94 > 0.7 AND growth triggered)
    → Exit code 2 → Handler logs slow leak detected
  
  Process attribution (contribution-based filtering):
    System growth: +2000 MB
    Min contribution threshold: 2000 × 10% = 200 MB
    
    syncd: 1200 → 2400 MB (+1200 MB, R² = 0.96) ✓ Reports (1200 > 200, very high R²)
    bgp:   850 → 1250 MB (+400 MB, R² = 0.94) ✓ Reports (400 > 200, very high R²)
    pmon:  235 → 275 MB (+40 MB, R² = 0.85) ✗ Filtered (40 < 200)
```
**Result:** Slow persistent leak (286 MB/day) caught in 7 days despite long time-to-OOM

**Flow 3: Oscillating Memory (No False Alert)**

```
System: 20 GB RAM, cache fluctuations
Window: Medium (24 hours, 24 samples every 60 min)

Sample 1:  Memory = 4400 MB
Sample 6:  Memory = 4800 MB
Sample 12: Memory = 4300 MB
Sample 18: Memory = 4900 MB
Sample 24: Memory = 4500 MB → Window full

  Regression:
    Slope = 0.07 MB/min (very small)
    R² = 0.15 (low confidence - data is scattered, not linear)
  
  Thresholds check:
    Time to 90% = very large (slow slope)
    
    Free memory = 20000 - 4500 = 15500 MB
    Growth = 4500 - 4400 = 100 MB
    Growth as % of free = (100 / 15500) * 100 = 0.6%
    
    R² > 0.7? NO (0.15)
    
    R² > 0.7 AND (time OR growth)? NO (0.15 < 0.7)
    → Exit code 0 → No alert (normal oscillation)
```
**Result:** Normal memory fluctuation (cache churn) does not trigger false alert due to low R²

**Flow 4: Spike Then Recovery (No False Alert)**

```
System: 20 GB RAM, config reload causes temporary spike
Window: Short (2 hours, 24 samples every 5 min)

Sample 1-10: Memory steady at 4400 MB (50 min)
Sample 11:   Memory spikes to 8000 MB (config reload)
Sample 12-24: Memory drops back to 4600 MB (70 min)

  Regression:
    Slope = 1.7 MB/min (small positive slope)
    R² = 0.35 (low confidence - spike creates non-linear pattern)
  
  Thresholds check:
    Free memory = 20000 - 4600 = 15400 MB
    Growth = 4600 - 4400 = 200 MB
    Growth as % of free = (200 / 15400) * 100 = 1.3%
    
    R² > 0.7? NO (0.35)
    
    R² > 0.7 AND (time OR growth)? NO (0.35 < 0.7)
    → Exit code 0 → No alert (transient spike, not a leak)
```
**Result:** Temporary spike filtered out by low R² and low net growth

### Output Format

**Log format:** Structured syslog entries with essential information for investigation.

```
INFO memory_gradual_handler: Gradual memory increase detected (window: 24 hours)
INFO memory_gradual_handler:   Current: 8800MB used, 11200MB free (44.0%) ; Growth: 4400MB -> 8800MB (+4400MB, +100.0%, R²=0.94), time to 90%: 2.0 days
INFO memory_gradual_handler:   Memory-consuming processes:
INFO memory_gradual_handler:     #1 PID:123 orchagent - +1330MB (+416%, R²=0.95) - /usr/bin/orchagent -d
INFO memory_gradual_handler:     #2 PID:456 syncd - +2200MB (+183%, R²=0.92) - /usr/bin/syncd -u -s
INFO memory_gradual_handler:     #3 PID:789 bgp - 800MB (4.0%) - /usr/lib/frr/bgpd
INFO memory_gradual_handler:     #4 PID:234 swss - 350MB (1.8%) - /usr/bin/swss -d
INFO memory_gradual_handler:     #5 PID:567 redis-server - 320MB (1.6%) - /usr/bin/redis-server
INFO memory_gradual_handler:   Memory-consuming containers: syncd: 2250MB (11.3%); orchagent: 1350MB (6.8%); bgp: 680MB (3.4%); swss: 290MB (1.5%); database: 160MB (0.8%)
```

**Field explanations:**
- **Line 1**: Detection header with window type (2 hours/24 hours/7 days)
- **Line 2**: System-level summary
  - Current memory (used, free, percentage)
  - Growth over window (baseline → current, absolute, percentage, R²)
  - Time to 90%: Predicted time until reaching 90% threshold
- **Line 3+**: Memory-consuming processes (each on separate line)
  - **Growth contributors** (first): Processes with ≥10% of system growth AND R² > 0.5
    - Shows: PID, name, growth (absolute + percentage + R²), command line
  - **Top consumers** (fill to 5-10): Largest processes by current RSS
    - Shows: PID, name, current memory (absolute + percentage), command line
- **Last line**: Memory-consuming containers (all on one line, semicolon-separated)
  - Same ordering: growth contributors first, then top consumers
  - Shows: name, memory (absolute + percentage) or growth stats if contributor

**Output sizing:** 5-10 processes, 5-10 containers (min 5 each, max 10 each)

**Total log overhead:** 4+ lines per detection (header + summary + process header + N processes + containers)

### CPU Monitoring Enhancement

When existing monit CPU threshold is breached, log detailed process-level CPU information:
- Top CPU-consuming processes with usage percentages
- Process command lines for identification
- CPU usage distribution across processes

Implementation: Handler script triggered by existing monit CPU check, logs to syslog (INFO level).

**Output Format:**

```
INFO cpu_threshold_check: CPU usage user=92.5% system=85.3% exceeds threshold 90%. Top 5 CPU consumers:
INFO cpu_threshold_check:   #1 PID:1234 orchagent - CPU:45.2% MEM:450MB - /usr/bin/orchagent
INFO cpu_threshold_check:   #2 PID:5678 syncd - CPU:38.5% MEM:380MB - /usr/bin/syncd
INFO cpu_threshold_check:   #3 PID:2345 bgpd - CPU:12.3% MEM:250MB - /usr/lib/frr/bgpd
INFO cpu_threshold_check:   #4 PID:3456 python3 - CPU:8.7% MEM:180MB - /usr/bin/portsyncd
INFO cpu_threshold_check:   #5 PID:4567 redis-server - CPU:6.2% MEM:320MB - /usr/bin/redis-server
INFO cpu_threshold_check: Container CPU: swss(52.3%), syncd(41.2%), bgp(15.8%), database(8.5%)
```

## Testing Considerations

### Test Strategy

The testing approach follows a phased methodology to ensure comprehensive validation before production deployment:

**Phase 1: Testbed Setup**
Bring up spytest testbed environment to provide a controlled testing infrastructure that mirrors production topology and configuration.

**Phase 2: Unit and Functional Testing**
Execute comprehensive test suite covering:
- **Basic Detection**: Simulate gradual memory increase to verify linear regression detects trends, calculates correct slope and R² values, triggers detection when thresholds are met, and logs expected output format
- **Contribution Filtering**: Run multiple processes with different growth patterns to verify contribution-based filtering correctly identifies processes that grew ≥10% of system growth with high R² (>0.5)
- **False Positive Prevention - Spikes**: Simulate temporary memory spikes to verify non-linear patterns result in low R² values and do not trigger false alerts
- **False Positive Prevention - Oscillations**: Simulate normal cache behavior with oscillating memory to verify scattered data patterns result in low R² values and do not falsely detect leaks
- **Time-to-90% Calculation**: Validate accuracy of time-to-90% predictions with known memory growth rates
- **CPU Monitoring Integration**: Trigger CPU threshold breach to verify handler logs top processes with CPU and memory usage along with container-level CPU aggregation

For rapid validation, modify time constants in `memory_gradual_check.py` to use accelerated sampling intervals (e.g., 30 seconds instead of 5 minutes). To avoid duplicate detections during testing with identical fast configurations, enable only one window check at a time in monit configuration by commenting out the other two.

**Phase 3: System-Level Validation**
Execute Jenkins ring 3 run against merged codebase, followed by syslog validation to verify:
- Detection events are logged with correct format and detail level
- No unexpected errors or warnings in system logs

**Phase 4: Soak Testing**
Build image with all changes integrated and deploy to soak test environment for extended validation:
- Monitor system behavior over multiple days to verify stability
- Validate detection across actual operational time scales (2 hours, 24 hours, 7 days)
- Confirm no resource leaks or performance degradation from monitoring itself
- Verify detection accuracy and false positive rate in real-world conditions

## Future Enhancements

- Add CLI to enable/disable monitoring (default: **DISABLED** for community release)
- Store detection events and historical data in Redis STATE_DB for telemetry
- Integrate with SONiC system health framework for early warnings
- Use cgroups to limit monit resource usage and prevent memory pressure from monitoring
- Support Grafana dashboards and log post-processing for visualization