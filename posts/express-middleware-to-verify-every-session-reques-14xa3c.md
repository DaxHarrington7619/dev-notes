# Express Middleware to Verify Every Session Request with Bounded Cache

TL;DR: In Express, put asynchronous session middleware before every authenticated route, look up an opaque session identifier in an authoritative server-side store, and attach identity only after the record is active and unexpired. A short cache may absorb repeated reads, but account deletion and global sign-out must invalidate both the source record and every cache entry tied to that account. Cache misses and backend errors are different states; treating an error as an authenticated cache miss turns an availability shortcut into an authorization failure.

For a media service, the difficult request is not a normal article-page load. It is account erasure while browsers, mobile clients, scrapers, and automated login attempts may still present old credentials. The invariant is blunt: once erasure enters its revocation phase, no surviving session may authorize another request. The middleware therefore has to make a fresh authorization decision on every request, even when a bounded cache makes most of those decisions cheap.

## What must the middleware prove on every request?

The cookie proves possession of a session identifier; it does not prove that the session is still valid. The middleware must extract the identifier, avoid accepting identity data supplied by the client, resolve the server-side session, check expiry and revocation state, and only then place the account identifier into request context. In Express terms, this is an async `req, res, next` boundary registered before protected routers. Missing or invalid credentials should produce an unauthenticated response, while a dependency failure should follow an explicit fail-closed policy rather than quietly calling `next()`.

That distinction matters during deletion. Model erasure as an ordered control-plane operation: mark the account unable to authorize, revoke all sessions indexed by its stable account ID, evict their cached representations, and then continue the data-erasure workflow. Do not search only for the cookie used to submit the deletion request. A person may have ten legitimate devices, and an attacker may hold an eleventh identifier that the person cannot see.

The abuse case changes the cache design too. Negative caching can reduce repeated work for random identifiers, but it can also become attacker-controlled memory growth. Bound its key count, use a much shorter lifetime than the positive cache, and avoid exposing whether a guessed identifier once existed. Rate limits belong at the edge and at credential-establishing endpoints; session validation still has to reject each invalid identifier independently. OWASP's authentication guidance recommends generic error responses because response differences can disclose account validity, and the same discipline is useful around session failures.

No shortcut here.

Prove it.

## The incident model that sets the invariant

Consider a bounded production failure rather than an invented success story. An erasure request revokes every row in the primary session store, but an application process retains a previously validated session in a 60-second local cache. During that minute, the old browser can still fetch subscriber-only media because the data plane never observes the control-plane write. The database is correct and the user-visible promise is still broken. This is the incident I would use in a review because it exposes the actual contract without claiming a benchmark or a lived outage: revocation latency is part of the authorization SLO.

A local cache is especially awkward across many application instances. Broadcasting invalidations can close the gap, but delivery, reconnect, and process-start behavior become security-sensitive. A shared cache makes invalidation easier to coordinate, although it adds another network dependency. Direct authoritative reads have the clearest revocation semantics and the highest read load. The right choice follows from a measurable revocation objective, not from the appeal of a tiny middleware function.

| Approach | Revocation behavior | Abuse and failure concern | Operational cost |
|---|---|---|---|
| Direct authoritative lookup | Observes committed state on the next successful read | Session-store exhaustion can affect every authenticated request | High read capacity; simple invalidation model |
| Per-process short cache | Stale until expiry unless every process receives invalidation | Random IDs can churn memory; lost invalidations extend access | Low network read rate; harder on-call diagnosis |
| Shared bounded cache | Central eviction can shorten propagation | Cache outage and hot-key behavior require explicit policy | Extra service and paging surface |

Capacity planning should start with authenticated request rate, cache hit ratio, record size, and the allowed revocation window. At 20,000 authenticated requests per second, even a modest miss ratio is material to the session store; that number is an example input, not a claimed measurement. Set a positive TTL from the revocation SLO, then load-test the resulting miss traffic. If the business requires near-immediate account deletion, a 60-second TTL is already disqualified unless invalidation is demonstrably reliable.

My first capacity decision would be direct reads, not a cache, until measurements reject that design. The trade-off is easy to state and harder to operate: caching reduces repeated reads but creates a period in which a decision can be stale, while direct reads preserve a single decision point but put peak traffic and abusive random identifiers against the authoritative dependency. To test the boundary, create 11 sessions for one synthetic account, warm several of them on every application instance, begin a continuous replay at a controlled rate, and commit the deletion while one cache consumer is disconnected from invalidation delivery. Record the last accepted request on each instance and compare it with the committed revocation timestamp. Repeat with the authoritative store unavailable, with an expired cache entry, and after a cold process restart. This is not a benchmark result; it is a test shape that forces the system to reveal its real revocation window. If any cached credential survives beyond the stated objective, shorten the TTL, repair invalidation, or remove the cache. Do not average away the slowest instance because one lingering process is enough to violate the deletion promise.

## A preventative request path

The following Go example makes the state machine explicit even though an Express implementation uses the same `request -> lookup -> attach identity -> next handler` sequence. It uses a 15-second positive cache as a visible policy choice, never as a universal recommendation. The cache stores an account ID and absolute session expiry, while the authoritative store remains responsible for active versus revoked state.

