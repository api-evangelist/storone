---
name: storone-provision-nas-and-object
description: >-
  Provision SMB/NFS file shares or S3-compatible object storage on StorONE S1, in the strict order
  the object graph requires.
api: StorONE S1 REST API
base_url: https://{s1-controller-node}
operations:
  - POST /login
  - GET /workflows/volume_provisioning/NAS
  - GET /workflows/volume_provisioning/Object
  - POST /applications/create
  - POST /applications/volumes/create
  - POST /applications/filesystems/add
  - PUT /applications/filesystems/mount
  - GET /applications/filesystems/list
  - POST /nas_servers/create
  - PUT /nas_servers/active_directory/test_join
  - POST /floatingips/create
  - POST /floatingips/pair
  - POST /applications/shares/add
  - GET /applications/shares/list
  - POST /applications/objects/stores/create
  - POST /applications/objects/access_keys/create
  - PUT /applications/objects/stores/access_keys/add
  - GET /applications/objects/stores/list
generated: '2026-08-29'
method: generated
source: https://docs.onestor.com/books/rest-api/page/applications
---

# Provision NAS and object storage on StorONE S1

The S1 object graph enforces an order, and the API will not do it for you. Get it wrong and calls
fail with an undifferentiated `400`.

**NAS:** Pool → Application → Volume → FileSystem → NAS server (+ floating IP) → Share
**Object:** Pool → Application → Volume → FileSystem → Object store → Access key

S1 publishes the guided path itself — read it first:

```
GET /workflows/volume_provisioning/NAS
GET /workflows/volume_provisioning/Object
```

## The step people skip

```
POST /applications/filesystems/add
```

Its published description says it plainly: **"Create a file system on a volume (required for
NAS/Object)."** A bare volume cannot host a share or an object store. Then mount it:

```
PUT /applications/filesystems/mount     # mount unmounted file systems
GET /applications/filesystems/list
```

## NAS path

```
POST /nas_servers/create
PUT  /nas_servers/active_directory/test_join   # test the AD join before relying on it
POST /floatingips/create
POST /floatingips/pair                          # pair the VIP so the NAS survives node failover
POST /applications/shares/add                   # "Create a SMB or NFS shares inside a volume"
GET  /applications/shares/list
```

Pair a floating IP. Without one, clients are bound to a single node and lose the share on failover.
`test_join` exists precisely so you verify Active Directory before users hit an authentication wall
— use it.

## Object path

```
POST /applications/objects/stores/create        # on a specific volume
POST /applications/objects/access_keys/create   # <accessKey> --secretKey=... --role=<objectPermissions>
PUT  /applications/objects/stores/access_keys/add
GET  /applications/objects/stores/list
```

Access keys are created **system-wide first**, then attached to individual stores. The `role`
parameter carries the object permissions — this is the S3 authorization model, so an existing S3
client works against the store once the key is attached.

Capture the secret at creation. There is no operation that reads a secret back, and
`DELETE /applications/objects/access_keys/delete` is not reversible — a lost key is recreated, not
recovered.

## Housekeeping

```
POST /applications/filesystems/fstrim/run
GET  /applications/filesystems/fstrim/status
```

`fstrim` reclaims space on thin-provisioned file-system volumes. It is asynchronous and has its own
status call — one of the few operations on this API that does.

## Reversing this

Only the mapping-shaped and pairing-shaped steps reverse cleanly (`DELETE /floatingips/unpair`,
re-pair later). Shares, object stores, access keys, volumes and applications all delete without an
undo. Snapshot the volume first — see `storone-snapshot-and-restore`.
