# mlx-run integration contract

This fork starts from `npanj/splash` Q8 commit
`10555a4f532217d46afdce8bd0624479bfed56c4`. It adds a bounded
session prompt cache and a private token estimate endpoint for mlx-run.

## Serving

`./splash serve --model nitinpanj/Qwen3.8-27B-Splash-HQ` binds only
`127.0.0.1:18080` by default. `--port` selects another loopback port and
bundled clients discover that port from the foreground server lock. An
occupied port fails before model work. `--prebuilt` verifies an existing
binary, Metal library, and model package without a build or download.

## Route contract

mlx-run calls `POST /internal/estimate` with
`{"protocol":"chat|responses|messages","request":<original body>}`. The
reply contains `input_tokens` and `output_tokens` from the same preparation
path used for generation. The estimate does not submit work to the engine.

The private loopback proxy alone adds `X-Splash-Session-ID` to generation
requests. The value must contain 1–128 ASCII letters, digits, underscores,
or hyphens. Missing header means no idle prompt retention. All three
generation protocols carry the mapped session identity through wire
protocol version 6 to the native engine.

After a successful generation, the native cache promotes only that request's
input prompt. For overlapping requests in one session, the highest admitted
request ID wins even if an older request completes later. Cancellation and
failure never promote a head. Up to 16 session heads and 8 GiB of shared
idle KV plus recurrent state are retained. The oldest completed session head
is evicted when either limit is exceeded; memory pressure may evict sooner.
Shared prefixes are stored once. Active request pages and pinned states are
never selected by session pruning. Only complete 32-token blocks can be
reused; an incomplete final prompt block is regenerated.

The 8 GiB bound applies to logical idle cache data, not active allocations
or the model weights. Reclaiming backing memory may lag logical eviction
while a Metal release is in flight.

## Verification

The launcher, Python wire, server preparation, native protocol, and native
cache suites cover port selection, exact estimates, session propagation,
completion order, no-session retention, the 16-head limit, and a reduced
byte-limit fixture. A live Metal check still needs a host with the Metal
Toolchain or a reviewed matching metallib and enough available memory to
load the target and draft beside any existing services.
