
# RP LC Command execution HLD

#### Rev 0.1

# Table of Contents

- [List of Tables](#list-of-tables)
- [Revision](#revision)
- [Definition/Abbreviation](#definitionabbreviation)
- [About This Manual](#about-this-manual)
- [1 Introduction and Scope](#1-introduction-and-scope)
  - [1.1 Limitation of existing design](#11-limitations-of-existing-design)
  - [1.2 Benefits of this feature](#12-benefits-of-this-feature)
- [2 Feature Requirements](#2-feature-requirements)
  - [2.1 Functional Requirements](#21-functional-requirements)
  - [2.2 Configuration and Management Requirements](#22-configuration-and-management-requirements)
  - [2.3 Scalability Requirements](#23-scalability-requirements)
  - [2.4 Warm Boot Requirements](#24-warm-boot-requirements)
- [3 Feature Description](#3-feature-description)
- [4 Feature Design](#4-feature-design)
  - [4.1 Design Overview](#41-design-overview)
  - [4.2 Decoding of Messages from RP to LC](#42-decoding-of-messages-from-rp-to-lc)
  - [4.3 Redis-DB usage](#43-redis-db-usage)
  - [4.4 Error handling of Redis connection failures](#44-error-handling-of-redis-connection-failures)
  - [4.5 Parallel execution of Commands](#45-parallel-execution-of-commands)
  - [4.6 Fragmentation of messages from LC to RP](#46-fragmentation-of-messages-from-lc-to-rp)
  - [4.7 List of CLI commands allowed for location based execution](#47-list-of-cli-commands-allowed-for-location-based-execution)
  - [4.8 Syslogs](#48-syslogs)
- [5 CLI](#5-cli)
  - [5.1 Output Format](#51-cli-output-format)
- [6 Serviceability and Debug](#6-serviceability-and-debug)
- [7 Unit Test Cases ](#7-unit-test-cases)
- [8 References ](#8-references)

# List of Figures

- [Figure 1: Specific LC command execution diagram](#figure-1-specific-lc-command-execution-diagram)

# List of Tables

- [Table 1: Abbreviations](#table-1-abbreviations)

# Revision
| Rev | Date | Author | Change Description |  
|:---:|:----------:|:-------------:|:---------------:|  
| 0.1 | 11 June 2025 | Madhan Babu, Anand Mehra | Initial version |  


# Definition/Abbreviation

### Table 1: Abbreviations

| **Term**  | **Meaning** |  
| ----------- | ----------- |  
| RP   | Route Processor or Supervisor Card |  
| LC   | Linecard    |  


# About this Manual

This document provides general information about Centralized command execution on Supervisor Card feature. This feature allows the user to execute commands from Supervisor Card and fetch the output from Supervisor Card as well as all Linecards or issue command on Supervisor Card to fetch output from a specific Linecard.

# 1 Introduction and Scope

This document describes the functionality and High level design of node specific command execution from Supervisor Card (RP/Route Processor) in SONiC chassis.

The system supports command execution across different module types:
- **LINE-CARDx modules**: Line card modules (LINE-CARD0, LINE-CARD1, etc.)

For multi-ASIC systems, the framework provides namespace options (asic0, asic1, asic2, etc.) to target specific ASICs within each module.

The scope of commands is limited to "show platform npu ..." and "config platform cisco ..." commands. The full list of commands as of now is given as:
```
cisco@sfd-01:~$ sudo show platform npu ?
Usage: show platform npu [OPTIONS] COMMAND [ARGS]...

  Show NPU

Options:
  -h, -?, --help  Show this message and exit.

Commands:
  acl   Show NPU ACL
  arc-debug-counters  Show NPU ARC debug counters
  asic-errors  Show NPU asic-errors
  bfd   Show platform npu bfd
  bp-interface-map  Show BP interface map.
  cem-db
  counters  Show NPU counters
  ecmp  Show NPU Equal Cost Multi Paths
  event-trap   Show NPU event traps
  global  Show NPU global debugging information
  histogram  Show NPU histogram
  l3-interface Show NPU l3 interface
  lag   Show NPU LAGs
  lpm-db
  lpts  Show NPU Local Packet Transport Service
  mac-state  Show NPU MAC state
  next-hop  Show NPU next-hops
  oq-debug  Show NPU output-queue statistics
  packet-debug show platform npu packet-debug
  port  Show NPU ports
  rate-check   Show NPU rate-check
  resource  Show NPU resource usage
  router  Show NPU routers
  rx Show NPU rx
  script  Run a NPU script
  sdk-debug  Show NPU debug shell information
  switch  Show NPU switches
  temperatures Show NPU ASIC temperatures
  trap  Show NPU traps
  trap-list  Show trap list with indices
  tx Show NPU tx
  udf-hash  Show NPU UDF-HASH setting
  voq   Show NPU voq
  vxlan   Show platform npu vxlan
cisco@sfd-01:~$
```

```
root@yy39-lc5:/home/admin# config platform cisco --help
Usage: config platform cisco [OPTIONS] COMMAND [ARGS]...

  Cisco platform configuration tasks

Options:
  -?, -h, --help  Show this message and exit.

Commands:
  bfd                 bfd counter Enable/Disable - Command Line to...
  disable-l2-traps    Disable/restore the L2 traps config
  fabric              Config for managing fabric interface
  fabric-port-speed   Command to change fabric port speed
  histogram           Configure NPU histograms
  interface           Utility for managing Cisco interface
  packet-debug        packet-debug drops Enable/Disable - Command Line to...
  role                Command to change role
  sdk-debug           sdk-debug Enable/Disable - Command Line to...
  tgen                Command Line to configure tgen
  trap-configuration  Configure trap for a Leaba event
  udf-hash            udf-hash Set/Clear - Command Line to set/clear udf...
  voq-watchdog        Enable/disable voq watchdog
  vxlan               vxlan counter Enable/Disable - Command Line to...
```

At present, "show platform npu ..." and "config platform cisco ..." commands can be independently executed on a modular chassis on a RP or a LC. When the command is executed on a RP, the command output corresponds to that of RP. Similarly, when the command is executed on a LC, the command output corresponds to that LC. 

**Current Design Limitations:**
- In current multi-ASIC devices, the `-n` option is mandatory and commands do not work without it
- There is no mechanism to execute the command on RP and fetch the output from RP as well as all LCs
- Cannot issue command on RP to fetch output from a specific LC
- Cannot fetch data from all namespaces in a single command

**This New Design Addresses:**
- For "show platform npu" commands: Makes `-n` option optional - when not provided, fetches data from all available namespaces
- For "config platform cisco" commands: `-n` option remains mandatory for safety; "all" must be explicitly provided to run commands for all namespaces
- Enables centralized command execution from Supervisor Card to all modules
- Provides location-based command execution capabilities

But, with SONIC architecture, where the RP and LC can communicate to each other over Redis-DB specifically over CHASSIS-STATE-DB, NotificationProducer / NotificationConsumer through Event notifications, PubSub through Keyspace notifications, Channel Notifications etc, this mechanism of issuing a command on RP and sending it to LCs and executing on LCs and sending the command response back to RP is feasible.

For the design choices for communication between RP and LC, the REDIS based communication over CHASSIS-STATE-DB was preferred. The choices were Pub-Sub keyspace notifications, Pub-Sub event notifications and Notification Producer / Notification Consumer channels. Pub-Sub keyspace notification involves updates to Redis Tables. Updating the table contents was not considered as a good design choice if we need to send messages from RP to LC to run the command on LC, these updates to tables - although they maintain a state, but it was not preferred as these changes are permanent. Pub-Sub event notifications are supported only by Redis APIs and not by SONIC-SWSS APIs. But, the Notification Producer / Notification Consumer channels are one shot event messages and don't update the REDIS tables in any way. So, Notification Producer / Notification Consumer channels mode of communication between RP and LC design choice was chosen.

A new python based LC-RP command client (lc-rp-command-client) service which is started as a platform service on the host on LCs during bootup. The sonic-utilities CLI click parser is where the RP generates the commands and sends it to all LCs or a specific LC. This framework on RP and LC gives provision for them to communicate with each other for command execution.

## 1.1 Limitations of existing design:
With existing design, there is no provision to execute command from RP that would fetch the command output of that of LC. If a user wants to get the command output of same command on all nodes including RP and LCs, the user has to execute the same command once on RP and once each on every LC. And these outputs will be on different terminals as they are executed on different terminals, which does not allow the user to view the command outputs from a single terminal.

## 1.2 Benefits of this feature:
 - Works based on Event notifications model where the command output responses are returned back to RP from LCs immediately.
 - "show platform npu ..." and "config platform cisco ..." commands can be executed from RP to get command output from RP only, from specific LC only or from all nodes including RP and all LCs.
 - For "show platform npu" commands: Makes `-n` (namespace) parameter optional - when not specified, automatically fetches data from all available namespaces, eliminating the current limitation where `-n` is mandatory.
 - For "config platform cisco" commands: Maintains `-n` parameter as mandatory for safety, with explicit "all" option to run commands across all namespaces.
 - The same framework can be extended to other commands so that RP can be the only point of command execution for the entire chassis.
 - LC-RP command client is a platform service which is started on bootup on the host on LCs. It can be stopped or restarted anytime.


# 2 Feature Requirements

## 2.1 Functional Requirements

Following requirements are addressed by the design presented in this document:

1. Command executed on Supervisor Card should provide options like location and namespace.
  a. If location is not provided, the command is executed locally on the current module (SUPERVISOR0 if run on Supervisor Card, or specific LINE-CARDx if run on that linecard) and output is collected.
  b. If the location is "all", the command is executed on Supervisor Card(s) as well as all the line card modules (LINE-CARDx) and the respective command outputs are collected.
  c. If the location is a specific module (e.g., LINE-CARD0, LINE-CARD1), the command is executed on that specific module and the command output is collected.
2. Namespace (asic-id) behavior differs by command type:
   a. For "show platform npu" commands: If namespace (e.g., asic0, asic1, asic2) is mentioned, the command is executed for the specific namespace. If asic-id is not mentioned, the command output shall contain outputs for all available namespaces of the Supervisor or linecard module.
   b. For "config platform cisco" commands: Namespace parameter remains mandatory for safety. To execute across all namespaces, "all" must be explicitly specified as the namespace value.
   This supports multi-ASIC scenarios where each module may have multiple ASICs. 
3. All "show platform npu ..." and "config platform cisco ..." commands must implement a standardized action component within the click parser to enable integration with the RP-LC command execution framework.
4. Parallel execution of command from multiple RP user shells/terminals should be allowed.
5. Client Service on LCs (LC-RP command client) should be started as a service during bootup.
6. Client Service should be stoppable and restartable.


## 2.2 Configuration and Management Requirements

This feature will use the existing CLIs as defined in section 1 and extend them to location wise command execution on RP and LCs.


## 2.3 Scalability Requirements

NA

## 2.4 Warm Boot Requirements

This feature does not have any warm boot impact, the LC-RP command client service is started even in the case of Warm boot.


# 3 Feature Description

This feature provides framework to invoke commands on RP such that based on the location provided in the command, the command actually gets executed on specific LC or all LCs and command output is collected on RP.

LC-RP command client is started as a platform service during bootup. And the "show platform npu ..." commands can be executed once the LC-RP command client is started on all LCs which is during bootup. LC-RP command client is restartable at any point of time.

# 4 Feature Design
## 4.1 Design Overview

The feature is divided into 2 tasks. The first task and always running task is LC-RP command client which is started as a platform Service during bootup. The other task lies in the action part of CLI click parser of the "show platform npu ..." commands of RP. This task comes into action when these commands are executed in RP.

The command executed on Supervisor Card does not specify a location:
1. The command is executed locally on the current module (SUPERVISOR0 if run on Supervisor Card, or specific LINE-CARDx if run on that linecard) and the output is displayed.
2. If no namespace is specified, data is captured for all available namespaces on that local module.


The command executed on Supervisor Card specifies a particular module (LINE-CARDx or SUPERVISORx) as the location:
1. Action part of CLI click parser of the commands has a generic routine which generates a random number called session-id. This session-id is very important in parallel execution of "show platform npu ..." commands from different RP terminals.
2. Before sending the command from RP to the target module, we check whether the module service is running on the specified module. To know which modules in the SONiC chassis are running Platform module service, the Platform module service itself calls a periodic heartbeat timer and on every heartbeat timeout callback, it updates CHASSIS_STATE_DB LC_SERVICE_TABLE with its own entry (the nodename as Key) and the current timestamp. The RP will check the CHASSIS_STATE_DB LC_SERVICE_TABLE to know what modules keep updating this table and if this particular module to which the command needs to be sent is active, it sends the command to the module, otherwise, the output is displayed as "Module is not active!!!".
3. Now, the actual command will be sent to the target module for execution. The message from RP to the module is sent over a Notification Producer on an Event called RP_LC_{specific-module}_NOTIF_CHANNEL (for example, RP_LC_LINE-CARD0_NOTIF_CHANNEL if location is LINE-CARD0).
4. The module will receive the message over a Notification Consumer on an Event called RP_LC_{specific-module}_NOTIF_CHANNEL (for example, RP_LC_LINE-CARD0_NOTIF_CHANNEL if location is LINE-CARD0).
5. The module will execute the command and send the response back to RP over a Notification Consumer on an Event called LC_RP_{session_id}_NOTIF_CHANNEL (for example, LC_RP_0.123456789_NOTIF_CHANNEL).

```
MESSAGE SCHEMA BETWEEN RP (Route Processor) AND MODULES (Line Cards):

1. INCOMING MESSAGE FROM RP TO MODULE:
   ------------------------------------
   Channel: "RP_LC_{nodename}_NOTIF_CHANNEL" (e.g., "RP_LC_LINE-CARD0_NOTIF_CHANNEL")

   Message Fields:
   {
       'message_type': 'shell_command',           # Type of message (currently only 'shell_command' supported)
       'sender': 'RP',                           # Sender identifier (always 'RP' for messages from Route Processor)
       'session_id': '<unique_session_id>',      # Unique session identifier for request-response correlation
       'rp_to_lc_message': '<cli_command>'       # The actual CLI command to execute on LC
   }

   Example:
   {
       'message_type': 'shell_command',
       'sender': 'RP',
       'session_id': '0.123456789',
       'rp_to_lc_message': 'show version'
   }

2. OUTGOING RESPONSE FROM LC TO RP:
   ---------------------------------
   Channel: "LC_RP_{session_id}_NOTIF_CHANNEL" (e.g., "LC_RP_0.123456789_NOTIF_CHANNEL")

   Response Fields:
   {
       'sender': '<nodename>',                   # LC node identifier (e.g., 'LINE-CARD0', 'LINE-CARD1')
       'rp_to_lc_message': '<original_command>', # Echo of the original command received from RP
       'cmd_output': '<command_output>',         # Command execution result or error message
       'session_id': '<session_id>'             # Same session ID from the original request
   }

   Example Success Response:
   {
       'sender': 'LINE-CARD0',
       'rp_to_lc_message': 'show version',
       'cmd_output': 'SONiC Software Version: SONiC.4.0.0...',
       'session_id': '0.123456789'
   }

   Example Error Response:
   {
       'sender': 'LINE-CARD0',
       'rp_to_lc_message': 'invalid_command',
       'cmd_output': 'command execution failed: [Errno 2] No such file or directory: invalid_command',
       'session_id': '0.123456789'
   }

  "sender" : "RP"
  "rp_to_lc_message" : command (without location)
  "session_id" : session-id
```  
4. Once, the CLI click parser action part generic subroutine sent the event notification payload to LC, it continues to listen on a Notification Consumer with Notification channel LC_RP_{session_id}_NOTIF_CHANNEL. It keeps listening on this notification channel for a specific number of timeouts, within which it is likely to get output from LC or LCs.
5. The LC-RP command client which is listening on RP_LC_LINE-CARD0_NOTIF_CHANNEL or RP_LC_LINE-CARD1_NOTIF_CHANNEL or RP_LC_LINE-CARD2_NOTIF_CHANNEL etc on respective linecards, receives the notification event with the notification payload (the Field Value pairs containing command, session-id etc).
6. The LC-RP command client on receiving the event, spawns a thread to execute the command and collects the output and creates a response event notification payload containing key value pairs as described in the schema above.  And, sends the notification event on "LC_RP_{session_id}_NOTIF_CHANNEL using its NotificationProducer.  
7. The generic routine in action part of CLI click parser is still listening on LC_RP_{session_id}_NOTIF_CHANNEL channel. This event notification sent from LC is received by the Notification consumer on RP.
8. The notification consumer on RP on receiving the command output message verifies the command response is received from desired Linecard and print the command output on the terminal.
9. In the event that the Notification consumer which is listening for event notifications from LC as part of the CLI click parser in RP times out after specific number of iterations, will return no output to the terminal.
10. This mechanism of RP-LC Command execution for location as specific LC can be shown in diagram
![Specific LC command execution diagram](./diagrams/specific_lc_command_execution.png "Figure 1: Specific LC command execution diagram")
###### Figure 1: Specific LC command execution diagram

The command executed on RP specifies "all" as the location:
The execution of location "all" is similar to that of specific line card location execution. But, in "all" location execution, the RP sends to all Linecards Notification channels and wait for response. Also, the RP executes the command locally and provide it as output from Supervisor. The command output received from different LCs are display as output from these respective LCs in seperate sections.

**Config Platform Cisco Commands Execution:**
The "config platform cisco ..." commands follow the exact same execution process as defined above for "show platform npu ..." commands. The key differences are:
- Namespace parameter (`-n`) remains mandatory for config commands for safety reasons
- To execute config commands across all namespaces, "all" must be explicitly specified as the namespace value (e.g., `-n all`)
- The same Redis-based communication, session management, and location-based execution mechanisms apply
- The same message schema and notification channels are used for command execution and response handling

## 4.2 Decoding of Messages from RP to LC

The LC-RP command client service is started as a platform service on the host on all the Linecards during bootup. The design of the LC service should be kept generic so that it receives message from RP and invokes it on LC and return back the command output response back to RP. But, to accomodate different types of messages which are required to be executed on LC based on show commands and config commands (the message types are shell commands, we can also make it work for dshell commands (for show platform npu commands). The LC-RP command client service will first decode the message type and based on the message type, it will decode rest of the message attributes. This will give the opportunity to come up with new message types and extend the LC service based on new use cases that may require the RP-to-LC communication. For now the supported message types on LC service is only "shell command".

## 4.3 Redis-DB usage:

Redis-DB is used in both CLI click parser and platform LC service. It is also used in communication between RP and LC.

Redis-DB is used in CLI click parser for the following:
1. To find the node-type and node-name as well as to get the chassis-lc-list, CHASSIS_MODULE_TABLE of STATE_DB is accessed.
2. To autofill ASIC-list (namespace-list), CHASSIS_ASIC_TABLE of CHASSIS_STATE_DB is accessed.

Redis-DB is used in LC client service for the following:
1. To find the node-type and node-name, CHASSIS_MODULE_TABLE of STATE_DB is accessed.
2. To update LC service liveliness on LC, we update LC_SERVICE_TABLE of CHASSIS_STATE_DB.

Redis-DB is used in communication between RP and LC in the following ways:
1. NotificationProducer / NotificationConsumer constructs are created over CHASSIS_STATE_DB. Becuase, it is communication between RP and LC, NotificationProducer / NotificationConsumer are created over CHASSIS_STATE_DB.


## 4.3 Error handling of Redis connection failures:

In the LC-RP command client service, the threads that wait on select() call for the NotificationConsumer of either RP_LC_NOTIF_CHANNEL or linecard specific consumer channel (say, RP_LC_LINE-CARD0_NOTIF_CHANNEL) could probably result in error conditions due to REDIS db connection failures. These kind of error conditions will be gracefully handled and the Notification Consumer will reconnect to REDIS db and be available again to wait on the select() call. 

## 4.5 Parallel execution of Commands:

The CLI command execution of "show platform npu ..." commands may possibly get invoked from different RP (SSH) terminals of the same SONiC chassis at any point of time. The design takes care of this by creating a session-id for every command that is invoked (the session-id is created as part of click parser action routine of the command) and the Notification consumer channel on the RP has the session-id part of the Notification consumer channel name. By this way, we ensure that the parallel execution of commands from different RP terminals of the SONiC chassis at any point of time.

## 4.6 Fragmentation of messages from LC to RP:

The command output of some "show platform npu ..." commands may be very big in size (For example, a SONiC chassis running as a Regional gateway can have a maximum of 151k routes in the Forwarding table, "show platform npu router route-table" inclusive of all the namespaces will be upto 151k route which is equivalent of text output of about 30 MB). But, the recommended size of messages shared across REDIS in SONIC should not exceed 512 KB. If the message size grows more than the recommended limit of 512 KB, there could be possible silent dropping or truncating of the messages. In the case of repeated large size messages across Redis components from LC to RP, it could possible cause crash some of the sonic services like orchagent or syncd. To overcome these issues expected with large size message transfers between LC to RP, we will fragment the messages sent from LC to RP.  In RP, before displaying the command output from LC, these fragemented messages will be reassemebled to form the original show command output from the linecards.

## 4.7 List of CLI commands allowed for location based execution

Since the platform LC service process runs in root mode, any CLI command that is sent from RP to LC will get executed in the platform LC service and send the CLI output back to RP. To avoid unprivileged user access of data at Linecards, which otherwise would be a security risk, in the linecards, we will be maintaining a list of CLI commands that can be invoked using this centralized CLI feature. If the CLI command invoked on RP with a specific LC location option or "all" location option, that command will be sent to LC, but in the platform LC service running in the LC, it will check whether that CLI command is part of the allowed list of CLI commands. If it is not an allowed command, the CLI will return "Not permitted to run this CLI command using centralized CLI feature". If the command is one of the allowed commands in the list, then the platorm LC service will execute the command and send the CLI output back to RP.

## 4.8 Syslogs:

- Syslog to be generated for tracking the progress of LC-RP command client service. The following actions are logged.
- Waiting for REDIS-DB to be up.
- Connected to REDIS-DB.
- When the service is able to collect localhost information from CONFIG-DB DEVICE_METADATA table. Otherwise, the service loops there for the REDIS-DB to be ready.
- When the service starts a thread to monitor RP_LC_NOTIF channel for commands from RP sent as event notifications.
- When the service starts a thread to monitor line card specif channel RP_LC_{specific-linecard}_NOTIF_CHANNEL (example, RP_LC_LINE-CARD0_NOTIF_CHANNEL channel) for line card specific command execution of command sent from RP sent as event notifications.
- When a event notification is received on Notification channel
- When service spawns a thread to execute the command 
- When the thread that runs the command on LC send an event notification (command output response) back to the RP.


## 5 CLI:

There is no new CLI introduced for this feature. We are providing an extension of capability of what "show platform npu ..." commands can do, basically when the command is issued on RP it communicates to LCs and executes the command on LCs and collects the command response back in RP displays to the terminal.

```
cisco@sfd-01:~$ show platform npu counters -h
Usage: show platform npu counters [OPTIONS]

Show NPU counters

Options:
-l TEXT Node
-n [asic0|asic1|asic2|asic3|asic4|asic5|asic6|asic7|asic8|asic9|asic10|asic11|asic12|asic13|asic14|asic15]
Namespace [optional - if not provided, fetches data from all namespaces]
-t TEXT Time out in sec
-c TEXT Clear counters
-h, -?, --help Show this message and exit.
cisco@sfd-01:~$
```

The existing CLI of show platform npu commands has -l option for specifying the location which is a Text option. Currently its syntax says it specifies a Node. But this -l option is ignored in current CLI click parser action.

**Current Behavior (without -l option):**
- When executed on SUPERVISOR (Supervisor Card): Gets data from local SUPERVISOR card
- When executed on LINE-CARDx: Gets data from the specific linecard where it's executed
- When no namespace (-n) option is provided: Captures data for all available namespaces on that module
- Currently only SUPERVISOR0 exists in the system

**Enhanced Behavior (with -l option):**
We will use the command in the following way:

**Examples for LINE-CARD modules:**
```bash
# Get command output from specific line card with specific ASIC
show platform npu counters -n asic0 -l LINE-CARD0

# Get command output from specific line card with all ASICs
show platform npu counters -l LINE-CARD0
```

**Examples for SUPERVISOR modules:**
```bash
# Get command output from SUPERVISOR (currently the only supervisor is SUPERVISOR0) with specific ASIC
show platform npu counters -n asic1

# Get command output from SUPERVISOR with all ASICs
show platform npu counters

# Note: Currently only SUPERVISOR0 exists. Future implementations may support SUPERVISOR1, etc. and in that case, the command will be able to fetch data from SUPERVISOR1 as well with enhance,nets to run the client on SUPERVISOR modules.
```

**Examples for all modules:**
```bash
# Get command output from all modules (line cards and supervisors) with specific ASIC
show platform npu counters -n asic0 -l all

# Get command output from all modules with all ASICs
show platform npu counters -l all
```

The above examples correspond to platform npu counters command, similarly for all show platform npu commands, we can use the -l option for specifying the location.

### 5.1 Output Format:

The "show platform npu ..." commands when executed from RP specifying location as a specific module or "all" is different from the current behavior of commands where they are independently executed on RP terminal or module terminal. For location "all", the command output would list outputs from all available modules:
- "Output for Supervisor" (RP output)
- "Output for LINE-CARD0" and its output
- "Output for LINE-CARD1" and its output
- etc.

The below section captures different show output of specific command "sudo show platform npu counters" based on namespace and location as executed on RP as well as LC.

The following is show output of command of "show platform npu counters -n asic0 -l LINE-CARD0". This is basically comand with specific namespace / asic-id and specific linecard mentioned.

```
cisco@sfd-01:~$ 
cisco@sfd-01:~$ sudo show platform npu counters -n asic0 -l LINE-CARD0

Output for LINE-CARD0:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 7    , bytes = 898    
INFO  Total Forwarding drop counter (DSP==1): packets = 13731    , bytes = 1633824    
INFO  NPU_HOST PktsOut=     13728    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =              95    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             925    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            8050    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =          117444    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

cisco@sfd-01:~$ 
```

The following is show output of command of "show platform npu counters -l LINE-CARD0". This is basically comand with no specific namespace / asic-id mentioned and specific linecard mentioned. This output fetches command output for all namespaces / asic-ids for the specific linecard.


```
cisco@sfd-01:~$ 
cisco@sfd-01:~$ sudo show platform npu counters -l LINE-CARD0

Output for LINE-CARD0:

Output for asic0:
INFO  Total Forwarding drop counter (DSP==1): packets = 2785    , bytes = 331418    
INFO  NPU_HOST PktsOut=      2784    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =              12    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             241    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            1074    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =           30602    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic1:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 7    , bytes = 898    
INFO  Total Forwarding drop counter (DSP==1): packets = 16588    , bytes = 1973810    
INFO  NPU_HOST PktsOut=     16584    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =             106    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =            1178    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            9050    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =          149572    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

cisco@sfd-01:~$ 
```

The following is show output of command of "show platform npu counters -n asic0 -l all". This is basically comand with specific namespace / asic-id mentioned and "all" linecard mentioned. This output fetches command output for the specific namespace / asic-id for the Supervisor and then the output for the specific namespace / asic-id for all the linecards.

```
cisco@sfd-01:~$ 
cisco@sfd-01:~$ sudo show platform npu counters -n asic0 -l all

Output for Supervisor:
INFO  ____________________Slice0____________________|____________Slice1___________|____________Slice2___________|____________Slice3___________|____________Slice4___________|____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFGB_RX0 packets         =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFGB_RX1 packets         =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 input packets  =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 input packets  =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  RXPP IFG0 output packets =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 output packets =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  PDVOQ Slice0 bytes       =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  TXPP0 packets            =               0    |TXPP2   =               0    |TXPP4   =               0    |TXPP6   =               0    |TXPP8   =               0    |TXPP10   =               0    |
INFO  TXPP1 packets            =               0    |TXPP3   =               0    |TXPP5   =               0    |TXPP7   =               0    |TXPP9   =               0    |TXPP11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for LINE-CARD0:
INFO  Total Forwarding drop counter (DSP==1): packets = 7320    , bytes = 871080    
INFO  NPU_HOST PktsOut=      7320    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =              22    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             630    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            1964    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =           80160    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

Output for LINE-CARD1:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 7    , bytes = 918    
INFO  Total Forwarding drop counter (DSP==1): packets = 23884    , bytes = 2842034    
INFO  NPU_HOST PktsOut=     23880    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =            1023    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             899    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =          124918    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =          114204    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

cisco@sfd-01:~$ 
```

The following is show output of command of "show platform npu counters -l all". This is basically comand with no specific namespace / asic-id mentioned and "all" linecard mentioned. This output fetches command output for all the specific namespaces / asic-ids for the Supervisor and then the output for all the namespaces / asic-ids for all the linecards. When iterating over namespaces on RP, there are certain namespaces that corresponds to Linecards and Fabric cards on RP, these namespaces may not generate any CLI output when the CLI is invoked on these namespaces. In such cases, the output from these inactive namespaces will be omitted while displaying the output. 

```
Last login: Fri Jun 27 03:52:56 2025 from 192.168.122.120
cisco@sfd-01:~$ sudo show platform npu counters -l all

Output for Supervisor:

Output for asic0:
INFO  ____________________Slice0____________________|____________Slice1___________|____________Slice2___________|____________Slice3___________|____________Slice4___________|____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFGB_RX0 packets         =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFGB_RX1 packets         =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 input packets  =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 input packets  =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  RXPP IFG0 output packets =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 output packets =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  PDVOQ Slice0 bytes       =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  TXPP0 packets            =               0    |TXPP2   =               0    |TXPP4   =               0    |TXPP6   =               0    |TXPP8   =               0    |TXPP10   =               0    |
INFO  TXPP1 packets            =               0    |TXPP3   =               0    |TXPP5   =               0    |TXPP7   =               0    |TXPP9   =               0    |TXPP11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic1:
INFO  ____________________Slice0____________________|____________Slice1___________|____________Slice2___________|____________Slice3___________|____________Slice4___________|____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFGB_RX0 packets         =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFGB_RX1 packets         =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 input packets  =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 input packets  =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  RXPP IFG0 output packets =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 output packets =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  PDVOQ Slice0 bytes       =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  TXPP0 packets            =               0    |TXPP2   =               0    |TXPP4   =               0    |TXPP6   =               0    |TXPP8   =               0    |TXPP10   =               0    |
INFO  TXPP1 packets            =               0    |TXPP3   =               0    |TXPP5   =               0    |TXPP7   =               0    |TXPP9   =               0    |TXPP11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic2:
INFO  ____________________Slice0____________________|____________Slice1___________|____________Slice2___________|____________Slice3___________|____________Slice4___________|____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFGB_RX0 packets         =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFGB_RX1 packets         =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 input packets  =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 input packets  =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  RXPP IFG0 output packets =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 output packets =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  PDVOQ Slice0 bytes       =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  TXPP0 packets            =               0    |TXPP2   =               0    |TXPP4   =               0    |TXPP6   =               0    |TXPP8   =               0    |TXPP10   =               0    |
INFO  TXPP1 packets            =               0    |TXPP3   =               0    |TXPP5   =               0    |TXPP7   =               0    |TXPP9   =               0    |TXPP11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic3:
INFO  ____________________Slice0____________________|____________Slice1___________|____________Slice2___________|____________Slice3___________|____________Slice4___________|____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFGB_RX0 packets         =               0    |IFG_RX2 =               0    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFGB_RX1 packets         =               0    |IFG_RX3 =               0    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 input packets  =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 input packets  =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  RXPP IFG0 output packets =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 output packets =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  PDVOQ Slice0 bytes       =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  TXPP0 packets            =               0    |TXPP2   =               0    |TXPP4   =               0    |TXPP6   =               0    |TXPP8   =               0    |TXPP10   =               0    |
INFO  TXPP1 packets            =               0    |TXPP3   =               0    |TXPP5   =               0    |TXPP7   =               0    |TXPP9   =               0    |TXPP11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

Output for LINE-CARD0:
Output for asic0:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 7    , bytes = 898    
INFO  Total Forwarding drop counter (DSP==1): packets = 21460    , bytes = 2553578    
INFO  NPU_HOST PktsOut=     21456    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =             116    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =            1585    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =           10048    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =          201396    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic1:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 8    , bytes = 992    
INFO  Total Forwarding drop counter (DSP==1): packets = 21436    , bytes = 2550722    
INFO  NPU_HOST PktsOut=     21432    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =             117    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =            1591    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =           10142    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =          202164    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

Output for LINE-CARD1:
Output for asic0:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 8    , bytes = 952    
INFO  Total Forwarding drop counter (DSP==1): packets = 21484    , bytes = 2556434    
INFO  NPU_HOST PktsOut=     21480    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =             909    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             791    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =          110628    |IFG_RX4 =               0    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =          100506    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic1:
INFO  Total Forwarding lookup errors (Fwd-destination==20'h0 or DSP==0): packets = 7    , bytes = 898    
INFO  Total Forwarding drop counter (DSP==1): packets = 21460    , bytes = 2553578    
INFO  NPU_HOST PktsOut=     21456    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =             793    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =             911    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =          100762    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =          111012    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow

cisco@sfd-01:~$ 
```

The following are show command output of "show platform npu counters" command as executed on LC. They can be executed for a specific namespace / asic-id mentioned, or no specific namespace / asic-id mentioned. But, they can not be executed with -l option (location option) mentioned. The location option is valid only for RP and not for Linecards. 

```
cisco@sfd-lc0:~$ 
cisco@sfd-lc0:~$ sudo show platform npu counters -n asic0
INFO  Total Forwarding drop counter (DSP==1): packets = 9840    , bytes = 1170960    
INFO  NPU_HOST PktsOut=      9840    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =              32    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             846    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            2842    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =          107616    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
cisco@sfd-lc0:~$ 
```

```
cisco@sfd-lc0:~$ 
cisco@sfd-lc0:~$ sudo show platform npu counters

Output for asic0:
INFO  Total Forwarding drop counter (DSP==1): packets = 2520    , bytes = 299880    
INFO  NPU_HOST PktsOut=      2520    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =               9    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =               0    |IFG_RX5 =             217    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =             774    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =               0    |IFG_RX5 =           27578    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
Output for asic1:
INFO  Total Forwarding drop counter (DSP==1): packets = 12336    , bytes = 1467984    
INFO  NPU_HOST PktsOut=     12336    , PktsIn (from TXPP)=         0    
INFO  ___________________Slice0_____________________|____________Slice1___________|___________Slice2____________|___________Slice3____________|___________Slice4____________|_____________Slice5___________|
INFO  IFG_RX0 packets          =               0    |IFG_RX2 =               0    |IFG_RX4 =              41    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 packets          =               0    |IFG_RX3 =            1062    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  IFG_RX0 bytes            =               0    |IFG_RX2 =               0    |IFG_RX4 =            3616    |IFG_RX6 =               0    |IFG_RX8 =               0    |IFG_RX10 =               0    |
INFO  IFG_RX1 bytes            =               0    |IFG_RX3 =          135120    |IFG_RX5 =               0    |IFG_RX7 =               0    |IFG_RX9 =               0    |IFG_RX11 =               0    |
INFO  RXPP IFG0 packets        =               0    |RXPP2   =               0    |RXPP4   =               0    |RXPP6   =               0    |RXPP8   =               0    |RXPP10   =               0    |
INFO  RXPP IFG1 packets        =               0    |RXPP3   =               0    |RXPP5   =               0    |RXPP7   =               0    |RXPP9   =               0    |RXPP11   =               0    |
INFO  SMS IFG0 write packets   =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 write packets   =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  REASSEMBLY Slc0 packets  =               0    |REAS1   =               0    |REAS2   =               0    |REAS3   =               0    |REAS4   =               0    |REAS5    =               0    |
INFO  PDVOQ Slice0 packets     =               0    |PDVOQ1  =               0    |PDVOQ2  =               0    |PDVOQ3  =               0    |PDVOQ4  =               0    |PDVOQ5   =               0    |
INFO  FILB Slice0 packets      =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  FILB Slice0 bytes        =               0    |FILB1   =               0    |FILB2   =               0    |FILB3   =               0    |FILB4   =               0    |FILB5    =               0    |
INFO  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX--C--R--O--S--S--XXXXXXXXXX--B--A--R--XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|
INFO  TXPDR Slice0 packets     =               0    |TXPDR1  =               0    |TXPDR2  =               0    |TXPDR3  =               0    |TXPDR4  =               0    |TXPDR5   =               0    |
INFO  TXCGM Slice0 packets     =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 bytes       =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 UC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  TXCGM Slice0 MC packets  =               0    |TXCGM1  =               0    |TXCGM2  =               0    |TXCGM3  =               0    |TXCGM4  =               0    |TXCGM5   =               0    |
INFO  SMS IFG0 read packets    =               0    |SMS2    =               0    |SMS4    =               0    |SMS6    =               0    |SMS8    =               0    |SMS 10   =               0    |
INFO  SMS IFG1 read packets    =               0    |SMS3    =               0    |SMS5    =               0    |SMS7    =               0    |SMS9    =               0    |SMS 11   =               0    |
INFO  IFGB_TX0 packets         =               0    |IFGB2   =               0    |IFGB4   =               0    |IFGB6   =               0    |IFGB8   =               0    |IFGB10   =               0    |
INFO  IFGB_TX1 packets         =               0    |IFGB3   =               0    |IFGB5   =               0    |IFGB7   =               0    |IFGB9   =               0    |IFGB11   =               0    |
INFO  IFG_TX0 good packets     =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good packets     =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  IFG_TX0 good bytes       =               0    |IFG_TX2 =               0    |IFG_TX4 =               0    |IFG_TX6 =               0    |IFG_TX8 =               0    |IFG_TX10 =               0    |
INFO  IFG_TX1 good bytes       =               0    |IFG_TX3 =               0    |IFG_TX5 =               0    |IFG_TX7 =               0    |IFG_TX9 =               0    |IFG_TX11 =               0    |
INFO  (*) = counter overflow
cisco@sfd-lc0:~$ 
```

## 6 Serviceability and Debug

- The system logging mechanisms explained in section 4.2 shall be used.

## 7 Unit Test Cases

**Test cases for show platform npu commands (-n parameter is optional):**
1. Issue a show platform npu command on RP with no location mentioned and a specific namespace mentioned.
2. Issue a show platform npu command on RP with no location mentioned and no namespace mentioned (should fetch data from all namespaces).
3. Issue a show platform npu command on LC with no location mentioned and a specific namespace mentioned.
4. Issue a show platform npu command on LC with no location mentioned and no namespace mentioned (should fetch data from all namespaces).

**Test cases for config platform cisco commands (-n parameter is mandatory):**
13. Issue a config platform cisco command on RP with no location mentioned and a specific namespace mentioned.
14. Issue a config platform cisco command on RP with no location mentioned and namespace set to "all" (should execute on all namespaces).
15. Issue a config platform cisco command on RP with no location mentioned and no namespace mentioned (should show error - namespace is mandatory).
16. Issue a config platform cisco command on RP with specific line card as location and namespace set to "all".
17. Issue a config platform cisco command on RP with "all" as location and specific namespace mentioned.
18. Issue a config platform cisco command on RP with "all" as location and namespace set to "all".

**Test cases for location-based execution:**
5. Issue a show platform npu command on RP by mentioning specific line card as location and a specific namespace mentioned.
6. Issue a show platform npu command on RP by mentioning specific line card as location and no namespace mentioned (should fetch data from all namespaces on that LC).
7. Issue a show platform npu command on RP by mentioning "all" as location and a specific namespace mentioned.
8. Issue a show platform npu command on RP by mentioning "all" as location and no namespace mentioned
9. Issue a show platform npu command on LC by mentioning specific line card as location and a specific namespace mentioned.
10. Issue a show platform npu command on LC by mentioning specific line card as location and no namespace mentioned
11. Issue a show platform npu command on LC by mentioning "all" as location and a specific namespace mentioned.
12. Issue a show platform npu command on LC by mentioning "all" as location and no namespace mentioned

## 8 References

NA
