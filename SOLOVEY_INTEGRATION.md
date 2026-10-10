# Solovey pinned SSM integration

Baseline: Shtorm sing-box-extended v1.14.3-extended-2.7.3,
`d9442d14b19ddd02605bf3ee893a171947038e19`, including Official v1.14.3
`7054cac5124465ffdb145870d7c01885566711b1`.
The bounded SSM integration is adapted from Solovey's Official integration
`20fdbaa1`. Cleanup is registered with Shtorm's lifecycle Scope. This is a
patched Shtorm source integration, not byte-identical upstream or Official.

The delta is confined to `service/ssmapi` and this source notice:

- `cache.go`: bounded cache parsing rejects null roots/endpoints with errors
  that exclude input values, preserves unreadable/corrupt preimages, checks
  managed user restoration, and snapshots user/counter maps under their existing
  locks in the user-before-traffic lock order. Cache writes are serialized.
- `cache_store.go`: an instance-local context adapter exposes only bounded
  read/write storage. Official SSM continues to interpret authentication and
  counters. The portable default retains filemanager routing, writes new files
  with mode0600 and directories0700; deployment ownership and atomic publication
  belong to the injected storage adapter, not this portable default.
- `server.go`: failed cache loading rejects startup, failed startup does not
  overwrite cache, repeated Start/Close are idempotent, cancellation stops and
  drains the periodic writer before final flush. Close has one two-second
  context budget. Injected storage must honor cancellation before publication;
  blocked external filesystem calls cannot be forcibly interrupted by Go.
  HTTP shutdown may already close the served TCP listener; the listener owner
  still cleans up and treats only its expected `net.ErrClosed` as success.
- `cache_test.go`: bounded/corrupt/redacted inputs, counters/authentication
  restoration, concurrent update snapshots, missing/disabled optional cache,
  failed startup preimage, periodic drain, bounded concurrent close and close-once.
  A real served loopback listener proves normal closure, port reuse and repeated
  close without a spurious hot-removal failure.

Solovey's deployment adapter independently owns anchored private data paths,
POSIX ownership/mode and Windows DACL checks, atomic publication, read-only
diagnostics and its existing private backup contributor. It does not replace
SSM authentication, StatsTracker, panel clients or panel quota authority.

Source and original copyright notices remain under GPL-3.0-or-later. These
integration additions are also GPL-3.0-or-later. The consuming repository records
the exact source commit, Go module checksums, complete replayable patch, package
build metadata and SBOM module replacement. No module cache is modified.
