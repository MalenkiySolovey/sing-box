# Solovey pinned Shtorm integration

Baseline: Shtorm sing-box-extended v1.14.3-extended-2.7.3,
`d9442d14b19ddd02605bf3ee893a171947038e19`, including Official v1.14.3
`7054cac5124465ffdb145870d7c01885566711b1`.
The bounded SSM integration is adapted from Solovey's Official integration
`20fdbaa1`. Cleanup is registered with Shtorm's lifecycle Scope. This is a
patched Shtorm source integration, not byte-identical upstream or Official.

The SSM delta is confined to `service/ssmapi` and this source notice:

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

- `protocol/naive/inbound.go`: its existing reverse-order Scope already closes
  the HTTP/3 lifecycle before the listener destroys UDP. This ordering is retained
  without adding the old engine's Close method. TCP/HTTP and TLS owners stay intact.
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

This seam is consumed with Solovey's adapted sing-quic parent integration on
`75c3ac4fa12b4ded2dc01b392ed7d5eaf3ea6876`, selected quic-go
`v0.61.0-sing-box-mod.9`, and the explicit Shtorm root replacement graph.
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

WireGuard integration adapts the privileged IPC seam from the stable comparison
source `deposist/sing-box-extended@cc14a046be0002aafd0950d06ea5c4aeb319f4e6`.
Original GPL source notices are retained in the adapted tests. On the selected
Shtorm baseline, the seam uses the existing protocol bind lock and a transport
device lifetime lock. Reads/writes are rejected before start and after scope
closure. Concurrent close detaches the device once before unregistering callbacks
without holding that lock. Startup failure diagnostics do not expose IPC values.
WARP forwards the native AWG 3.1 `random_trailers` and `disable_cookies` flags.
Live userspace device tests cover peer read/add/remove, AWG flag readback,
concurrent IPC/close, repeated close and restart rejection. The Panel must retain
its runtime generation and redact key material before exposing IPC observations;
this seam does not introduce an independent user/device or secret owner.

Native manager hot lifecycle retains the existing Scope model. At the pinned
Shtorm baseline only the outbound manager can remove/replace active objects.
The inbound, endpoint and service managers reject duplicate active tags, expose
no Remove operation, and do not start dynamically constructed candidates. The
bounded integration adds Remove to those three interfaces and adapts their
managers to the same native outbound Scope pattern. It serializes lifecycle
stages and mutation, closes a failed candidate's private Scope, keeps the accepted
object published until startup succeeds, and detaches returned inventory slices.
CoreRuntime retains its lifecycle lease across mutations and fences generation
shutdown. Native Scope cleanup is the sole owner of failed candidate resources;
the consumer's legacy Close-based candidate workaround is not used for this ABI.
`adapter/hot_lifecycle_test.go` covers every failure stage, old-object retention,
exact cleanup, successful replacement/removal, duplicate construction rejection,
snapshot isolation and served TCP listener reuse for all three manager kinds.
The consuming Panel's existing authenticated QUIC/Naive lifecycle and SSM HTTP
tests exercise the same public runtime mutations without changing generation.

`service/api/server.go` retains native Scope cleanup but treats the TCP listener's
expected net.ErrClosed after http.Server.Close as normal. Other cleanup errors
still fail. The consuming runtime's actual private API handshake, stop/restart,
API-only degradation, log redaction and generation fencing tests cover this seam.

Mieru v3.33.0 exposes listener factories, but the upstream sing-box integration
left both unset and therefore ignored its ListenOptions at bind time. The bounded
adapter supplies those factories through the existing sing-box Listener. Mieru
still selects primary/range ports and its wildcard default when no address is
configured; sing-box supplies an explicitly configured address,
socket flags, interface and namespace. No second protocol or socket owner is
introduced. Mieru's Mux.Start waits for factories but returns success even when
one fails. The adapter captures that failure before declaring readiness and
registers unconditional server.Stop with the native Scope before starting.
Each successful listener also registers its existing listener.Close with Scope:
the Mieru startup barrier precedes UDP underlay registration, so immediate Stop
can otherwise miss the underlay and leave its socket open. Expected net.ErrClosed
from Mieru's concurrent cleanup is normal. Explicit/wildcard listener isolation,
range port mapping, failed startup and repeated scope cleanup are covered by
protocol/mieru tests;
the consuming Panel qualifies actual TCP/UDP traffic and its existing user,
runtime identity, counters and reload owners.

An empty Mieru user list represents an inactive inbound. The selected upstream
rejects it, which prevents the existing Users owner from disabling or revoking
its final enabled user. The adapter still validates transport, ports/ranges and
traffic patterns, but does not construct or start the Mieru API server when no
users remain. Native hot replacement retires the old scope and its connections;
there is no listener, anonymous access, substitute credential or fallback.
The same saved options and credentials can serve again after an explicit Users
re-enable. Engine tests prove omitted/empty lists keep TCP/UDP ports reusable,
repeatable cleanup and strict invalid-option rejection; the consuming Panel
must qualify final-user revocation and re-enable through its real save/runtime owners.

TrustTunnel Scope cleanup retains the existing HTTP/TLS/listener ownership. Its
HTTP server closes the served TCP listener before the shared listener cleanup;
the latter now treats each expected net.ErrClosed as normal while retaining
other TCP/UDP cleanup errors. Real TLS serving, TCP/UDP port reuse, partial dual
startup and repeated Scope close cover the correction. No new authenticated
parent-control capability is introduced. The consuming Panel separately tests
actual HTTP/2 and HTTP/3 traffic, authenticated Users isolation and revocation
through normal hot replacement.
