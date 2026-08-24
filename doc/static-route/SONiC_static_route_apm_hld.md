# SONiC Active Performance Monitoring for Static Routes HLD

## Table of Contents

### 1. Revision

- Initial version

### 2. Scope

This document describes the design for using Active Performance Monitoring (APM) to check the availability of static route next hops or application endpoints. Based on the state of APM sessions, it manages the installation, update, and removal of static routes in the system.

### 3. Definitions/Abbreviations

- **APM**: Active Performance Monitoring
- **APM**: Bidirectional Forwarding Detection
- **Probe**: A synthetic packet sent to measure the performance of a network path.

### 4. Overview

Active Performance Monitoring (APM) in SONiC is a proactive framework for monitoring network performance using synthetic probes. APM enables real-time measurement of critical metrics such as reachability, round-trip latency, jitter, and packet loss across network paths. Operators can configure APM to monitor connectivity between SONiC switches or between a switch and remote endpoints.

This document discusses the use of APM probes to monitor the reachability and performance of next-hop IP addresses associated with static routes. By continuously probing these next hops, SONiC can automatically detect failures or degraded performance. When a probe indicates that a next hop is unreachable or fails to meet defined performance thresholds, SONiC can update the static route configuration by removing the affected next hop. This automated corrective action ensures that traffic is dynamically rerouted over healthy paths, minimizing downtime.

In the following example, southbound static routes are installed by an operator to VM-based hosts running anycast-style services. APM is used to monitor the reachability of these hosts. If a host becomes unreachable, SONiC will automatically remove the static route to that host and stop advertising it to the network. This allows the network to quickly adapt to changes in host availability without manual intervention.

![Static Route Tracking Example](./APM-Static-Route-tracking-in-a-EVPN-topology.png)

This feature has many similarities with BFD for static route, but also some key differences. It's discussed in details later in the document.

- StaticRouteBFD creates the BFD sessions internally for each nexthop while StaticRouteAPM requires the user to create APM sessions for each nexthop.

- StaticRouteBFD is inherently a bidirectional protocol and requires a matching configuration on both the endpoints, while StaticRouteAPM is unidirectional and doesn't require any configuration on the other side.

### 5. Requirements

1. Monitor static route configuration and if programmed with an APM session, take the ownership of the static route programming.
2. Monitor the state of APM sessions and update static routes based on the current state of each APM session.
3. Manage the lifecycle of static route configuration for FRR including:
   - Creation/deletion of Static Route.
   - Addition/deletion of nexthops as needed.
4. Handle application restarts gracefully by recovering static route and APM session states from the Redis databases (CONFIG_DB, APPL_DB, and STATE_DB) without impacting any existing installed static routes or active APM sessions.

### 6. Architecture Design

A new component, StaticRouteApm, has been introduced to support integration of static route with APM functionality. It will run as a separate process inside the SWSS container. StaticRouteApm will monitor the static route configuration in CONFIG_DB, manage APM sessions in APPL_DB, and update static routes in APPL_DB based on the state of APM sessions in STATE_DB. StaticRouteApm will take ownership of static routes that are configured with APM by checking the "APM" field in CONFIG_DB's STATIC_ROUTE_TABLE. If the "APM" field is set to "true", StaticRouteApm will manage the static route and its nexthops based on the state of APM sessions.

### 7. High-Level Design

## System Overview with Static Route APM

In the diagram below, StaticRouteApm monitors the STATIC_ROUTE_TABLE in config_db. When a static route is configured with APM, StaticRouteApm checks for a corresponding APM session in config_db's APM_TABLE. If a matching session exists, it then checks appl_db for an existing APM session. If none is found, StaticRouteApm creates a new APM session entry in appl_db's APM_SESSION_TABLE.

Additionally, StaticRouteApm monitors the APM session state in state_db. Based on the session state, it updates the STATIC_ROUTE_TABLE in appl_db to add, delete, or modify the static route's nexthop list.

To ensure compatibility with the existing bgpcfgd component StaticRouteMgr, an optional "APM" field is introduced in STATIC_ROUTE_TABLE. When this field is present and set to a valid APM entry in config_db, StaticRouteMgr ignores the static route, and StaticRouteApm takes over its handling. It writes the static route to appl_db based on the APM session state and nexthop configuration.

To prevent StaticRouteTimer from deleting static routes created by StaticRouteApm, the "expiry" field is set to "false" when writing to appl_db's STATIC_ROUTE_TABLE. When StaticRouteTimer encounters "expiry"="false", it skips deletion of that entry.

