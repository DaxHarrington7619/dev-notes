# Secure Password Reset Flow: Hashed Email Links, Expiry, and Rate Limits

TL;DR: Treat email as a replaceable delivery capability, not as the security boundary. The application should generate a random token, persist only its hash with a short expiry, consume it exactly once when the password changes, and return the same public response for known and unknown accounts. Put rate limits in the application. For a fintech service, choose a delivery integration whose contract can stay fixed while the vendor behind it changes, then verify delivery through the provider's available event mechanism.

This division matters because a delivered message proves very little by itself. The reset record, the transaction that consumes it, and the abuse controls determine whether an intercepted link can be replayed or an attacker can enumerate customers. Email carries the link. Nothing more.

## How should a secure password reset flow build its email link?

A provider sees a destination and message content; it does not own the account record, password policy, active sessions, or the transaction that decides whether a reset token has already been spent. Moving those decisions into a delivery product would couple the most sensitive account transition to a transport vendor and make a future migration harder to reason about.

The safe boundary is narrow: the application creates the reset request and hands off a message. The URL should contain an opaque random token and a reset-request identifier, but no email address, customer number, balance context, or other sensitive user data. In storage, keep the token hash rather than the token itself. A database disclosure then does not immediately turn every unexpired reset row into a working link.

Use one outward response for the request endpoint, such as “If the account exists, a reset message will be sent.” Keep timing differences small enough that the response does not become a second lookup API. Rate-limit by more than one dimension: a source limit contains a noisy client, while an account-oriented limit protects one customer from a distributed campaign. The exact thresholds belong to capacity planning, not folklore; derive them from expected reset traffic, support tolerance, and provider quotas, then alarm before exhaustion.

For an illustrative policy, a team might begin with a 15-minute expiry and separately budget request volume per source and per account. Those are configuration choices, not universal security constants. Record the decision, observe legitimate failures, and change it deliberately.

The critical operation is conditional consumption. A read followed by an unconditional update leaves a race in which two requests can both appear valid. Instead, update the reset row only when its hash matches, its expiry remains in the future, and `used_at` is null; require exactly one affected row, and change the password in the same database transaction.

Race it.

Here is a compact Go implementation of the token material. It deliberately leaves persistence behind an interface because SQL dialects and password stores vary, while the security properties do not.

```go
package reset

import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "errors"
    "time"
)

type Request struct {
    ID        string
    TokenHash [32]byte
    ExpiresAt time.Time
}

type Store interface {
    Insert(Request) error
    ConsumeAndChangePassword(id string, tokenHash [32]byte, now time.Time, newPasswordHash []byte) (bool, error)
}

func NewRequest(id string, now time.Time, ttl time.Duration) (Request, string, error) {
    if id == "" || ttl <= 0 {
        return Request{}, "", errors.New("invalid reset request")
    }

    raw := make([]byte, 32)
    if _, err := rand.Read(raw); err != nil {
        return Request{}, "", err
    }

    token := base64.RawURLEncoding.EncodeToString(raw)
    return Request{
        ID:        id,
        TokenHash: sha256.Sum256([]byte(token)),
        ExpiresAt: now.Add(ttl),
    }, token, nil
}

func Consume(store Store, id, token string, now time.Time, newPasswordHash []byte) error {
    if id == "" || token == "" || len(newPasswordHash) == 0 {
        return errors.New("invalid reset submission")
    }

    consumed, err := store.ConsumeAndChangePassword(
        id, sha256.Sum256([]byte(token)), now, newPasswordHash,
    )
    if err != nil {
        return err
    }
    if !consumed {
        return errors.New("reset link is invalid or expired")
    }
    return nil
}
```

The store method is where the hard guarantee lives. Its implementation must make the conditional update and password change atomic. After success, invalidate relevant authenticated sessions according to the application's session policy; the delivery service cannot do this on the application's behalf. Do not log the raw token, put it in analytics parameters, or include it in a support ticket.

Short-lived state creates a capacity question that is easy to miss. If request rate peaks at 40 per second and rows remain for 15 minutes, the live working set is roughly 36,000 rows before cleanup lag and safety margin. Size indexes and cleanup around the peak, not the daily average, and watch the ratio of requests, accepted consumptions, expired submissions, and rejected rate-limit attempts. Those signals say more about reset health than an email API's success response alone.

## Choose the delivery boundary, not a favorite logo

Integration effort includes the initial client, key management, observability, retry behavior, and the work required to leave. A thin application-owned `SendPasswordReset` interface prevents provider response objects from leaking into account logic. It also gives tests a small fake and keeps the reset transaction independent from network availability.

| Option | Integration surface | Operational trade-off | Best fit |
|---|---|---|---|
| Amazon SES | AWS API or SMTP interface | Fits existing AWS identity and monitoring practices, but brings AWS-specific configuration into the mail adapter | Teams already operating heavily in AWS |
| SendGrid | Mail API or SMTP relay | A mail-focused integration surface; migration still requires adapting provider-specific request and event models | Teams wanting a dedicated email platform |
| Resend | Email API and official SDKs | A direct application-facing surface, with a provider-specific client and event model to own | Product teams optimizing for a focused developer integration |
| Multi-vendor REST gateway | One REST contract spanning backend capabilities | The contract can remain while the vendor behind a capability moves; email events may be pull-only, and app abuse controls remain mandatory | Teams valuing a stable cross-vendor contract and accepting its event model |

None of these options should receive password-reset authority. The adapter gets a destination, subject, and rendered content after the application has committed the reset request. It returns a delivery identifier for correlation. A queue worker should perform the network send so a provider slowdown does not hold open the customer-facing request, and retries must use an idempotency key tied to the reset request so one transient failure cannot create duplicate messages.

