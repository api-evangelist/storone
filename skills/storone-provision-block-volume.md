---
name: storone-provision-block-volume
description: >-
  Provision a block volume on a StorONE S1 system and present it to an iSCSI, Fibre Channel or
  NVMe-oF host, using only operations published in the S1 REST API reference.
api: StorONE S1 REST API
base_url: https://{s1-controller-node}
operations:
  - POST /login
  - GET /Version
  - GET /workflows/volume_provisioning/block
  - GET /resources/drives/pools/list
  - GET /applications/list
  - POST /applications/create
  - POST /applications/volumes/create
  - GET /applications/volumes/list
  - GET /hosts/list
  - POST /hosts/create
  - PUT /hosts/wwn/add
  - POST /hosts/pair
  - POST /applications/mappings/add
  - GET /applications/mappings/list
generated: '2026-08-29'
method: generated
source: https://docs.onestor.com/books/rest-api
---

# Provision a block volume on StorONE S1

Every operation below is published in the S1 REST API reference at
<https://docs.onestor.com/books/rest-api>. Nothing here is inferred.

## Before you start

S1 is customer-deployed software. The base URL is the address of one of the customer's own
controller nodes — there is no StorONE-hosted API. Get it from the operator; do not guess it.

Two things about this API will bite an agent that assumes normal REST:

- **The action is in the path, not the verb.** `PUT` is used for edits, restores, mounts and
  enables alike. Read the path.
- **There is no idempotency key.** Retrying a create makes a second object. If a create times out,
  `list` before you retry.

## 1. Authenticate

```
POST /login
Content-Type: application/json
{"Username": "<user>", "Password": "<password>", "InactivityTimeoutInMinutes": 30}
```

Read `SessionToken` from the 200 response and send it as the `Authorization` header on every
subsequent call. A 401 anywhere means log in again — the reference only documents 401 on `/login`,
but the token expires on all of them.

Confirm what you are talking to before you write anything:

```
GET /Version
```

Two StorONE customers can be on materially different releases. The published reference lags the
3.8 line, so treat the operation set as a floor.

## 2. Read the guided path (optional but cheap)

```
GET /workflows/volume_provisioning/block
```

S1 exposes its own provisioning workflow. Read it before composing your own sequence.

## 3. Confirm there is a pool

```
GET /resources/drives/pools/list
```

Volumes are backed by pools of approved drives. If none exists, drives must be approved
(`PUT /resources/drives/approve`) and a pool created (`POST /resources/drives/pools/create`) —
both are operator decisions about physical hardware. Do not make them unattended.

## 4. Create or select the application

An "application instance" is S1's consistency group: the unit of snapshotting and replication.

```
GET /applications/list          # filter by application instance name
POST /applications/create
```

Group volumes that must be snapshotted together into one application.

## 5. Create the volume

```
POST /applications/volumes/create
```

Then verify, always:

```
GET /applications/volumes/list  # filter by volume name
```

Entities are addressed by **name**, not by opaque ID. Choose names that will not collide.

## 6. Register the host

```
GET /hosts/list
POST /hosts/create
PUT /hosts/wwn/add
POST /hosts/pair
```

The host is the initiator — iSCSI, FC or NVMe-oF. WWNs identify it. Add every WWN the initiator
presents, or paths will be missing.

## 7. Map the volume to the host

```
POST /applications/mappings/add
GET /applications/mappings/list
```

This is the step that makes the LUN visible. Confirm with the list call.

## Reversing this

- The mapping is fully reversible: `DELETE /applications/mappings/delete` and re-add later.
- The volume is **not**. `DELETE /applications/volumes/delete` has no documented undo. Take a
  snapshot first (`POST /applications/snapshots/take`) — it is the only undo this API offers, and it
  must exist *before* the destructive call.

## Errors

Every failure is `400 Bad Request` with a bare JSON string body. Validation errors, missing objects
and permission failures are indistinguishable by status code. Do not blind-retry a 400 — read the
string, then confirm actual state with the matching `/list` call.