<img src="static_rt_APM_overview.png" alt="Static Route APM Overview Diagram" width="500" />

## APM session local IP address

StaticRouteApm requires the user to create an APM session for each nexthop in an APM-enabled static route. The nexthop IP should be used as the APM destination IP.

To determine the APM local address (i.e., the source IP address of the APM packet), the following logic is applied:

1. If a source IP is explicitly configured in the APM session, use that IP address.
2. If not, check if the src-intf field is configured in the APM session. If yes, use the IP address assigned to that interface.
3. If neither is configured, determine the interface name for the nexthop:
    a. Look up the interface name in config_db's INTERFACE_TABLE.
    b. If the interface is a PortChannel, check PORTCHANNEL_TABLE.
    c. If it's a VLAN, check VLAN_TABLE.
    d. Use the first IP address listed for that interface.
    e. If no IP address is configured on the interface, APM probes will fail with an error indicating that no source IP is available for the session.

In the example (assume the "APM" field is "true"), for the nexthop "20.0.10.3", the corresponding interface name is PortChannel10.
From the interface configuration, we can found two IP addresses, an IPv4 address 20.0.10.1 and an IPv6 address 2603:10E2:400:10::1. we choose 20.0.10.1 because the nexthop IP address is IPv4 address.

<img src="static_rt_APM_cfg.png" alt="Static Route APM Configuration Example" width="800" />
<img src="static_rt_APM_interface.png" alt="Static Route APM Interface Example" width="500" />

## DB changes

An optional fields called "APM" is introduced in the STATIC_ROUTE_TABLE. The default value of this field is false when this field is not configured. If the "APM" field is set to "true", StaticRouteApm will take ownership of the static route and set the "expiry" field to "false" in the corresponding appl_db entry. StaticRouteTimer will skip the timeout checking for this static route entry based on the "expiry" field value.

