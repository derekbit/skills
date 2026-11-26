# Longhorn component profiles

Reviewers read the section whose heading matches the repo name (falls back to `generic`). Keep one `## <repo>` heading per component, and keep each section short:
it is a map of hot spots, not documentation. Paths listed under "Not reviewed" are excluded from the
review diff by the pathspecs in SKILL.md step 3.

## longhorn-manager

Control plane: one pod per node (DaemonSet), plus CSI driver, webhooks, REST API, upgrade jobs. The
repo's `AGENTS.md` is the primary standard; it is provided from the base commit.

Hot spots:
- `controller/`: reconcilers. `volume_controller.go` (largest state machine: attach, detach,
  replica scheduling, auto-salvage, rebuild, expansion, migration), `engine_controller.go` (engine
  monitor, rebuild kick-off, snapshot/backup status), `instance_handler.go` (instance state
  machine shared by engine and replica), `node_controller.go` (disk and node conditions, eviction),
  `instance_manager_controller.go` (IM pod lifecycle and live upgrade). Ownership through
  `isResponsibleFor` and `isControllerResponsibleFor` in `controller_manager.go`.
- `datastore/`: cached vs uncached reads (`uncached.go`). A delete or purge decided from a cached
  read is a stale-cache candidate.
- `engineapi/`: the only place that talks to the data plane. API version gates
  (`CurrentInstanceManagerAPIVersion`, `MinInstanceManagerAPIVersion`, proxy API version, BIM API
  version) live here and must match what old peers support.
- `scheduler/`: replica placement. Soft and hard anti-affinity (node, zone, disk), disk tags,
  over-provisioning and minimal available percentage, v1 vs v2 disks.
- `webhook/resources/<name>/`: validators and mutators. Defaults set by a mutator apply only to new
  or updated objects; existing objects need an upgrade migration too.
- `upgrade/vXXXtoYYY/`: migrations. Must be idempotent and must handle objects created by any
  supported earlier version.
- `types/setting.go`: setting definitions. A change normally also touches the setting webhook or
  `controller/setting_controller.go`.
- `csi/`: CSI controller/node servers. Idempotency is a CSI requirement (the CO retries every call).
  Error codes (`codes.NotFound`, `codes.Aborted`, ...) drive kubelet and external-provisioner retries.
- `manager/`, `api/`: REST handlers. Input validation and RBAC.
- `k8s/pkg/apis/longhorn/v1beta2/`: CRD types. A field change without regenerated
  `k8s/crds.yaml` is caught by CI; a field change without an upgrade path is not.

Not reviewed: `vendor/`, `k8s/pkg/client/`, `**/zz_generated*.go`, `k8s/crds.yaml`,
`client/generated_*.go`, `go.sum`.

## longhorn-engine

v1 data engine: the engine (controller) process, replica processes, sync agent, and the CLI used by
longhorn-manager through the engine image.

Hot spots:
- `pkg/controller/`: `control.go` (replica set, modes RW/WO/ERR, revision counter, frontend
  lifecycle), `multi_writer_at.go` and `multi_unmapper_at.go` (fan-out of writes and unmaps, error
  handling that must push failed replicas to ERR before acknowledging), `rebuild.go`,
  `revert.go`, `range_lock.go`.
- `pkg/replica/`: the on-disk format (snapshot chain, volume head, `volume.meta`, revision counter
  file, sparse files and extents). Crash-order of metadata vs data writes.
- `pkg/sync/`: sync agent: replica rebuild, snapshot purge/hash, backup and restore.
- `pkg/backup/` and the `backupstore` dependency: backup block mapping, incremental restore.
- `pkg/frontend/`: tgt-blockdev, socket, rest frontends.
- `pkg/meta/version.go`: `CLIAPIVersion`, `ControllerAPIVersion` and their min versions. longhorn-
  manager drives older engine images through the CLI, so any CLI flag or output change must stay
  readable by the manager for every supported engine image.
- `pkg/qcow`, `pkg/backingfile`: backing image access.

Not reviewed: `vendor/`, `go.sum`. Python tests under `integration/` are reviewed.

## longhorn-instance-manager

gRPC servers that run engine/replica processes (v1) and SPDK instances (v2), plus the proxy that
forwards manager calls to engines.

Hot spots:
- `pkg/process/`: process manager for v1 engine/replica processes. Port allocation and release,
  process state transitions, log handling, kill/cleanup on every failure path.
- `pkg/instance/`: the unified instance service that dispatches to v1 process manager or v2 SPDK.
- `pkg/proxy/`: forwards snapshot, backup, restore, volume calls to engines. Backup target
  credentials pass through here as env (`env_allowlist.go`); never log them, and do not let a request
  inject arbitrary env vars into a child process.
- `pkg/disk/`: v2 disk service (SPDK bdevs, lvstores).
- `pkg/meta/version.go`: `InstanceManagerAPIVersion`, `InstanceManagerProxyAPIVersion` and mins.
  The manager gates behavior on them; a new RPC needs a version bump that the manager checks.
- Every gRPC request field is untrusted input to a privileged process (paths, names, URLs).

Not reviewed: `vendor/`, `go.sum`.

## longhorn-spdk-engine

v2 data engine logic on top of SPDK, used by instance-manager.