```go
package sessionauth

import (
    "context"
    "errors"
    "net/http"
    "time"
)

type Session struct {
    ID        string
    AccountID string
    ExpiresAt time.Time
    Active    bool
}

type Store interface {
    FindActive(ctx context.Context, id string, now time.Time) (Session, error)
}

type Cache interface {
    Get(id string, now time.Time) (Session, bool)
    Put(session Session, validUntil time.Time)
    Delete(id string)
}

var ErrNotFound = errors.New("session not found")

type accountKey struct{}

func VerifyEveryRequest(store Store, cache Cache, positiveTTL time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            cookie, err := r.Cookie("session_id")
            if err != nil || cookie.Value == "" {
                http.Error(w, "authentication required", http.StatusUnauthorized)
                return
            }

            now := time.Now()
            session, ok := cache.Get(cookie.Value, now)
            if !ok {
                session, err = store.FindActive(r.Context(), cookie.Value, now)
                if errors.Is(err, ErrNotFound) {
                    http.Error(w, "authentication required", http.StatusUnauthorized)
                    return
                }
                if err != nil {
                    http.Error(w, "authentication unavailable", http.StatusServiceUnavailable)
                    return
                }
                if !session.Active || !now.Before(session.ExpiresAt) {
                    http.Error(w, "authentication required", http.StatusUnauthorized)
                    return
                }

                validUntil := now.Add(positiveTTL)
                if session.ExpiresAt.Before(validUntil) {
                    validUntil = session.ExpiresAt
                }
                cache.Put(session, validUntil)
            }

            if !session.Active || !now.Before(session.ExpiresAt) {
                cache.Delete(cookie.Value)
                http.Error(w, "authentication required", http.StatusUnauthorized)
                return
            }

            ctx := context.WithValue(r.Context(), accountKey{}, session.AccountID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

In Express, the equivalent adapter should pass dependency errors to centralized error handling or return a deliberate 503, return 401 for absent or inactive sessions, and set something like `req.auth.accountId` only after validation. Keep route handlers ignorant of cookies. That ownership boundary prevents one new endpoint from bypassing verification because its author forgot a helper call.

The deletion transaction needs a second index from account ID to session IDs. Without it, global revocation becomes a scan or an unreliable list assembled from client-visible devices. After the authoritative revocation commits, publish invalidations keyed by the affected session IDs or by an account generation. Generation checks are attractive when an account has many sessions: increment the account's generation during deletion and require cached sessions to carry the current value. They still need a trustworthy way to observe the increment within the revocation objective.

## Operate the revocation promise, not just the cache

Measure the decision path separately: cache hits, authoritative hits, invalid identifiers, expired sessions, store errors, and validation latency. Do not put raw session IDs in logs or metric labels. A hash may still be sensitive and can create unbounded cardinality, so use request correlation and coarse reason codes for routine telemetry. Audit records for account erasure should record the account, operation, outcome, and timing under the service's privacy and retention rules.

The most useful SLO is end to end: from a committed delete-or-revoke action until an old session is denied across all serving instances. Test it by creating several sessions, warming caches on different instances, invoking deletion, and replaying every credential until each receives 401. Include cache loss, session-store timeout, duplicate invalidation, delayed invalidation, and application restart. A unit test that mocks one successful lookup does not exercise the promise.

Rollout deserves equal care. Start with shadow comparison between cached decisions and authoritative decisions, but never use shadow acceptance to grant access. Alert on disagreements, then canary enforcement while watching authorization errors and dependency saturation. Maintain a kill switch that disables caching and returns to authoritative reads; capacity must be reserved for that mode, or the escape hatch exists only on paper.

**The decision rule is to spend cache complexity only when direct reads cannot meet the capacity target, then cap staleness at the revocation SLO and prove invalidation under failure.** A small team with moderate traffic may rationally choose authoritative reads because the reduced on-call surface outweighs infrastructure savings. A larger platform may buy a managed cache or operate one itself, but the security contract stays with the application team.

| Decision | Managed service | Self-hosted component | Direct store reads |
|---|---|---|---|
| On-call load | Less component maintenance, dependency incidents remain | Full upgrade, scaling, and recovery ownership | Concentrated in the session store |
| Lock-in | Service APIs and operational behavior may couple the design | Greater portability with more labor | Tied to the chosen authoritative store |
| Revocation proof | Requires tested invalidation semantics | Requires tested topology and delivery | Simplest path after a committed write |
| Abuse controls | Quotas and eviction behavior must be verified | Limits are configurable and must be operated | Store protection and admission control are mandatory |

Limitations and trade-offs are explicit here. This design is not suitable unchanged for stateless bearer tokens that a service validates without a server-side session record. Immediate global revocation then requires another mechanism, such as a denylist, short token lifetime, or an account-level version checked online; adding that online check recreates part of the session architecture. It also does not replace reauthentication for sensitive actions. OWASP recommends reauthentication after risk events, and account deletion is a sensible place to demand stronger proof before starting an irreversible workflow.

## Sources

References:

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
