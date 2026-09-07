# Capacity Limits for Presigned URLs: Node.js, Express, React, PUT, and Retrieval

**Short answer:** A direct browser upload should move object bytes around the application tier, not through it, while the application keeps control of authorization, object names, size limits, and expiry. The practical design is a three-step protocol: React asks a Node.js/Express signing endpoint for one narrowly scoped presigned PUT URL, the browser uploads the exact bytes to object storage, and the application later issues a short-lived signed download URL for the private object. Treat those URLs as temporary credentials.

That conclusion carries a catch: removing the application from the data path reduces its bandwidth burden, but it also removes the convenient place where teams used to validate every byte. Capacity planning therefore moves from application throughput to signing rate, abandoned-object rate, browser concurrency, storage request rate, and cleanup lag. Don't call the migration finished when the happy-path PUT returns successfully.

## How should a React browser and Node.js Express service control presigned PUT uploads?

Start with ownership. React owns file selection, progress, cancellation, and the actual PUT. Express authenticates the user, decides whether an upload is allowed, allocates an opaque object key, and asks a storage-specific signer to authorize that key for a bounded period. Object storage accepts the bytes. Application metadata records when an upload becomes usable. A download request returns another expiring URL rather than making a private bucket public.

The browser-to-storage request is cross-origin in the normal deployment shape, so CORS is part of the protocol rather than a dashboard detail. The bucket's CORS policy must admit the browser origin, the PUT method, and every request header that the browser will send. The upload request must then match the signed request: method, key, and any signed headers cannot drift between signing and transfer. MDN's CORS guide explains why a browser may send a preflight before the transfer and why a successful command-line request does not establish that the browser path is valid.

Use a small state machine: `allocated`, `uploaded`, then `committed`. The presign response allocates a key but does not prove that an object exists. After PUT succeeds, React tells the application to commit that allocation; the application can verify the expected object metadata through its storage adapter before making the object visible to the rest of the system. This avoids a surprisingly expensive category error — treating issuance of a URL as completion of an upload.

The signing response needs little data: an upload URL, its expiry time, the allocated object key, and the headers that must accompany PUT. The client should send the file body directly, not wrap it in `multipart/form-data` unless the signed operation explicitly expects that encoding. Keep the private object key out of user control; derive it from server-side tenancy and a random identifier, then retain the original filename as separately escaped metadata.

A bounded failure exercise makes the invariant clearer. Suppose 2,000 clients obtain ten-minute upload URLs, 300 users close their tabs before PUT, 40 complete PUT but never call commit, and one deployment changes a signed content-type header. Those are scenario inputs for a load test, not reported production measurements. The first group creates unused allocations, the second can create unreferenced objects, and the deployment can turn an otherwise healthy storage path into rejected requests. The invariant is simple: URL issuance, byte transfer, and business-level commit are different events, and retries must preserve that distinction.

Short version: authorize once, transfer once, verify, then publish.

## The incident lesson is an accounting problem

The most useful incident review begins before an incident exists. I would put four ledgers on the whiteboard: URLs issued, PUTs accepted, uploads committed, and objects expired or deleted. If those counters cannot be reconciled by tenant and time window, the system has an unknown storage liability. I'm not sure what abandonment threshold is acceptable for your workload; upload size distribution, mobile usage, and retention policy would resolve that. The requirement is to measure the gap rather than assume it is small.

A 403 on the browser's preflight and a rejected PUT should not be collapsed into one generic “upload failed” event. Record the stage, a stable reason category, object-key prefix, expected size band, and correlation ID, while excluding the signed query string because it is credential material. On the client, retry network failures only within the URL's useful lifetime and request a fresh authorization after expiry. On the server, make allocation and commit idempotent so a double click or a delayed response does not create two visible records.

This is where capacity planning becomes concrete. Model peak signer requests per second separately from peak upload bytes per second. The signer is a small control-plane dependency and should have an SLO for successful, authorized URL issuance; object transfer is a data-plane dependency and needs latency and completion objectives split by size band. Watch commit lag as a distribution, not merely an average. A long tail tells you that clients are slow, disconnected, blocked by CORS, or abandoning work, and each explanation calls for a different response.

Budget storage for incomplete work as well as retained work. Lifecycle management can expire temporary objects under a dedicated prefix, but cleanup timing should not be the correctness mechanism: the application still needs explicit state and reconciliation. A periodic job can compare allocated keys with committed metadata, wait beyond the maximum upload authorization window, and then mark or remove abandoned allocations according to the retention policy. Lifecycle rules are the final safety net for material that escapes application cleanup.

No drama. Just ledgers.

For observability, emit one trace or correlated event sequence across allocation, browser result, commit, verification, and download authorization. Never log a complete signed URL. Alert on ratios — committed uploads divided by issued authorizations, rejected transfers divided by attempted transfers, and cleanup backlog divided by its normal baseline — because raw counts rise with legitimate traffic. An SLO breach should identify which plane failed before it pages the application team.

## Put policy before presigning

The preventive code path belongs at the signing boundary. The following Go example is intentionally a transport-neutral model of what a Node.js/Express handler should enforce; all storage-specific signing stays behind an adapter, because inventing a universal presigning algorithm would be wrong. The same JSON contract is straightforward for React to consume. The signer implementation must use the chosen storage system's documented signing mechanism.

