---
name: storone-monitor-capacity-and-health
description: >-
  Read live and historical health, capacity, performance and utilization from a StorONE S1 system,
  and route its alerts to email, syslog, SNMP, Seq, Slack or StorONE support.
api: StorONE S1 REST API
base_url: https://{s1-controller-node}
operations:
  - POST /login
  - GET /monitoring/health/current
  - GET /monitoring/capacity/pools/current
  - GET /monitoring/capacity/pools/history
  - GET /monitoring/capacity/provisioning
  - GET /monitoring/capacity/volumes
  - GET /monitoring/utilization/live
  - GET /monitoring/utilization/history
  - GET /monitoring/performance/live
  - GET /monitoring/performance/history
  - GET /monitoring/top/volumes
  - GET /monitoring/alua/live
  - GET /monitoring/evacuations/live
  - GET /monitoring/reserve/current
  - GET /resources/drives/smart
  - GET /resources/drives/performance_analyzer
  - GET /notifications/query
  - GET /notifications/targets/list
  - POST /notifications/targets/email/add
  - POST /notifications/targets/syslog/add
  - POST /notifications/targets/snmp/add
  - POST /notifications/targets/seq/add
  - POST /notifications/targets/slack/add
  - POST /notifications/targets/storone_support/add
  - POST /notifications/test
generated: '2026-08-29'
method: generated
source: >-
  https://docs.onestor.com/books/rest-api/page/monitoring and
  https://docs.onestor.com/books/rest-api/page/notifications
---

# Monitor a StorONE S1 system

S1 runs on the customer's own hardware, so there is no StorONE status page to read. **The system is
its own status page**, and these are the endpoints that report it.

## Poll politely

There are **no rate limits and no rate-limit headers** on this API — nothing will throttle a runaway
loop. The controllers answering these calls are the same controllers serving production host I/O.
Use the `/history` variants for trends and treat `/live` as on-demand.

## Health first

```
GET /monitoring/health/current
```

The single call to make before anything else. Nested payload objects carry a `Status` field
describing the health of the thing they describe (values seen in the reference include `OK` and
`SystemPerformance`) — this is *not* the HTTP outcome, which is always 200 or 400.

## Capacity

```
GET /monitoring/capacity/pools/current      # pool capacity now
GET /monitoring/capacity/pools/history      # pool capacity over time
GET /monitoring/capacity/provisioning       # provisioned vs. consumed
GET /monitoring/capacity/volumes            # per-volume capacity
GET /monitoring/reserve/current             # reserve capacity
```

`provisioning` versus `pools/current` is the thin-provisioning overcommit picture — the number that
matters for "when do we buy drives".

## Performance and utilization

```
GET /monitoring/performance/live
GET /monitoring/performance/history
GET /monitoring/utilization/live
GET /monitoring/utilization/history
GET /monitoring/top/volumes                 # accepts a count of top volumes to show
```

`top/volumes` is the noisy-neighbour call.

## Path and rebuild state

```
GET /monitoring/alua/live         # ALUA path state per host path
GET /monitoring/evacuations/live  # drive/tier evacuation progress
```

Check `alua/live` after any mapping or node change — it is where a half-presented LUN shows up.
Check `evacuations/live` during drive replacement or auto-tiering activity.

## Drives

```
GET /resources/drives/smart
GET /resources/drives/performance_analyzer
GET /resources/drives/list
```

SMART data is the leading indicator of the failure you are about to have.

## Events

```
GET /notifications/query
```

This is where operational faults actually surface. The S1 REST API has no error catalog — every
failure is a `400` with a bare string — so the notification stream, not the API response, is the
diagnostic vocabulary of this system.

## Route alerts outward

```
GET  /notifications/targets/list
POST /notifications/targets/email/add
POST /notifications/targets/syslog/add
POST /notifications/targets/snmp/add
POST /notifications/targets/seq/add
POST /notifications/targets/slack/add       # takes a WebhookUrl
POST /notifications/targets/storone_support/add
POST /notifications/test
PUT  /notifications/targets/enable
PUT  /notifications/targets/disable
```

There is **no generic customer-defined HTTP webhook target** — the Slack target is the only
webhook-shaped one, and it is Slack-specific. If you need events in an arbitrary system, syslog or
Seq is the integration point.

`POST /notifications/targets/storone_support/add` routes alerts to StorONE support directly, which
with the node support tunnel (`PUT /nodes/support/tunnel/enable`) is how vendor support gets
visibility. Both are consequential — enabling a support tunnel opens vendor access to the system.
Do not enable one unattended.

Always run `POST /notifications/test` after adding a target. Disable
(`PUT /notifications/targets/disable`) is the reversible form; delete is not.
