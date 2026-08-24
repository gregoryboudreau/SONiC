# Show Platform NPU User Mode CLI

## Table of Contents

1. [Revision](#1-revision)
2. [Scope](#2-scope)
3. [Definitions/Abbreviations](#3-definitionsabbreviations)
4. [Overview](#4-overview)
5. [Requirements](#5-requirements)
6. [Architecture Design](#6-architecture-design)
7. [High-Level Design](#7-high-level-design)
7.3. [Recommended Design: Two Sockets](#73-recommended-design-two-sockets)
7.8. [Alternative Design: Single Socket](#78-alternative-design-single-socket)
7.9. [Design Comparison](#79-design-comparison)
8. [SAI API](#8-sai-api)
9. [Configuration and Management](#9-configuration-and-management)
10. [Warmboot and Fastboot Design Impact](#10-warmboot-and-fastboot-design-impact)
11. [Memory Consumption](#11-memory-consumption)
12. [Restrictions/Limitations](#12-restrictionslimitations)
13. [Testing Requirements/Design](#13-testing-requirementsdesign)
14. [Open/Action Items](#14-openaction-items)

---

## 1. Revision

| Version | Date       | Author    | Description                          |
|---------|------------|-----------|--------------------------------------|
| 0.1     | 2025-03-01 | Anand Mehra (anamehra)  | Initial HLD for user mode CLI |
| 0.2     | 2026-03-22 | Anand Mehra (anamehra)  | Updated CLI list and RP to LC execution behavior |

---

## 2. Scope

This document describes the high-level design for enabling a subset of **show platform npu** CLIs to run in **user mode** (without `sudo`) on Cisco 8000 SONiC platforms. The change is limited to:

- Infrastructure: **two Unix domain sockets** — a root socket (privileged only) and a user socket (world-connectable); the dshell client enforces an allow-list when the request is received on the user socket. User-mode CLIs connect to the user socket via `use_user_socket=True`.
- A hand-picked set of `show platform npu` subcommands that are safe for production and have low runtime.

**Out of scope:**

- Enabling all `show platform npu` commands without sudo.
- Changes to SAI, ConfigDB, or SONiC core CLI framework.
- gRPC-based reimplementation of commands (future work).

---

## 3. Definitions/Abbreviations

| Term            | Definition |
|-----------------|------------|
| **CX**          | Customer Experience team; often needs to run diagnostics on production without customer intervention. |
| **dshell**      | Debug shell client running inside the syncd container; serves CLI requests over Unix sockets. |
| **Root socket** | Unix socket with default permissions; only root can connect. Full command set. |
| **User socket** | Unix socket with `chmod 0o666`; any user can connect. Only allow-listed commands are accepted. |
| **User mode CLI** | Allow-listed subcommands that connect to the user socket (e.g. via `use_user_socket=True`) and run without sudo. |
| **Peer credentials** | (Single-socket approach only.) UID/GID from `SO_PEERCRED`; used to distinguish privileged vs. normal user on a single socket. |

---

## 4. Overview

**Customer/CX requirement:** Run selected **show platform npu** commands in production **without sudo**. Customers want CX to execute these commands without customer intervention (e.g., no need to provide root or run `sudo`). Only some CLIs shall be available at user level, not all. Selective access via TACACS (e.g. group-level or command-level authorization) is not an option for CX at this time; the solution must enforce the subset in-platform (allow-list in the dshell layer).

**Approach:** Commands were analyzed for:

- Usage by CX and operational value.
- Safety in production (low runtime, no long SAI/API stalls).
- Reliance on the existing Python/dshell path vs. gRPC.

Only commands that are highly used, safe, and have acceptable runtime are enabled for user mode. Commands that still use the Python infrastructure and can cause SAI call delays remain restricted to privileged users. This HLD defines the **infrastructure** for user-mode execution using the **recommended approach: two sockets** — a root socket (full command set, root only) and a user socket (allow-list only, chmod 0o666). Allow-listed CLIs connect to the user socket; all others use the root socket and require sudo. Adding further commands to the allow-list requires evaluation of runtime and data size; gRPC-based implementations do not by themselves guarantee safe execution if they pull large data from the SDK.

On T2 chassis, where CLI execution on RP for LC npu commands is supported (202511 release onwards), the non-sudo access CLIs will use the RP-LC redis channel to execute the CLIs on LC with user level access. All other sudo access CLIs should use rexec.

For any new CLI development, it is mandatory to set use_user_socket to True if the CLI is safe for non-sudo access, or False otherwise to keep sudo access only.

---

## 5. Requirements

| ID   | Requirement | Notes |
|------|-------------|--------|
| REQ-1 | CX and operators shall be able to run a defined subset of `show platform npu` commands without `sudo`. | Customer ask. |
| REQ-2 | Only commands on an explicit allow-list shall be executable when the connecting client is a normal user (non-root, not in privileged group). | Security and stability. |
| REQ-3 | When the connecting client is root or in the privileged group, the full command set remains available (current behavior). | No change for sudo/privileged users. |
| REQ-4 | Allow-listed commands shall be chosen for low runtime and production safety. | Avoid SAI/API stalls and large data pulls. |
| REQ-5 | Adding new user-mode commands shall require code change (allow-list only) and evaluation. | No automatic promotion of commands. |
| REQ-6 | Only a **subset** of `show platform npu` CLIs shall be available at user level (without sudo); not all commands. Selective per-CLI access via TACACS (e.g. group-level access lists or command authorization) is **not** an option for CX at this time; therefore the allow-list and enforcement shall be implemented in-platform (dshell/socket layer), not via AAA configuration. | CX need in-platform, selective user-mode access without relying on TACACS to define which CLIs a user can run. |

**Exemptions / non-goals:**

- Not all `show platform npu` subcommands are enabled for normal users; many remain restricted to privileged (root/sudo group) users.
- **TACACS/AAA:** Controlling which specific `show platform npu` commands a user can run via TACACS command authorization or group access lists is out of scope for this design; CX cannot rely on that at this time. Enforcement is in-platform via the allow-list.
- gRPC-based implementations are out of scope for this HLD; when added later, commands that pull large data from the SDK still require caution.

---

## 6. Architecture Design

The change fits into the existing architecture as follows (**recommended approach: two sockets**, used for implementation):

- **CLI (host):** `show platform npu` is implemented in `sonic-platform-modules-cisco` (e.g. `platform_cli.py`). It talks to the dshell client via `dshell_proxy.run_command_dshell()`. For allow-listed subcommands the CLI passes `use_user_socket=True` and connects to the **user socket**; for all others it uses the default and connects to the **root socket** (requires sudo).
- **syncd container:** The dshell client (`dshell_client.py`) listens on **two** Unix sockets. The root socket has default permissions (root only); the user socket is `chmod 0o666` so any user can connect. When a request is received on the user socket, the server enforces the allow-list; when received on the root socket, the full command set is allowed.
- **No change** to Redis DBs, SAI, or other SONiC core components.

```text
+------------------+     Root socket (root only)    +---------------------------+
|  show platform   |  ------------------------>  |  dshell_client (syncd)    |
|  npu ... (sudo)  |     sonic_cli_{asic}.sock    |  - sock_welcome: full set |
+------------------+                              |  - sock_welcome_user:     |
+------------------+     User socket (0o666)       |    allow-list only        |
|  show platform   |  ------------------------>  |                          |
|  npu ... (no     |     sonic_cli_user_{asic}.   +---------------------------+
|  sudo)           |     sock                     |
+------------------+
```

---

## 7. High-Level Design

### 7.1 Type of Change

- **Platform-specific enhancement** within the Cisco 8000 platform code (sonic-platform-modules-cisco, docker-syncd-cisco).
- No change to built-in SONiC features or Application Extension infrastructure.

### 7.2 Repositories and Modules

| Repository / path | Changes |
|-------------------|---------|
| `dshell_proxy.py` | `run_command_dshell()` takes `use_user_socket`. If True, connect to `SONIC_CLI_URI_USER`; if False, connect to `SONIC_CLI_URI`. |
| `platform_cli.py` | For each allow-listed `show platform npu` subcommand, call `run_command_dshell(..., use_user_socket=True)`. For all others, use default `use_user_socket=False`. |
| `dshell_client.py` | Bind and listen on **both** sockets. Root socket: `sonic_cli_{asic}.sock` (default perms). User socket: `sonic_cli_user_{asic}.sock`, then `chmod(socket_file_user, 0o666)`. When request is received on user socket, pass `from_user_socket=True` to `handle_user_cli()`; in `handle_user_cli()`, when `from_user_socket` and `cmd` not in allow-list, reject. |

### 7.3 Recommended Design: Two Sockets

**This is the recommended and implemented approach.** User mode is implemented using **two separate Unix sockets**.

**Socket paths:**

- **Root socket:** `/var/cache/cisco/sonic_cli_{asic}.sock` — default permissions so only root (or the process user) can connect. Full command set.
- **User socket:** `/var/cache/cisco/sonic_cli_user_{asic}.sock` — after `bind()`, the dshell client calls `os.chmod(socket_file_user, 0o666)` so any user can connect. Only allow-listed commands are accepted when the request is received on this socket.

**Server (dshell_client.py):**

- Bind and listen on **both** sockets (e.g. `sock_welcome` and `sock_welcome_user`).
- When a connection is accepted, the server knows which socket it came from. If it came from the user socket, set `from_user_socket=True` and pass that to `handle_user_cli()`. In `handle_user_cli()`, when `from_user_socket` is True and `cmd` is not in `USER_SOCKET_ALLOWED_COMMANDS`, reject the command.

**Client (dshell_proxy.py, platform_cli.py):**

- `run_command_dshell()` takes a parameter `use_user_socket`. If True, connect to `SONIC_CLI_URI_USER`; if False, connect to `SONIC_CLI_URI`.
- For each allow-listed `show platform npu` subcommand, the CLI calls `run_command_dshell(..., use_user_socket=True)`. For all other subcommands, it uses the default `use_user_socket=False`, so they connect to the root socket and require sudo.

**Pros:** Root socket stays restricted by file permissions (defense in depth); no dependency on `SO_PEERCRED`; privilege is implied by which socket the client can connect to. **Cons:** Two sockets to create, bind, listen, and clean up; CLI must be updated per-command to pass `use_user_socket=True` when adding a new user-mode command. **Recommendation:** Use this design for implementation; it is the preferred approach in this HLD.

### 7.4 Initial User-Mode Commands (Allow-List)

The following dshell command names are on the allow-list. When the connecting client is a normal user (non-privileged), only these commands are executed; all have corresponding `show platform npu` CLIs:

| Dshell command | `show platform npu …` |
|----------------|------------------------|
| `show_port_entries` | `port entries` |
| `show_port_counters` | `port counters` |
| `show_next_hop_entries` | `next-hop entries` |
| `show_next_hop_usage` | `next-hop usage` |
| `show_router_entries` | `router entries` |
| `show_router_ports` | `router ports` |
| `show_router_details` | `router details` |
| `show_route_table` | `router route-table` |
| `show_prefix_map` | `router prefix-map` |
| `show_router_port_counters` | `router port-counters` |
| `show_ecmp` | `ecmp` |
| `show_packet_path` / `show_packet_path_ipv6` | `packet-path` |
| `show_trap` | `trap` |
| `event_trap` | `event-trap` |
| `show_and_dump_capture` | `packet-debug` → `capture` only |
| `show_globals` | `global` |
| `get_resource_usage` | `resource` |
| `show_lpts` | `lpts` |
| `show_npu_l3_interface` | `l3-interface` |
| `show_temperatures` | `temperatures` |
| `show_bfd_summary` | `bfd summary` |
| `show_bfd_counter_stats` | `bfd counter` |
| `show_histograms` | `histogram` |
| `get_counters` | `counters` |
| `oq_debug` | `oq-debug` |
| `show_mac_state` | `mac-state` |
| `show_npu_rx` | `rx …` |
| `show_npu_tx` | `tx …` |
| `show_npu_voq` | `voq …` |


The following are **not** on the allow-list and require the client to be privileged (root or in the privileged group): port entries, router ports, event-trap, counters, oq-debug; as well as asic-errors, router details/entries, switch entries/ports, lag entries, techsupport, and other privileged-only subcommands.

### 7.5 Sequence (Normal User, Two-Socket Approach)

1. User runs e.g. `show platform npu trap` **without** sudo (as a normal user).
2. CLI resolves namespace/asic and calls `run_command_dshell("show_trap", asic=asic_id, use_user_socket=True)`.
3. `dshell_proxy` connects to `sonic_cli_user_{asic}.sock` and sends the JSON request.
4. dshell client accepts on the user socket, sets `from_user_socket=True`, and calls `handle_user_cli(..., from_user_socket=True)`.
5. `handle_user_cli` checks `cmd in USER_SOCKET_ALLOWED_COMMANDS`; if not, sends error and returns.
6. If allowed, command is executed (gRPC or local list) and response is sent back.
7. CLI prints the response.

### 7.6 Scalability and Performance

- Two sockets; one set lookup per request when the connection is on the user socket.
- Allow-listed commands were chosen for low runtime to avoid blocking syncd or causing SAI delays. Adding new commands must consider execution time and data volume.

### 7.7 Platform Specificity

This design is specific to the Cisco 8000 platform and its dshell/gRPC-based NPU CLI. Other platforms do not use this socket or allow-list.

### 7.8 Alternative Design: Single Socket (Second Approach)

An alternative is to use a **single socket** with world-connectable permissions and determine privilege via **peer credentials** (`SO_PEERCRED` on Linux).

**Socket path:** `/var/cache/cisco/sonic_cli_{asic}.sock` — after `bind()`, `chmod(socket_file, 0o666)` so both root and non-root can connect. On each `accept()`, the server gets peer UID/GID via `SO_PEERCRED`; if `uid == 0` or gid in a privileged group (e.g. sudo), full command set; otherwise only the allow-list.

**CLI side:** No `use_user_socket`; every command connects to the same socket. The server decides access based on peer credentials.

**Pros:** One socket; no per-command `use_user_socket=True` in the CLI; flexible privilege (root or group). **Cons:** Relies on `SO_PEERCRED`; no file-permission barrier (defense in depth); a server bug could allow bypass.

### 7.9 Design Comparison

**Recommended and implemented: two-socket approach.** The single-socket approach is documented as an alternative only.

| Aspect | First approach: Two sockets (implementation) | Second approach: Single socket |
|--------|------------------------------------------|----------------------------|
| **Sockets** | Two: `sonic_cli_{asic}.sock` (root) and `sonic_cli_user_{asic}.sock` (world) | One: `sonic_cli_{asic}.sock`, chmod 0o666 |
| **How privilege is known** | Server infers from which socket the connection arrived on (root socket = privileged) | Server gets peer UID/GID via `SO_PEERCRED`; root or privileged group → full access |
| **CLI changes** | Allow-listed commands pass `use_user_socket=True`; others use root socket | Same call for all commands; no per-command socket choice |
| **Adding a user-mode command** | Add to allow-list **and** add `use_user_socket=True` at CLI call site | Add to allow-list only |
| **Platform / OS** | No OS-specific socket options; works on any Unix with two listeners | Requires Linux `SO_PEERCRED`; privileged group (e.g. sudo) configurable |
| **Privileged group** | Root or members of a designated group (e.g. sudo) get full set | Only root can connect to root socket; “privileged” = root by connectivity |
| **Security** | Root socket restricted by file permissions; user socket exposes only allow-list (defense in depth) | Single world-connectable socket; enforcement is allow-list + credential check |


**Why the two-socket approach is recommended:** Root socket can be kept root-only at the file level, so unprivileged users cannot connect to the privileged listener even if the server has a bug — defense in depth. No dependency on `SO_PEERCRED`. The single-socket approach is a valid alternative if fewer code paths and no per-command `use_user_socket` are preferred; it trades defense in depth for simpler CLI and one socket.

---

## 8. SAI API

No SAI API changes are introduced by this HLD. The same underlying SAI/gRPC or local logic used for the existing `show platform npu` commands is used; only the transport (root vs. user socket) and allow-list enforcement are new.

---

## 9. Configuration and Management

### 9.1 CLI

- **No new CLI commands.** Existing `show platform npu` subcommands are unchanged in syntax and behavior.
- **Behavioral change:** The listed subcommands can be run **without** `sudo`; the server accepts them for normal users because they are on the allow-list. All other subcommands are accepted only when the client is privileged (root or in the configured group).
- **Backward compatibility:** Users who run with `sudo` (or are in the privileged group) get the same behavior as before (full command set). Downward compatibility is preserved.

### 9.2 Config DB

No Config DB or state DB changes. No new configuration keys.

---

## 10. Warmboot and Fastboot Design Impact

- No impact on warmboot or fastboot. Socket creation and listen loop are unchanged; only the socket file is chmod'd for world access and peer credentials are read on accept. No stalls or extra I/O in the boot-critical path. No change when the feature is unused (same process, same startup).

---

## 11. Memory Consumption

- Negligible: no extra socket; a fixed frozenset (allow-list) and a small privileged-group list. Peer credential lookup is per-connection and does not grow. No growing memory when the feature is unused.

---

## 12. Restrictions/Limitations

- Only the commands explicitly listed in `USER_SOCKET_ALLOWED_COMMANDS` are executable by normal (non-privileged) users. All other `show platform npu` subcommands require the client to be root or in the privileged group.
- Adding a new user-mode command requires: (1) adding the dshell command name to `USER_SOCKET_ALLOWED_COMMANDS`, (2) ensuring the command has low runtime and is safe for production (e.g. no large SDK data pulls).
- gRPC-based implementation of a command does not by itself make it safe for user mode; commands that pull large data from the SDK need evaluation before being added to the allow-list.
- The single socket is created only when the dshell client (e.g. SDK debug / syncd) is running; if debug shell is not enabled, the same “socket not available” behavior applies for both privileged and normal users.

---

## 13. Testing Requirements/Design

### 13.1 Unit Tests

- dshell client: after `accept()`, peer UID/GID from `SO_PEERCRED` correctly drive `is_privileged` (e.g. uid 0 or gid in privileged group).
- dshell client: when `is_privileged=False` and `cmd` not in `USER_SOCKET_ALLOWED_COMMANDS`, client returns an error and does not execute the command. When `is_privileged=True`, any command is allowed.

### 13.2 System Tests

- Run each allow-listed `show platform npu` subcommand **without** sudo as a non-root user; expect success and sane output.
- Run a non-allow-listed `show platform npu` subcommand without sudo; expect failure (e.g. “command not allowed for non-privileged user”).
- Run the same non-allow-listed command with sudo (or as a user in the privileged group); expect success.
- Multi-ASIC: verify user-mode commands work for each namespace/asic.

---

## 14. Open/Action Items

- **Documentation:** Update Command-Reference or platform-specific docs to state which `show platform npu` subcommands can be run without sudo.
- **Future commands:** Before adding new commands to user mode, evaluate runtime and data size; document decision.
- **gRPC:** If more commands are moved to gRPC, apply the same allow-list and safety criteria before enabling them for normal users.
- **Privileged group:** Document the chosen privileged group(s) (e.g. `sudo`) in platform docs and ensure supplementary groups are checked if only primary GID is available from `SO_PEERCRED`.
