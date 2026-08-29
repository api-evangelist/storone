---
name: storone-snapshot-and-restore
description: >-
  Take, schedule and restore StorONE S1 snapshots — the only undo path the S1 REST API publishes —
  and understand what restore actually does before relying on it.
api: StorONE S1 REST API
base_url: https://{s1-controller-node}
operations:
  - POST /login
  - GET /applications/list
  - GET /applications/volumes/list
  - POST /applications/snapshots/take
  - GET /applications/snapshots/list
  - PUT /applications/snapshots/schedule
  - PUT /applications/snapshots/restore
  - PUT /applications/snapshots/vss
  - DELETE /applications/snapshots/delete
  - POST /applications/mappings/add
generated: '2026-08-29'
method: generated
source: https://docs.onestor.com/books/rest-api/page/applications
---

# Snapshot and restore on StorONE S1

Snapshots are the **only reversal path** the S1 REST API documents. Deletes of volumes,
applications, shares, object stores and access keys have no undo. Treat a snapshot as the
prerequisite for any destructive call, not as a nice-to-have.

## The one thing to know about restore

```
PUT /applications/snapshots/restore
```

Its published description is **"Create a new volume from a snapshot."**

It does not roll the original volume back. It creates a *new* volume beside it, named with the
`Suffix` you supply. The source volume is untouched. So the safe undo pattern on S1 is
**restore beside, verify, then re-map** — not "restore and hope".

Body:

```json
{"Application": "<app>", "Volume": "<volume>", "Suffix": "<suffix-for-new-volume>"}
```

Query parameters: `cgid` (consistency group id, int64), `snapshot` (snapshot id, int64), `async`
(boolean), `uncommitAndAggregateId` (string).

After restoring, the new volume is not visible to any host until you map it:
`POST /applications/mappings/add`.

## Take a manual snapshot

```
POST /applications/snapshots/take
{"All": true|false, "Application": "<app>", "Volume": "<volume>", ...}
```

Take one before every delete. There is no idempotency key on this API, so if the call times out,
run `GET /applications/snapshots/list` before retrying rather than firing again.

## Schedule snapshots

```
PUT /applications/snapshots/schedule
{"Frequency": ["..."], "Retention": ["..."], "Enable_vss": true, "Force": true}
```

`Retention` is what actually bounds your ability to undo. StorONE publishes no default and no
maximum — the window is entirely whatever the customer configured here. If you need to state a
recovery window to a human, read this schedule; do not assume one.

`Enable_vss` produces application-consistent snapshots on Windows. Configure it per volume with
`PUT /applications/snapshots/vss`.

## Consistency groups

An application instance *is* the consistency group. Volumes that must be snapshotted at the same
instant belong in the same application. The `cgid` query parameter addresses the group directly
where you have its id.

## Deleting snapshots

```
DELETE /applications/snapshots/delete
```

Irreversible, and it destroys your undo. Never delete a snapshot in the same sequence as a
destructive change to the volume it covers.

## Async

Snapshot operations accept an `async` boolean. There is **no documented status resource to poll and
no callback**, so if you go async you must confirm completion with
`GET /applications/snapshots/list` rather than awaiting a completion signal.
