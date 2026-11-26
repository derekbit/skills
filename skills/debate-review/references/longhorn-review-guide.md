# Longhorn review guide (cross-component)

Every reviewer reads this before the diff. The component profile (`components.md`) and the repo's
`AGENTS.md` from the base commit (`AGENTS.base.md` in the run directory) complete it. When this
guide and `AGENTS.md` disagree, `AGENTS.md` wins.

## Order of work

Do not start line-by-line review before steps 1 to 4 are done (the order longhorn-manager's
`AGENTS.md` mandates; it applies to every Longhorn repo).

1. Read the PR title and description (`pr.md`).
2. Read the linked `longhorn/longhorn` issue (`issues.md`), the source of truth for intent, and the
   LEP (`lep.md`) if there is one.
3. Read cross-repo PRs this one depends on (`related-*.patch`). A manager change that consumes a new
   engine, instance-manager, or types API is only correct if both sides agree.
4. Read the changed files and their direct callers and callees to learn the local pattern.
5. Check the change against the issue intent, then do the detailed review.

## Severity (the Longhorn scale)

| Severity | Use for | Shown as |
| --- | --- | --- |
| `critical` | data loss or corruption, a write acknowledged but not persisted, split brain, a deadlock, a race that corrupts state, a security or credential exposure | P0 |
| `major` | a regression, an unintended behavior change, an unsafe state transition, a missing or dropped error, broken retry/idempotency, an upgrade or version-skew break, a leak in a long-lived process | P1 |
| `minor` | maintainability with a concrete cost: an inconsistent pattern that will mislead the next change, a misleading log or event, a test that does not pin the behavior | P2 |

`critical` and `major` are blocking. Informational items (style, naming, wording) are never emitted.

## Axes

Every finding carries one axis. Use the most specific one.

- `data-integrity`: anything that can lose, corrupt, duplicate, or silently diverge user data or
  replica state. Writes, snapshots, revision counters, rebuild, backup and restore, purge, expansion,
  trim/unmap, checksum, encryption.
- `concurrency`: races, lock ordering, goroutine or channel misuse, context propagation, stale
  informer cache reads leading to wrong writes, two owners acting on one object.
- `compatibility`: upgrade paths, CRD schema changes, conversion, version skew between manager and
  data-plane components (engine image, instance-manager, backing-image-manager, share-manager),
  protobuf wire compatibility, API version constants, settings defaults and value formats.
- `correctness`: logic bugs that are none of the above.
- `error-handling`: dropped errors, wrong retryable vs terminal handling, missing cleanup on a
  failure path, errors without actionable context.
- `resource`: anything that grows with uptime or with cluster size, repeated expensive calls in a
  hot reconcile path, missing timer/ticker/goroutine stop.
- `security`: secrets in logs or events or CR status, untrusted gRPC/HTTP input used in paths,
  commands, or namespaces, RBAC widening, host path exposure, TLS or auth weakening.
- `spec`: the issue or LEP asks for X and the PR does Y, does part of X, or does X plus Z.
- `scope`: unrelated refactors or behavior changes mixed into a fix.
- `standards`: a written rule in the repo's `AGENTS.md` (quote it). Not tooling-enforced rules.
- `tests`: a fix without a test that fails without it, a test that cannot fail, missing error path.
- `docs`: docs or comments that now contradict the code.

## Longhorn invariants to check

These are the invariants a Longhorn change most often breaks. For each changed hunk ask whether it
touches one of them, and if so, where the new code re-establishes it.

### Reconcile and ownership (longhorn-manager)

- `syncXxx(key)` must be idempotent and safe to re-run after a crash at any line. A side effect
  (instance start, snapshot create, backup create, engine API call) followed by a status update that
  can fail must not repeat the side effect on retry, or the repeat must be harmless.
- Objects from the informer cache are shared. Mutating one without `DeepCopy()` corrupts the cache.
- The cache can be stale. A decision that deletes, purges, or rebuilds based on a cached read needs
  either a fresh read (`datastore` uncached helpers) or a guard that makes the stale case safe.
- Only the owner reconciles. Changes around `isResponsibleFor`, `OwnerID`, `PreferredOwnerID`,
  node down detection, or controller handover can create two writers or zero writers.
- Spec and status are written separately (`UpdateXxx` vs `UpdateXxxStatus`). Writing status through
  the spec path or the reverse silently drops the change. A conflict (`apierrors.IsConflict`) must
  requeue, not be treated as success or as a terminal error.
- Retryable errors are returned to the workqueue; terminal ones are logged and `return nil`.
  `maxRetries` is small, so a transient error returned on a path that never recovers just drops the key.