Hot spots:
- `pkg/spdk/engine.go`, `replica.go`, `enginefrontend.go`: engine (RAID1 bdev over replicas),
  replica (lvol chain in an lvstore), frontend (NVMe-oF/ublk exposure). State is split between SPDK's
  in-memory bdev graph and Longhorn's view; a restart or a failed JSON-RPC call midway leaves them out
  of sync unless the code reconciles from SPDK on the next pass.
- Rebuild: replica add with shallow copy and range copy, suspend/resume of I/O around RAID
  membership changes (`replica_add_suspend_test.go` shows the intended ordering).
- Snapshot, clone, restore, backup (`backup.go`, `restore.go`): lvol snapshot and clone lifetimes,
  xattrs used as metadata.
- Expansion (`expand*`): lvol resize vs RAID resize vs frontend resize order.
- Shard groups and EC (`backend_shardgroup.go`, erasure coding paths): stripe math and degraded
  reads.
- `pkg/spdk/server*.go`: gRPC surface for instance-manager.

Not reviewed: `vendor/`, `go.sum`.

## go-spdk-helper

Thin Go client for SPDK JSON-RPC plus host-side NVMe-oF/ublk initiator helpers.

Hot spots:
- `pkg/spdk/client/`: JSON-RPC wrappers (`BdevLvol*`, `BdevRaid*`, `NvmfSubsystem*`, EC). Request
  field names and optional fields must match the SPDK version Longhorn pins; omitting vs sending a
  zero value changes SPDK behavior.
- `pkg/jsonrpc/`: request IDs, timeouts, concurrent use of one connection, response matching.
- `pkg/initiator/`: `nvme`/sysfs discovery, connect/disconnect, device naming; runs in the host
  namespace. Retries, partial-connect cleanup, and parsing `nvme-cli` output across versions.

Not reviewed: `vendor/`, `go.sum`.

## longhorn-share-manager

NFS-Ganesha based RWX export for a Longhorn volume.

Hot spots:
- `pkg/server/`: share manager lifecycle, export create/remove, grace period, recovery backend
  (client state survives failover through longhorn-manager's `recovery_backend`).
- `pkg/volume/`, `pkg/crypto/`: mount, filesystem, encryption (passphrase handling).
- Stale NFS handles and client recovery after the share-manager pod moves.

Not reviewed: `vendor/`, `go.sum`.

## backing-image-manager

Per-disk manager for backing images: download, upload, export from volume, sync between disks.

Hot spots:
- `pkg/datasource/`: data sources (download URL, upload, export from volume, clone). URL and path
  handling is untrusted input.
- `pkg/manager/`, `pkg/sync/`: file states, checksum verification, partial files on failure,
  concurrency between sync senders and receivers.
- `pkg/crypto/`: encrypted backing images.
- API version (see longhorn-manager `engineapi/backing_image_manager.go`).

Not reviewed: `vendor/`, `go.sum`.

## go-common-libs

Shared utilities used by nearly every Longhorn Go component.

Hot spots:
- Any exported signature or behavior change ripples into manager, engine, instance-manager,
  share-manager, and BIM through Renovate bumps. A changed default, error type, or return shape is a
  `compatibility` finding unless every consumer is handled.
- `ns/`: namespace executor (runs commands in the host namespace). Argument handling, timeouts.
- `sys/`, `io/`, `exec/`, `proc/`, `nfs/`, `kubernetes/`: host interaction helpers.

Not reviewed: `vendor/`, `go.sum`.

## types

Protobuf definitions and generated Go/Python code shared by all gRPC peers.

Hot spots:
- `protobuf/**/*.proto`: wire compatibility. Never renumber, reuse, or retype a field or an enum
  value. New fields need a zero value that old peers interpret safely. Removing a field must
  `reserved` its number and name.
- Every proto change implies consumers in engine, instance-manager, spdk-engine, share-manager,
  BIM, and manager; check the `related-*.patch` files for them.

Not reviewed: `pkg/generated/`, `generated-py/`, `vendor/`, `go.sum`.

## backupstore

Backup target drivers (S3, NFS, CIFS, Azure Blob) and the block-level backup format.

Hot spots:
- Backup format and lock files: concurrent backup/restore/delete on one backup volume, lock
  expiry, partial uploads.
- Credentials handling and logging.
- Block deduplication and reference counting on delete (data loss if a shared block is removed).

Not reviewed: `vendor/`, `go.sum`.

## longhorn

Umbrella repo: Helm chart, deploy manifests, LEPs (`enhancements/`), issue tracker.

Hot spots:
- `chart/`: values defaults, templates, RBAC (`ClusterRole` widening), PSP/PSA, upgrade of existing
  installs, `values.yaml` vs `questions.yaml` vs `deploy/longhorn.yaml` consistency.
- `enhancements/`: LEPs are reviewed for completeness of upgrade, compatibility, and test plans.

`deploy/longhorn.yaml` is generated from the chart: review it only for consistency with the chart
change, not line by line.

## longhorn-ui

React UI. Review for API field usage that matches the manager REST API, setting value formats, and
destructive actions without confirmation. No Go-specific rules apply.

## generic

No profile for this repo. Apply the cross-component guide, and treat the repo's own `AGENTS.md`,
`CONTRIBUTING.md`, or `README.md` as the local standard.
