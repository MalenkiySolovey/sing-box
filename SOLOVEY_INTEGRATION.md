# Solovey pinned sing-box integration

Baseline: official SagerNet/sing-box v1.14.2,
`af6e64c3b69e6132ebaee0e1a3d24e93903f6709`.
The Go module declaration, protocol services and dependency requirements remain
those of that exact official baseline. This is a patched source integration,
not byte-identical official sing-box v1.14.2.

The delta is confined to `service/ssmapi`, `protocol/naive` and this source notice:

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

The Naive HTTP/3 addition repairs independently reproduced authenticated parent
connections remaining open after normal removal, hot replacement and Box shutdown:

- `protocol/naive/inbound.go`: close HTTP/3 lifecycle before the enclosing listener
  destroys UDP. TCP/HTTP handler and TLS cleanup retain their existing owners.
- `protocol/naive/quic/inbound_init.go`: preserve official TLS, congestion and HTTP
  semantics; validate congestion before allocating UDP, and hand accepted parents
  to the already pinned sing-quic transport owner. Serve each with the existing
  public HTTP/3 `ServeQUICConn` interface. An accept/handler goroutine does not close
  shared UDP ahead of the lifecycle owner.
- `protocol/naive/quic/inbound_lifecycle.go`: close once, fence acceptance, drain
  exact live/queued transport handles through the existing two-second parent
  contract, then bound HTTP/3 handler cleanup separately to two seconds. Errors
  and deadlines remain failures. QUIC's emission attempt cannot guarantee delivery
  across a broken network.

This seam is consumed with Solovey's paired sing-quic integration
`b2eea8ca8762dd1e438171e2f7b56f9715c30cbb`, selected quic-go
`v0.61.0-sing-box-mod.7`, and the unchanged official dependency requirements.
Build/test it under the consuming Solovey module's pinned graph. It introduces
no second transport inventory: the same per-inbound parent owner is reused.
Naive authentication remains per HTTP request and may use different principals
on one HTTP/3 parent. Its private lifecycle handle type exposes no authenticated
selective parent control, DB identity, address lookup or new product capability.
The existing CoreRuntime flow-only/session authorization contract remains intact.
Consumer integration tests exercise actual authenticated CONNECT/nonce, removal,
same-port replacement, shutdown/restart, UDP reuse and a shared parent carrying
two principals whose individual flow close must leave the parent usable.

Solovey's deployment adapter independently owns anchored private data paths,
POSIX ownership/mode and Windows DACL checks, atomic publication, read-only
diagnostics and its existing private backup contributor. It does not replace
SSM authentication, StatsTracker, panel clients or panel quota authority.

Source and original copyright notices remain under GPL-3.0-or-later. These
integration additions are also GPL-3.0-or-later. The consuming repository records
the exact source commit, Go module checksums, complete replayable patch, package
build metadata and SBOM module replacement. No module cache is modified.

Additional bounded WebSocket transport correction: the official v1.14.2 server
compared a decoded literal path against RequestURI when decoding path early
data, including when early data was disabled. Literal question marks/percent
escapes failed even when the native client sent the correct path. Oversized
early data also bypassed the configured maximum before transport admission.
`transport/v2raywebsocket/server.go` now uses URL.Path for the suffix, decodes
only when enabled, and rejects oversized data before the handler. Existing
protocol authentication and runtime/session ownership remain unchanged.
`server_path_test.go` exercises disabled/path/header modes with two literal
paths and oversized-header rejection through actual loopback transports.
Solovey separately adapts server-to-client path and public ECH serialization;
those changes belong to its existing entity/subscription owners.
