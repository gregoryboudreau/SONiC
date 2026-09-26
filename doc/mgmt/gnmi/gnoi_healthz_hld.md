# gNOI Healthz High Level Design

## Revision

| Rev | Date | Change |
|-----|------|--------|
| 0.1 | 2025-06-16 | Initial diagnostic collection design |
| 0.2 | 2026-09-26 | DLDD fault events, durable host catalog, OpenConfig component health, standard RPC lifecycle, and separate legacy diagnostic compatibility |

## Scope and authority

This design covers DLDD faults and rule-defined diagnostic archives exposed through gNOI Healthz and OpenConfig platform Healthz telemetry. The wire and lifecycle contracts are the [gNOI Healthz v0.3.0 proto](https://github.com/openconfig/gnoi/blob/v0.3.0/healthz/healthz.proto) and [Healthz design](https://github.com/openconfig/gnoi/blob/v0.3.0/healthz/README.md), the version pinned by `sonic-gnmi`, together with [platform health state](https://github.com/openconfig/public/blob/master/release/models/platform/openconfig-platform-healthz.yang) and [platform faults](https://github.com/openconfig/public/blob/master/release/models/platform/openconfig-platform-healthz-fault.yang). The [DLDD HLD](../../device_local_diagnosis/device-local-diagnosis-daemon.md) defines fault publication and artifact generation. The general [gNMI server design](SONiC_GNMI_Server_Interface_Design.md) and [Docker-to-host D-Bus design](../Docker%20to%20Host%20communication.md) define the existing management boundary.

Healthz `Get`, `List`, and `Acknowledge` operate on recorded events. They do not initiate collection. `Artifact` streams an existing archive, possibly waiting within a bounded deadline for DLDD's asynchronous archive generation to finish. Standard `Check` for DLDD components remains unsupported until a component-specific validation procedure exists. The existing diagnostic-selector route is retained separately for compatibility.

## Architecture and ownership

| Owner | Responsibility |
|-------|----------------|
| `sonic-host-services/dldd` | Publish the complete `FAULT_INFO` row and a compact transition record atomically at first publication, including inactive-only local recovery, and at each later actual status change. Preserve rule-defined actions and archive generation. |
| `sonic-host-services/host_modules/healthz.py` | Consume transitions, reconcile current faults, own a bounded durable event and acknowledgement catalog, compute component aggregate state, and expose quick metadata methods through the existing host D-Bus server. |
| `sonic-gnmi/gnmi_server/gnoi_healthz*.go` | Authenticate, validate OpenConfig component paths, map host metadata to Healthz protobufs, and stream allowlisted artifact files through the contained resolver. |
| `sonic-mgmt-common/translib` | Map `COMPONENT_HEALTH_INFO` to OpenConfig component `healthz/state` for GET and ON_CHANGE; retain the separate `FAULT_INFO` fault-subtree translation. |

The host module uses Python `sqlite3` under a root-owned `/var/lib/sonic/healthz/` directory. It stores events, acknowledgement flags, component aggregate state, and the consumed Redis stream checkpoint. Neither TCP nor UDS gNMI server instance owns this catalog. The gNMI container requires no new writable mount for the catalog; metadata crosses the existing `sonic_service_client` D-Bus boundary, and existing read-only `/mnt/host` access supports contained archive streaming. Host-only vendor commands and ASIC diagnostic shells remain on the host. This feature does not add a general gNOI backend framework or an on-demand DLDD collection worker.

### Publication, replay, and aggregate state

DLDD replaces `FAULT_INFO` and appends to the bounded `DLDD_FAULT_TRANSITIONS` Redis stream in one atomic Redis Lua operation on the first published fault row and every actual published status change. The record has `producer`, `fault_key`, `component`, `component_type`, `symptom`, `status`, `occurrence`, `observed_at` (Unix epoch seconds), and optional `artifact_id` fields. Exact `MAXLEN 10000` bounds the stream. Repeated publication and metadata-only changes create no new event. DLDD does not manage Healthz acknowledgement, event retention, or artifact-completion polling.

One bounded host worker consumes stream IDs idempotently. It commits events, acknowledgement-independent aggregate updates, and the stream checkpoint in one SQLite transaction; failed `COMPONENT_HEALTH_INFO` projection is retried from the catalog. It republishes known aggregate state after restart and reconciles the current `FAULT_INFO` snapshot at startup and after Redis reconnect without blocking D-Bus methods. The host module exposes quick `get`, `list`, and `ack` methods through the existing D-Bus server; they take JSON request strings and return an `int32` result code plus a JSON response string. An `ACTIVE` fault creates an `UNHEALTHY` event. The first observed unhealthy component state, or a later `HEALTHY` to `UNHEALTHY` transition, increments `unhealthy-count`; another overlapping active fault creates its own event without incrementing the component count. Clearing the last active fault creates a `HEALTHY` event. An initial `INACTIVE` row after successful local recovery creates a `HEALTHY` event with its new artifact, if any, without inventing an earlier unhealthy event or count increment. The `last-unhealthy` value comes from a confirmed unhealthy observation, never the clear time. The projected `COMPONENT_HEALTH_INFO|<URI-escaped component>` row contains `status` and `unhealthy_count`, plus `last_unhealthy` (Unix epoch nanoseconds) when such an observation exists.

The stream size must cover the supported host-backend outage window, and Redis Stream and Lua `EVAL` support must be checked on the target image. If trimming has passed the checkpoint or Redis loses unconsumed records across reboot, the worker logs an explicit history gap, preserves catalogued events, and reconciles present rows without fabricating lost transitions. An explicit `FAULT_INFO` snapshot row may create one current-state observation event, tagged `source=snapshot` and deduplicated separately from stream transitions; it cannot reconstruct unseen transitions. A missing or expired `FAULT_INFO` row is not proof of recovery. A complete active/clear cycle can be lost during a longer outage, so this design does not promise unlimited history recovery. `COMPONENT_HEALTH_INFO` is published only for components with a known assessment; lack of a fault row does not imply `HEALTHY`.

The fault `origin_time` remains fixed for the life of a retained row. `last_detection_time` identifies the latest sample that confirmed the underlying condition asserted; a clear observation does not update it. An earlier locally vendored `openconfig-platform-healthz-fault.yang` description included clear time; the vendored model is aligned to this asserted-sample contract for this integration. The health-state YANG defines `status`, `last-unhealthy`, and `unhealthy-count` with ON_CHANGE telemetry.

### Event IDs, artifacts, and retention

The host catalog assigns Healthz event identity once per source transition. When an event has a **new** DLDD archive, its `artifact_id` is `ComponentStatus.id` immediately and the first `ArtifactHeader.id` once the final regular archive exists. When no new archive belongs to the transition, the backend allocates an opaque event ID and persists its association with the stream ID. A retained `INACTIVE` fault row may still carry its earlier artifact ID; the recovery event then has a distinct ID and does not advertise that archive again. If a new archive is attached after an event is recorded, the existing event ID stays fixed. `ComponentStatus.created` records the stored observation time. The fields remain conceptually separate so a future event-specific Check can add artifacts without renaming the event.

The catalog retains events and acknowledgement across service and device restarts within a disk bound. It prunes the oldest acknowledged events first. Acknowledgement is a catalog flag, not an archive deletion request. DLDD owns its separate bounded archive retention. The backend advertises an `ArtifactHeader` only while the final regular archive file exists: no header during asynchronous collection, a header after atomic final rename, and no header immediately after archive garbage collection. The event ID still permits `Artifact(event ID)` to perform its existing bounded wait while collection is pending. The historical event may remain after archive removal. `ComponentStatus.expires` stays unset unless the archive's actual deletion deadline is known. The pinned Healthz README recommends removing acknowledged artifacts before unacknowledged ones when reclaiming space; existing DLDD archive retention does not consult the host acknowledgement flag in this initial scope. That SHOULD-level recommendation remains a known limitation of archive garbage collection, distinct from catalog pruning.

## Standard RPC contract

All path-bearing standard Healthz requests use the complete component path `/components/component[name=X]`, with the component name key. The `healthz` subtree and diagnostic selectors are not part of a standard component path. Healthz protobuf status is an assessment: successful file collection alone cannot establish `STATUS_HEALTHY`. The methods retain existing gNOI authentication and authorization. Read-only gNMI mode permits `Get`, `List`, and `Artifact` reads while gating `Acknowledge` and `Check` as writes.

| RPC | DLDD behavior |
|-----|---------------|
| `Get` | Read the latest stored event for the requested component and include known child statuses only through published OpenConfig platform parent relationships. If only children have events, return a parent wrapper with `STATUS_UNSPECIFIED`, no event ID, `created`, artifact, or acknowledgement, and child statuses in `subcomponents`; this does not assert parent health. Return `NotFound` when neither parent nor children have status. If a component type has no published parent relationship, exact-component retrieval is the tested scope. |
| `List` | Read retained events for the requested component. Exclude acknowledged events by default; include them when `include_acknowledged` is true. |
| `Acknowledge` | Validate path and event ID, idempotently mark that `(component, event)` as acknowledged, and return the updated `ComponentStatus`. It does not delete an artifact. |
| `Check` | Return `Unimplemented` for a standard DLDD component path, with or without `event_id`, until a defined component procedure exists. A future event-specific procedure may add diagnostics but must preserve prior artifacts and derive status from validation, not collection completion. |
| `Artifact` | Stream a contained, allowlisted completed archive as header, one or more data frames, and trailer. A call using the pending event's archive ID may wait within the existing bound for the final file even though `Get`/`List` currently omit its `ArtifactHeader`. Retain size/SHA-256 metadata and cancellation. Use a basename for `FileArtifactType.name` and `application/gzip` for a `.tar.gz` archive. |

`Get` returns the latest event even when that event has been acknowledged; `List` applies the acknowledged filter. A later recovery event is separate from the earlier unhealthy event, so `List` can retain both while `Get` reports the latest. For a parent `Get`, names are never parsed to invent descendant relationships.

### Legacy diagnostic compatibility

`/components/component[name=X]/healthz/alert-info`, `/critical-info`, and `/all-info` are nonstandard diagnostic selectors. Existing `sonic-gnmi` code accepts them through `Check` and calls the host `debug_info.collect`; `Get` does not collect on these paths. Keep this compatibility route separate from standard component paths and preserve the contained resolver for legacy absolute `/tmp/dump` artifacts. The diagnostic collector is synchronous while the current D-Bus caller has a ten-second timeout. If long-running legacy collection must be supported, address its timeout or asynchrony as a separate bounded change. Collection success must not be reported as a healthy component assessment.

The in-repository `sonic-gnmi` tests use a legacy selector and the CLI accepts arbitrary Healthz path JSON. The `sonic-mgmt` port-debug Healthz helper is currently a stub. These sources do not establish whether deployed or external consumers use the selectors; retain compatibility until that question has been checked in the deployed environment.

## Impact and verification

No SAI API change is required. The event catalog and checkpoint survive warm, fast, and cold restarts within the configured retention bounds; Redis loss or stream trimming can still leave an explicitly reported gap. The host backend reprojects its durable aggregate after restart. A gNMI container restart does not reset acknowledgement or event IDs.

Focused tests cover first `ACTIVE` and inactive-only publication, overlapping faults, actual status changes versus metadata refreshes, stable event/artifact IDs, replay and gap detection, projection retry, restart persistence, bounded pruning, and archive disappearance. RPC tests cover authenticated fault-to-`Get`/`List`/`Artifact` flow, repeat `Acknowledge`, default `List` filtering, read-only mode, path rejection, `Check` with and without `event_id`, artifact containment and metadata, and legacy diagnostic-selector compatibility. Integration checks exercise real D-Bus metadata calls. DUT qualification verifies one fault with an archive, one without, live gNMI GET/ON_CHANGE and gNOI lifecycle, and catalog persistence after host-service and gNMI restarts. Stream support and the bounded outage window must be verified on that image before claiming complete transition history.

## Implementation file map

| Repository | Files | Role |
|------------|-------|------|
| `sonic-host-services` | `dldd/telemetry.py`, `dldd/orchestrator.py`, `dldd/artifacts.py` | Atomic fault/transition publication, corrected timestamps, and existing asynchronous archive generation |
| `sonic-host-services` | `host_modules/healthz.py`, `host_modules/healthz_catalog.py`, `scripts/sonic-host-server`, host-module tests | D-Bus methods, stream worker, reconciliation, durable catalog, aggregate projection |
| `sonic-gnmi` | `gnmi_server/gnoi_healthz.go`, `gnoi_healthz_artifact.go`, `gnoi_healthz_artifact_resolver.go`, `sonic_service_client/dbus_client.go` and tests | Authentication, standard/legacy path handling, protobuf mapping, D-Bus metadata calls, contained artifact streaming |
| `sonic-mgmt-common` | `models/yang/common/openconfig-platform-healthz*.yang`, `translib/pfm_healthz.go`, `translib/pfm_fault.go` and tests | Component health state GET/ON_CHANGE and existing fault subtree mapping |
| `sonic-buildimage`, `whitebox-sonic-buildimage`, `sonic-mgmt` | Package/gitlink pins and focused qualification | Update after producer tests and DUT verification; keep public and whitebox equivalents synchronized |
