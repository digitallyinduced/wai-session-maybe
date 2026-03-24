# wai-session-maybe

A fork of [wai-session](https://github.com/singpolyma/wai-session) that changes the `SessionStore` type to return `Maybe ByteString` instead of `ByteString` for the cookie value. This allows session backends to signal "no change needed" by returning `Nothing`, causing `withSession` to skip the `Set-Cookie` header entirely.

## Why this exists

The upstream `wai-session` package has not been updated since 2018. We submitted [PR #17](https://github.com/singpolyma/wai-session/pull/17) with this change but the repository appears dormant. Since the type change is backwards-incompatible, we publish this as a separate package with distinct module names so both packages can coexist in a dependency tree.

## What changed

The `SessionStore` type alias:

```haskell
-- wai-session (original)
type SessionStore m k v = (Maybe ByteString -> IO (Session m k v, IO ByteString))

-- wai-session-maybe (this package)
type SessionStore m k v = (Maybe ByteString -> IO (Session m k v, IO (Maybe ByteString)))
```

When the `IO (Maybe ByteString)` action returns `Nothing`, `withSession` skips the `Set-Cookie` header on the response. When it returns `Just bytes`, the cookie is set as before.

## Performance impact

This change enables session backends (like [wai-session-clientsession-deferred](https://github.com/digitallyinduced/wai-session-clientsession-deferred)) to skip expensive crypto operations when the session is never accessed. In benchmarks with a minimal WAI app where the handler never touches the session:

| Scenario | Before (wai-session) | After (wai-session-maybe) | Improvement |
|---|---|---|---|
| No cookie (new visitor) | 43,858 req/s | 128,990 req/s | **2.9x throughput** |
| With cookie (returning user) | 39,533 req/s | 123,160 req/s | **3.1x throughput** |

The p50 latency dropped from ~2.3ms to ~0.4ms (**5-6x improvement**) because AES decryption and encryption are completely eliminated for routes that don't use the session.

## Module names

| wai-session | wai-session-maybe |
|---|---|
| `Network.Wai.Session` | `Network.Wai.Session.Maybe` |
| `Network.Wai.Session.Map` | `Network.Wai.Session.Maybe.Map` |

## Usage

```haskell
import Network.Wai.Session.Maybe (withSession, Session, SessionStore)
```

The API is otherwise identical to `wai-session`.

## License

ISC (same as the original wai-session).