- Conditions, event reasons, and messages are API. Tests and users match on them.
- The finalizer path must complete even when dependents are already gone, and must not delete data
  for an object that is only being re-created.

### Data path (engine, spdk-engine, instance-manager)

- A write is acknowledged only after every replica in RW mode has it. A replica that fails a write
  must leave RW (ERR) before the next write is acknowledged.
- Revision counter, snapshot chain metadata, and the volume head must stay consistent across crash
  points. Look at the order of metadata writes vs data writes.
- Rebuild must not let a WO replica serve reads, and must not mark it RW before its data is
  complete and verified.
- Snapshot delete, purge, and coalesce must not remove data a child, a backup in progress, or a
  rebuilding replica still depends on.
- Size math: sector alignment, 4 KiB vs 512 B, integer overflow on large volumes, expansion that
  changes size before the backend has grown.
- v1 and v2 data engines take different paths. A change guarded by data engine type must be right
  for both, or explicitly unreachable for one (`types.IsDataEngineV2` and friends).

### Version skew and upgrade

- During an upgrade, a new manager talks to old engine images, old instance-managers, and old
  backing-image-managers. API version constants (`CLIAPIVersion`, `ControllerAPIVersion`,
  `InstanceManagerAPIVersion`, `InstanceManagerProxyAPIVersion`, the BIM API version) and their min
  versions must move together with the code that relies on them, and the manager must keep handling
  the minimum supported version.
- Protobuf in `longhorn/types`: never renumber, reuse, or change the type of an existing field.
  New fields must have a safe zero value for old peers.
- A new or changed CRD field in a released API version needs defaulting for existing objects (an
  `upgrade/vXXXtoYYY` migration and/or the mutating webhook) and must keep old objects valid.
  Migrations must be idempotent because the upgrade can be interrupted and re-run.
- Settings: changing a default, a value format (including per-data-engine JSON values like
  `{"v1":"true","v2":"false"}`), or a name affects existing installs. Check the validator, the
  setting controller, and the upgrade path.

### Long-lived processes and resources

- Manager, instance-manager, and engine run for months. Anything keyed by dynamic data (maps,
  caches, metric label sets), un-stopped tickers, goroutines without a stop channel, and per-sync
  allocations that never free are leaks.
- gRPC clients and connections must be closed on every path, including errors.
- Timeouts on calls to the data plane. An unbounded call in a reconcile loop blocks the worker.

### Security

- Never log or put into events/status: backup target credentials, encryption passphrases, CSI
  secrets, tokens. Names, not values.
- Fields that come from a gRPC request, a CR spec, or an HTTP body and end up in a file path, a
  command line, an `nsenter` target, an env var for a child process, or a URL are untrusted input.
  Look for validation at the receiving side, not only in the webhook.
- Never write exploit steps; the report may be copied into a public PR. Name the class of problem
  and the location.

### Cheap checks on the changed paths

- `k8s/pkg/apis/longhorn/v1beta*/` changed but `upgrade/` or `webhook/` did not: do existing objects
  need a migration, a mutator default, or a validator rule?
- `types/setting.go` changed without the setting webhook or `controller/setting_controller.go`:
  is the new value validated, and what happens to existing installs?
- `go.mod`: a `github.com/longhorn/*` bump must match the consuming code; an added `replace` is
  usually a temporary cross-repo pin that must go before merge; a bump inside a `fix` PR that the fix
  does not need is a `scope` finding.
- `vendor/` changed without `go.mod` or `vendor/modules.txt`: vendored code edited by hand.
- A `.proto` file or an `*APIVersion*` / `*APIMinVersion*` constant changed: see "Version skew".
- A `fix` PR with no test change: `testing_status` must say so; a finding only if the fix is
  unit-testable.
- `Fixes #N` / `Closes #N` in the PR body: mention in `context_check` (the Longhorn template asks for
  `Issue #N` so QA can verify before close). Never a finding.

## Bar for a finding (both passes)

- One discrete defect at one location, introduced by this diff, or on an unchanged line of a
  function this diff touches, or in an unchanged caller broken by a contract this diff changed.
  Pre-existing defects elsewhere are out of scope, even when real.
- `evidence` names the trigger (input, state, timing, version skew, crash point) and the wrong
  result. If you cannot name the trigger, you do not have a finding.
- Nothing `gofmt`, `go vet`, `golangci-lint`, `codespell`, the import-order check, the Conventional
  Commits check, or `crds.yaml` freshness check already catches.
- No style, naming, or "consider extracting".
- A behavior change that the PR body or the linked issue states is intended is not a bug. A
  behavior change nobody stated is, and goes on the `scope` or `correctness` axis.
- Do not review `vendor/`, generated code, or `go.sum`. Do check that a dependency bump is the one
  the change needs and that the consuming code matches the new API.