```go
package upload

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "errors"
    "fmt"
    "path/filepath"
    "strings"
    "time"
)

type Request struct {
    TenantID   string
    Filename   string
    ContentType string
    SizeBytes  int64
}

type Grant struct {
    ObjectKey string            `json:"objectKey"`
    UploadURL string            `json:"uploadUrl"`
    Headers   map[string]string `json:"headers"`
    ExpiresAt time.Time         `json:"expiresAt"`
}

type Signer interface {
    SignPut(ctx context.Context, key, contentType string, expiresAt time.Time) (string, map[string]string, error)
}

type Service struct {
    Signer   Signer
    MaxBytes int64
    TTL      time.Duration
    Now      func() time.Time
}

func (s Service) Authorize(ctx context.Context, r Request) (Grant, error) {
    if r.TenantID == "" {
        return Grant{}, errors.New("authenticated tenant required")
    }
    if r.SizeBytes <= 0 || r.SizeBytes > s.MaxBytes {
        return Grant{}, errors.New("file size outside policy")
    }
    if !allowedType(r.ContentType) {
        return Grant{}, errors.New("content type outside policy")
    }

    id, err := randomID()
    if err != nil {
        return Grant{}, fmt.Errorf("allocate object id: %w", err)
    }
    ext := strings.ToLower(filepath.Ext(filepath.Base(r.Filename)))
    key := fmt.Sprintf("pending/%s/%s%s", r.TenantID, id, ext)
    expiresAt := s.Now().Add(s.TTL)
    url, headers, err := s.Signer.SignPut(ctx, key, r.ContentType, expiresAt)
    if err != nil {
        return Grant{}, fmt.Errorf("sign put: %w", err)
    }
    return Grant{ObjectKey: key, UploadURL: url, Headers: headers, ExpiresAt: expiresAt}, nil
}

func allowedType(v string) bool {
    switch v {
    case "image/jpeg", "image/png", "application/pdf":
        return true
    default:
        return false
    }
}

func randomID() (string, error) {
    b := make([]byte, 16)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}
```

The handler around this service should map authentication failures and policy rejections to stable client-visible categories, apply per-tenant rate limits, persist the allocation before returning the grant, and avoid returning internal signer errors. React uses the returned headers exactly as supplied, performs `fetch(uploadUrl, { method: "PUT", headers, body: file })` as an operation described here rather than a second language code sample, and calls the commit endpoint only after the PUT completes. The commit operation accepts the allocated key or, better, an allocation ID bound to the authenticated tenant; it does not accept an arbitrary bucket key.

Test the contract at three layers. Unit tests should reject oversized files, disallowed media types, blank tenants, and unsafe filename assumptions. An integration test should use the real signer adapter against a disposable private bucket and exercise preflight, PUT, commit, and signed download. A browser test should run from the actual application origin, because server-side HTTP clients do not enforce browser CORS behavior. Include expiry, cancellation, duplicate commit, stale authorization, and a file whose declared type does not match policy. The exact content-verification approach depends on the threat model; high-risk inputs may require quarantine and asynchronous inspection before publication.

## Choose the operating model, not the prettiest API

Direct upload is not suitable when the application must synchronously transform, scan, or reject the byte stream before storage accepts it, when clients cannot perform the required cross-origin flow, or when files are so small and infrequent that another distributed protocol adds more on-call burden than it removes. In those cases, proxying through the application can be the more legible design, provided its memory, timeout, and bandwidth limits are deliberately sized. For very large or unreliable transfers, use the storage system's documented resumable or multipart mechanism rather than pretending one PUT has unlimited reliability.

The buy-versus-build decision is mostly about operational ownership:

| Operating model | Team owns | Useful when | Principal cost |
| --- | --- | --- | --- |
| Application proxy | authorization and byte transfer | inspection must happen inline | application bandwidth and scaling |
| Managed object storage with direct upload | policy, signing, CORS, metadata, reconciliation | clients can transfer directly | provider dependency and control-plane integration |
| Self-hosted object storage with direct upload | all of the above plus storage durability and upgrades | infrastructure control is a hard requirement | on-call load and capacity engineering |

A managed service can reduce the amount of storage machinery a team operates, but it does not own application authorization or orphan reconciliation. Self-hosting can improve control and portability, but only if the team can staff durability testing, upgrade rehearsals, monitoring, and recovery. Stick with an application proxy when policy genuinely requires inline handling; choose direct upload when bypassing application bandwidth is worth the additional client and control-plane states. Your mileage may vary, especially if egress policy or data residency dominates the decision.

For private downloads, repeat the same separation of duties. The application authenticates and authorizes each logical download, then returns a short-lived URL scoped to the intended object. Do not reuse the upload grant, expose a bucket publicly, or store signed URLs as durable metadata. Cache policy deserves explicit thought: a URL expiry controls authorization at the storage edge, while already retrieved bytes may still exist in a browser, intermediary, or user device. If revocation must be immediate, a direct signed URL may not satisfy the requirement; route downloads through a control point that can enforce the stronger policy.

The final readiness review is boring by design. Confirm private-by-default access, least-scope grants, an allowlisted origin and method set, bounded file size, collision-resistant server-assigned keys, idempotent allocation and commit, URL redaction, reconciliation, lifecycle cleanup, browser-origin tests, and SLOs for both planes. Then load-test the signing path and the abandonment path, not only successful transfer throughput.

The architecture earns its keep when a failed tab, an expired URL, and a duplicated callback produce explainable state rather than mystery objects.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
