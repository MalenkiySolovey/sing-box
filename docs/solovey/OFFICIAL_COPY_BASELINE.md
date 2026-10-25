# Official copy baseline

SOURCE_AUTHORITY=https://github.com/SagerNet/sing-box

COPY_ROLE=mirror/cache/development

Copy: https://github.com/MalenkiySolovey/sing-box. Local origin points here; upstream points to SagerNet/sing-box. Pristine mirror/v1.14.2 is exact upstream af6e64c3b69e6132ebaee0e1a3d24e93903f6709, tree d4edc37a0f8ea49f5307aefdbcaf67295c56c45c.

The unchanged accepted Panel Official graph consumes 8b914e4be5103d5f343dac94f26c3ebeeb226ffc, tree 4ca03e05f360b25c958c2bfb6b4e246569789558. Four existing, separately reproduced engine blockers require temporary patches on this selected 1.14.2 base. Removing them would undo accepted bounded cache state, intentional listener cleanup, Naive HTTP/3 parent cleanup and WebSocket validation. No Official source/version/module pin is changed in this correction. These are exceptions justified by regressions, not a new distribution.

| Existing logical commit | Required behavior | Why adapter-only is insufficient | Regression |
|---|---|---|---|
| 20fdbaa1cf7c7f9fd1c22116c63731d25916290d | Preserve bounded SSM cache, failed-load state and cleanup | SSM private cache and endpoint maps are engine-owned; the Panel cannot atomically snapshot or retire them. | ./service/ssmapi |
| afd12f10473eb4d2d1459897488deb623527782e | Accept intentional SSM HTTP listener closure | The internal HTTP Serve result must distinguish expected listener closure. | ./service/ssmapi |
| d0feb7cd0e3cfd46a2a90c06be901c6c87aaeed8 | Drain Naive HTTP/3 parents before UDP closure | The private QUIC accept loop and HTTP/3 server own accepted connections; an outer Panel wrapper cannot retire those parents. | ./protocol/naive/quic; PANEL: ./service/coreinboundcontrol -run Naive |
| 8b914e4be5103d5f343dac94f26c3ebeeb226ffc | Preserve literal WebSocket paths and bound early data | Only the engine transport can compare the raw request path and reject over-limit early data before handing off the connection. | ./transport/v2raywebsocket |

All four are ENGINE_PATCH_REQUIRED, with upstream contribution candidacy recorded separately. They remain LOCAL_REQUIRED until a reviewed SagerNet base passes the same regressions without them. No external upstream issue or PR was submitted. Existing commit messages/authorship and all upstream licenses are preserved; this repository-local rationale covers the historical accepted commits. Every future patch must use the complete problem/adapter/regression/base/status/removal commit-message discipline.

solovey/official-patches contains these four linear accepted commits plus one policy/CI-only commit. Historical Shtorm-derived R7 refs are migration evidence only. They must not be used for Extended builds after the dedicated copy is accepted. They are not rewritten or deleted.

Pristine refs accept reviewed upstream synchronization only, never patch PRs. Development PRs target solovey/official-patches. Upstream updates use the documented rebase/range-diff procedure, not repeated upstream merge commits.