[*Reference: STATIC_ROUTE_TABLE schema:*
 [STATIC_ROUTE table in CONFIG_DB](https://github.com/Azure/SONiC/blob/master/doc/static-route/SONiC_static_route_hdl.md#3211-static_route).]

```JSON
;Defines IP static route  table
;
;Status: stable

key                 = STATIC_ROUTE|vrf-name|prefix ;
vrf-name            = 1\*15VCHAR ; VRF name
prefix              = IPv4Prefix / IPv6prefix
nexthop             = string; List of gateway addresses;
ifname              = string; List of interfaces
distance            = string; {0..255};List of distances.
                      Its a Metric used to specify preference of next-hop
                      if this distance is not set, default value 0 will be set when this field is not configured for nexthop(s)
nexthop-vrf         = string; list of next-hop VRFs. It should be set only if ifname or nexthop IP  is not
                      in the current VRF . The value is set to VRF name
                      to which the interface or nexthop IP  belongs for route leaks.
blackhole           = string; List of boolean; true if the next-hop route is blackholed.
                      Default value false will be set when this field is not configured for nexthop(s)
bfd                 = string; "true" or "false". "true" if use BFD to monitor nexthop
                      Default value false will be set when this field is not configured for nexthop(s)
apm                 = string; List of APM session names (one for each nexthop);
                      If this field is set to a valid APM session for any of the nexthops, StaticRouteApm will monitor the corresponding APM session state to update the Static Route's active nexthops. If some nexthops have a valid APM session and others do not, the ones without a valid session will be treated as inactive.
expiry              = string; "true" or "false". "false" if need to skip timeout checking
```

## Internal tables in StaticRouteApm

StaticRouteApm uses three internal tables (implemented as dictionaries or maps) to monitor nexthops via APM sessions and update static routes accordingly:

1. TABLE_CONFIG
    - purpose: Caches entries from config_db's STATIC_ROUTE_TABLE for routes with valid APM sessions.
    - key: {vrf, prefix}
    - value: {nexthop list, APM session list, other static route parameters (e.g., vrf, blackhole, etc.)}

2. TABLE_APM
    - purpose: Tracks APM sessions actively used by StaticRouteApm.
    - key: APM session name
    - value: {nexthop IP address, APM session parameters (e.g., type, frequency, multiplier, etc.)}

3. TABLE_SRT
    - purpose: Stores static routes written to appl_db's STATIC_ROUTE_TABLE by APMRouteManager (with "expiry"="false").
    - note: The nexthop list may differ from the configuration depending on APM session state.
    - key: {vrf, prefix}
    - value: {active nexthop list, APM session list, other static route parameters (e.g., vrf, blackhole, etc.)}

<img src="static_rt_APM_table.png" alt="Static Route APM Internal Tables" width="400">


## Adding/updating static route flow

When a new static route is added to config_db::STATIC_ROUTE_TABLE, StaticRouteApm first checks for the "APM" field. If it's missing, the route is ignored.

If the route includes "APM", StaticRouteApm checks whether it already exists in TABLE_CONFIG. If not, it adds the route to TABLE_CONFIG and creates an entry in TABLE_SRT with an empty nexthop list (to be updated later based on APM session state). For each nexthop, if a valid APM session is configured, it updates or creates the corresponding entry in TABLE_APM.

If the route already exists in TABLE_CONFIG, the nexthop list is compared to identify additions and deletions. Newly added nexthops are reflected in TABLE_SRT and TABLE_APM. Deleted nexthops are removed from both tables.

Finally, if none of the remaining nexthops have an associated APM session, all related entries in TABLE_CONFIG, TABLE_SRT, and TABLE_APM are deleted, and StaticRouteApm stops managing the route.

## Deleting static route flow

When a static route is removed from config_db::STATIC_ROUTE_TABLE, StaticRouteApm performs the following actions:

If the route exists in TABLE_CONFIG, it is deleted from TABLE_CONFIG and TABLE_SRT. For each nexthop associated with the route, the corresponding APM session entry in TABLE_APM is also removed.

## APM session state update flow

StaticRouteApm will be notified if there is any update in state_db APM_SESSION_TABLE.

* 1\. Look up TABLE_APM, ignore the event if session is not found in local table. Otherwise get the nexthop
* 2\. Look up TABLE_NEXTHOP using nexthop from step #1, get nexthop entry
* 3\. For each prefix in the nexthop entry, lookup TABLE_SRT table to get the static route entry.
    * If the APM session state is UP and this nexthop is in the static route entry's nexthop list, no action needed, break;
    * If the APM session state is UP and this nexthop is NOT in the static route entry's nexthop list:
        * Add this nexthop to the static route entry's nexthop list, set "expiry": "false", write this static route to redis appl_db STATIC_ROUTE_TABLE.
    * If the APM session state is DOWN and this nexthop is NOT in the static route entry's nexthop list, no action needed
    * If the APM session state is DOWN and this nexthop is in the static route entry's nexthop list,
        * Delete this nexthop from the static route entry's nexthop list
            * if the static route entry's nexthop list is empty, delete this static route from redis appl_db, break;
            * if the static route entry's nexthop list is NOT empty, write it to redis appl_db STATIC_ROUTE_TABLE with "expiry"="false".



## Table reconciliation after StaticRouteApm crash/restart

When StaticRouteApm crashes or restarts, it loses all internal table contents. The process must rebuild the internal tables from Redis DB (i.e., configuration DB, application DB, state DB, etc.), perform cross-checking between these tables and Redis DB to ensure consistency, and then start processing events received from Redis DB.



## Dynamic static route "APM" config change support

When the "APM" field of a static route changes (from "true" to "false" or vice versa), the ownership of the static route (between StaticRouteMgr and StaticRouteApm) also changes. StaticRouteMgr and StaticRouteApm must work together to handle this runtime ownership transition.
When StaticRouteMgr receives a static route update (redis SET event), it checks the "APM" field and its local cache. If the "APM" field is "true" and the prefix is present in its local cache (meaning it previously handled this route), StaticRouteMgr deletes it from its local cache and does not perform any further processing for this route.
For static routes without the "APM" field (or with "APM" set to "false"), the current StaticRouteMgr behavior is to compare the nexthop list between the updated static route and the nexthop list in its local cache to determine whether to delete or add nexthops. StaticRouteApm relies on this behavior to handle dynamic changes to the "APM" field.

### APM field changes from "false" to "true"

1. when the "APM" was "false", the route was installed by StaticRouteMgr(config_db), and StaticRouteMgr(config_db) maintains its local cache.
2. When StaticRouteMgr detects "APM" changing from "false" to "true", the StaticRouteMgr deletes the static route from its local cache, but it does NOT uninstall the route from FRR, so the system can still use installed route before the APM session get created and state becomes UP.
   - Note: StaticRouteApm (work with StaticRouteMgr(appl_db)) may update/install the route immediately if APM session is already created and ready. Deleting the route from StaticRouteMgr(config_db) may conflict with StaticRouteApm update, cause race condition and unpredictable result.
3. When StaticRouteApm detects "APM" changing from "false" to "true" (using its local cache), it writes a static route entry to APPL_DB STATIC_ROUTE_TABLE with "APM" field "false" to let StaticRouteMgr(appl_db) install the route with full nexhop list and update StaticRouteMgr(appl_db) local cache.

![APM field change from false to true](static_rt_APM_change_1.png)

4. Depends on the APM session state, StaticRouteApm will update the static route immediately or hold for a while, to not blocking the current traffic.
   - 4.1 If all the APM sessions are DOWN, StaticRouteApm will NOT update the route installed until at least one nexthop become reachable (APM session state becomes UP). Because that APM session may be temporarily DOWN (in state_db) during APM session creation and initialization stage.

![APM session hold when DOWN](static_rt_APM_hold_1.png)

   - 4.2 Considering nexthop sharing, some (or all) of the nexhop APM sessions might be already created and becomes UP, StaticRouteApm updates the routes with new nexthop list (depends on which APM sessions are UP).

![APM session hold when UP](static_rt_APM_hold_2.png)

### APM field changes from "true" to "false"

1. when StaticRouteMgr(config_db) get an updated static route with "APM" field "false", it install the route as usual. Because it will install the route will all the nexthops in the route, it does not need to uninstall the StaticRouteApm installed route (the nexthop list is a subset of configured nexthop list).
2. StaticRouteApm delete the static route entry from APPL_DB.
3. StaticRouteMgr(appl_db) checks the deleted static route key, if the key is also in CONFIG_DB STATIC_ROUTE_TABLE, StaticRouteMgr(appl_db) just clear its local cache, but does not uninstall the route from FRR.

### 8. SAI API

### 9. Configuration and Management

#### 9.1. CLI Enhancements

#### 9.2. YANG model Enhancements

#### 9.3. Config DB Enhancements

#### 9.4 APPL_DB Enhancements

#### 9.5 STATE_DB Enhancements

#### 9.6 Counters DB Enhancements

### 10. Warmboot and Fastboot Design Impact

APM will store the state of all active sessions in the StateDB and restore the state after the application reboots. This will ensure that all active probes will maintain the previous state and will not need to wait for frequency * multiplier time for the session state to be valid again.

### 11. Memory Consumption

This sub-section covers the memory consumption analysis for the new feature: no memory consumption is expected when the feature is disabled via compilation and no growing memory consumption while feature is disabled by configuration.

### 12. Restrictions/Limitations

### 13. Testing Requirements/Design

#### Tests

Description:
* 1\. Plan to use a single testbed to test the StaticRouteApm.
    * 1.1 using 'show APM peer' cnd to check APM session
    * 1.2 test script update state_db APM_SESSION_TABLE directly without peer APM session from remote system, then check the static route in local system.
* 2\. Because that there are design changes in StaticRouteMgr and StaticRouteTimer, need to run regression test (without "APM" field configured, and "APM"="false" configured)described in the following documents.
    * 2.1 https://github.com/sonic-net/SONiC/blob/master/doc/static-route/SONiC_static_route_hdl.md#4-unit-test-and-automation
    * 2.2 https://github.com/sonic-net/SONiC/blob/master/doc/static-route/SONiC_static_route_expiration_hdl.md#tests
* 3\. for the StaticRouteApm, use 2 routes for the following tests
    * 3.1 static route A: has 3 nexthops -- nh_1, nh_2 and nh_3 (the address depends on test setup) in its configuration
    * 3.2 static route B: has 1 nexthop -- nh_2 (same in static route A, to test nexthop overlap among static routes, for APM session creation and APM state update)



#### Non-restart Testing (testcase without application restart/crash)

### 1. Test static route config with "APM"="true"

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
| config static route A with "APM"="true" | Verify APM session creation.  | 1, APM sessions are created  2,this static route is not installed to the system because current APM session state should be DOWN.  |
| change one APM session state to UP state for nh_1 in state_db | verify APM state change handling | static route A (with 1 nexthop nh_1) should be installed to the system. "expiry" field should be "false" in appl_db |
| change one APM session state to UP state for nh_2 in state_db | verify APM state change handling | static route A (with nexthop nh_1 and nh_2) should be installed to the system |
| change one APM session state to UP state for nh_3 in state_db | verify APM state change handling | static route A (with nexthop nh_1,nh_2 and nh_3) should be installed to the system |


### 2. Test static route config update with "APM"="true"

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
| from test #1, reconfig static route A with 2 nexthops nh_1 and nh_2 and "APM"="true" | Verify static route reconfig with APM | 1, APM sessions for nh_3 is removed  2,static route A is updated to the system with 2 nexthop nh_1 and nh_2  |


### 3. Test static route ("APM"="true") removed from config

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
| from test #2, remove static route A from config | Verify static route with APM uninstallation | 1, APM sessions for nh_1 and nh_2 are removed  2,static route A is uninstalled from the system  |


### 4. Test 2 static routes ("APM"="true") share same nexthop

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
| config static route A with 3 nexthops and "APM"="true", change APM sessions state to UP for all the nexthops | install static route A | static route A is installed to the system  |
|config static route B with nh_2 and "APM"="true"|verify shared nexthop|static route B is installed to the system, because the APM session for nh_2 was changed to UP in the above step |


### 5. Test APM session DOWN causing static route updated/removed

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|from above test #4, 2 static routes (sharing nh_2) were installed|pre-install route A and route B||
| change APM session state for nh_2 to DOWN in state_db | verify APM session state handling | 1, nh_2 is removed from satic route A in the system.2, static route B is uninstalled from the system because it has no nexthop in its nexthop list  |


### 6. Test APM session UP causing static route updated/reinstalled

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|from above test #5, APM session for nh_2 is DOWN ||
| change APM session state for nh_2 to UP in state_db | verify APM session state handling | 1, nh_2 is added back to satic route A in the system.2, static route B is reinstalled to the system because nh_2 is active now  |
|remove static route A and B|clear static route for next tests|1, route A and B are removed.2, APM sessions for the nexthops are removed|


## StaticRouteApm retart/crash testing (crash/restart StaticRouteApm process)

Verify StaticRouteApm restore information from redis DB and rebuild its internal data struct and continue to handle config or state change.

### 7. Restart StaticRouteApm between static route config and APM state update

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|configure static route A with "APM"="true"  |verify APM session creation| 3 APM sessions for nh_1/nh_2/nh_3 are created|
|kill and restart StaticRouteApm process|verify StaticRouteApm restart||
|update APM session state to UP for nh_1/nh_2/nh_3|verify APM state handling after restart|static route A (with nh_1/nh_2/nh_3) is installed to the system|


### 8. Restart StaticRouteApm between static route adding/deleting

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|install static route A (result from the above step #7)  |install route A| route A with nh_1/nh_2/nh_3 are installed|
|kill and restart StaticRouteApm process|verify StaticRouteApm restart||
|update APM session state to UP for nh_1/nh_2/nh_3|verify APM state handling after restart|static route A (with nh_1/nh_2/nh_3) is installed to the system|
|config static route B with "APM"="true"|verify internal table recovery after restart|route B is installed because APM session for nh_2 is up before StaticRouteApm is UP|
|kill and restart StaticRouteApm process|verify StaticRouteApm restart||
|remove static route A from config|verify internal table recovery after restart|1,route A should be uninstalled from the system2,APM sessions for nh_1 and nh_3 are removed3, APM session for nh_2 keep unchanged because route B still need it|
||||

## StaticRouteApm APM field dynamic change

Verify StaticRouteApm handling for static route "APM" flag dynamic changing

### 9. Change a static route "APM" field from true to false

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|configure static route A with "APM"="true"  |verify APM session creation| 3 APM sessions for nh_1/nh_2/nh_3 are created|
|update APM session state to UP for nh_1/nh_2/nh_3|verify APM state handling |static route A (with nh_1/nh_2/nh_3) is installed to the system|
|change "APM" flag to "false"|verify flag change handling|1, APM session should be deleted2, StaticRouteApm deletes the static route from appl_db |


### 10. Change a static route "APM" field from false to true

| Step              | Goal                              | Expected Outcome    |
|---------------------------|---------------------------------------|------|
|start from the above test#9   |set a "APM"="false" state inside StaticRouteApm||
|reconfigure static route A with "APM"="true"  |verify APM session creation after config change| 1, 3 APM sessions for nh_1/nh_2/nh_3 are created2, before any APM session becomes UP, StaticRouteApm install the configured static route with its full nexthop list, this update appl_db and StaticRouteMgr(appl_db) local cache|
|update APM session state to UP for nh_1/nh_2/nh_3|verify APM state handling |static route A (with nh_1/nh_2/nh_3, which APM session is UP) is installed to the system|


### 14. Open/Action items

## Traffic convergence acceleration

The StaticRouteApm add/remove nexthop to/from prefix via FRR. This may takes time to recursively update all the FRR route. Using the approach similar to [ECMP acceleration](https://github.com/sonic-net/SONiC/blob/master/doc/ecmp/sonic-ecmp-acceleration.docx) can speed up the traffic convergence.