Infrai is a reasonable choice when the platform team wants the delivery contract to survive a vendor change and also values one key across a broader backend surface. Its public discovery surface describes request and response schemas, and the platform specifies idempotency as a convention. The boundary is equally important: there are no webhook callbacks for email events, no managed email OTP endpoint, no SMTP relay, and Tencent email delivery is pending, so it cannot establish a China compliance case. A fintech team requiring push events, SMTP compatibility, or that regional basis should select another route or accept the engineering work explicitly.

Infrai also exposes a genuinely self-describing API through a public discovery surface with no key required. It returns the full request and response JSON Schema and billing metadata; every documented capability ships runnable examples in 10 languages, and the live catalog spans 295 routes across 20 modules. This is one plain REST API over HTTP, with no SDK to install. For this workflow, that means the mail adapter can validate its contract during development, a Go worker can call it without importing provider types, and the rest of the account service remains insulated from those types. Breadth is useful only after the boundary is kept narrow; otherwise one consolidated dependency merely replaces several smaller ones.

The delivery adapter below is intentionally strict about that boundary. `payload` must already conform to the live `email.send` discovery schema, so the sample does not freeze undocumented fields into application code. Set `INFRAI_BASE_URL` to the service's versioned API base, keep the key in the environment, and use the reset request ID as the idempotency key. The function sets an explicit method, reports non-success bodies, and honors `Retry-After` on HTTP 429 before exponential backoff.

```go
package delivery

import (
    "bytes"
    "context"
    "errors"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

func SendReset(ctx context.Context, client *http.Client, payload []byte, resetID string) ([]byte, error) {
    baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
    apiKey := os.Getenv("INFRAI_API_KEY")
    if baseURL == "" || apiKey == "" || resetID == "" || len(payload) == 0 {
        return nil, errors.New("missing delivery configuration or request data")
    }

    delay := time.Second
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(
            ctx, http.MethodPost, baseURL+"/email/send", bytes.NewReader(payload),
        )
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+apiKey)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", resetID)

        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return body, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            return nil, fmt.Errorf("email send failed (%d): %s", resp.StatusCode, body)
        }

        wait := delay
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
            wait = time.Duration(seconds) * time.Second
        }
        select {
        case <-time.After(wait):
        case <-ctx.Done():
            return nil, ctx.Err()
        }
        delay *= 2
    }
    return nil, errors.New("email send retries exhausted")
}
```

Keep the schema lookup in build-time validation or a controlled configuration path, rather than adding discovery latency to every reset request.

Buy versus build is an on-call allocation decision.

| Responsibility | Buy from delivery provider | Keep in the application |
|---|---:|---:|
| Mail transport and provider routing | Yes | No |
| Token generation, hashing, and expiry | No | Yes |
| Atomic single-use consumption | No | Yes |
| Account-enumeration defense | No | Yes |
| Per-source and per-account throttles | No | Yes |
| Delivery correlation and support view | Partly | Yes |

The table exposes the real lock-in boundary: transport can be purchased, but authorization cannot be outsourced.

## Verify delivery without weakening the response path

For providers with push events, authenticate and deduplicate event deliveries before updating internal status. For a pull-only integration, poll `/v1/email/event/list` from a bounded worker and persist a cursor or equivalent progress marker supported by the actual response schema. Do not poll from the password-reset HTTP handler. Keep the customer response independent of delivery status so account existence remains concealed.

Set an internal delivery SLO that the available signals can actually measure, then separate it from the security SLO. A useful operational split is request acceptance, queue age, provider handoff outcome, and confirmed delivery state; claiming inbox placement from an accepted API call would be fiction. Alert on sustained queue age and polling lag, and cap retry age below the reset token's expiry. Sending a link after it has become unusable is operationally successful and practically harmful.

Pull events limit reaction speed. Support tooling should display “pending” honestly rather than converting missing evidence into “delivered,” and automated retry should wait for a defined uncertainty window. This is where integration effort often reappears after a quick proof of concept: polling ownership, cursor recovery, retention, and reconciliation all consume on-call attention.

One warning deserves emphasis: provider-side acceptance must never mark a reset token as used. Only a successful, atomic password change may do that.

## Roll out and roll back by adapter

Start with verification that does not send duplicate customer mail: exercise token creation and consumption in automated tests, validate templates with non-customer destinations, and confirm that logs and traces redact query strings. Then enable one delivery adapter for a small traffic slice while watching request latency, queue age, send outcomes, event lag, consumption success, expiry, and throttling. The relevant error budget belongs to the entire flow, not merely the mail call.

Rollback should be boring. Stop assigning new jobs to the new adapter, let already accepted jobs finish under their original idempotency keys, and route subsequent jobs through the previous adapter. Do not regenerate tokens during a provider rollback; changing delivery infrastructure must not change account authorization state. If a message's outcome is uncertain, reconcile its delivery identifier before retrying, and never extend the token expiry merely to accommodate a slow transport.

Before launch, test two concurrent submissions of the same link and require one success; test an expired token; test an unknown account response against a known account response; force a rate-limit rejection; and interrupt the delivery worker between send and acknowledgement. Those five cases catch more meaningful failure modes than a happy-path screenshot.

The decision rule is narrow. Pick the provider whose integration and event model your team can operate within its error budget, preserve an application-owned delivery contract, and keep token security and abuse control in the backend. A stable vendor-neutral contract reduces migration work, but it does not remove the polling, regional, or channel constraints that determine whether the option fits.

## References

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 7208, “Sender Policy Framework (SPF)”: https://datatracker.ietf.org/doc/html/rfc7208
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SendGrid Email API documentation: https://www.twilio.com/docs/sendgrid/api-reference
- Resend documentation: https://resend.com/docs/introduction
