# SaaS Document Downloads: Private Object Storage and Signed Links

Use private object storage with application-side authorization and short-lived signed download links for ordinary SaaS documents; choose a proxy only when every download needs a live policy decision. The lowest storage rate is a poor selection rule, because egress, request volume, recovery objectives, and the operator's pager are part of the same system.

The useful boundary is plain: the application decides who may see a document, while the storage data plane delivers bytes after it receives a narrowly scoped, expiring capability. This keeps large transfers out of application workers without mistaking a URL for an authorization system. It also gives an incident commander somewhere concrete to look: authentication and document ownership in one trace, URL issuance in another, and transfer behavior in the storage and browser telemetry. Teams that collapse those signals into a single "download failed" counter usually make the wrong capacity or policy change first, then spend the next deploy unwinding it.

The boundary matters.

## What caused the private-file download incident?

Consider a bounded production scenario: a SaaS service stores tenant documents privately, then ships a new browser download flow shortly before a large customer export. The authorization check passes, yet the browser cannot fetch the generated URL from the application's origin. Meanwhile, an operator sees a rising retry count and starts treating it as a storage-capacity problem.

It may be neither. Browser cross-origin rules are a separate control plane from the URL signature. MDN documents that a browser can require a CORS response permitting the requesting origin, method, and headers, including a preflight exchange in applicable cases. A command-line request that obtains bytes does not prove that the browser path will work. Test the real application origin, the real request method, and staging too.

The invariant from this class of incident is that authentication, authorization, capability issuance, browser policy, and byte delivery are different steps with different owners. Put them on one dashboard, but do not merge them into one diagnosis. A valid signature cannot repair a rejected CORS policy; a permissive CORS policy cannot authorize the wrong tenant.

## How should SaaS teams protect user documents, private files, and signed download links?

Treat the signed URL as a bearer capability. AWS describes a presigned URL as time-limited access created with the permissions of the principal that generated it. Anyone who obtains that capability may use it until it expires, subject to the storage service's rules. The handler must therefore authenticate the caller, authorize that caller against the document record, resolve an object key that belongs to the authorized tenant, and only then produce a URL for that exact key.

Keep the expiration aligned with the transfer you intend to allow. A few minutes can be reasonable for a small PDF and inadequate for a large export on an unreliable connection. Do not place signed URLs in analytics events, referrer-bearing pages, support tickets, or ordinary application logs. They are temporary credentials, not harmless identifiers.

The preventative path is small enough to review. The storage-specific signing implementation remains behind an interface; the handler exposes the security order and returns only a bounded capability.

```go
package documents

import (
	"context"
	"encoding/json"
	"net/http"
	"time"
)

type Authorizer interface {
	CanReadDocument(context.Context, string, string) (bool, error)
}

type ObjectKeyResolver interface {
	KeyForDocument(context.Context, string) (string, error)
}

type Signer interface {
	SignGet(context.Context, string, time.Duration) (string, error)
}

type DownloadHandler struct {
	Auth     Authorizer
	Keys     ObjectKeyResolver
	Signer   Signer
	UserID   func(*http.Request) string
}

func (h DownloadHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	documentID := r.PathValue("documentID")
	allowed, err := h.Auth.CanReadDocument(r.Context(), h.UserID(r), documentID)
	if err != nil || !allowed {
		http.NotFound(w, r)
		return
	}

	key, err := h.Keys.KeyForDocument(r.Context(), documentID)
	if err != nil {
		http.NotFound(w, r)
		return
	}
	url, err := h.Signer.SignGet(r.Context(), key, 5*time.Minute)
	if err != nil {
		http.Error(w, "download could not be issued", http.StatusServiceUnavailable)
		return
	}
	w.Header().Set("Cache-Control", "no-store")
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]any{"url": url, "expires_in": 300})
}
```

The example assumes `UserID` reads verified identity middleware, not a client-controlled header. Contract tests should cover cross-tenant denial, expiration around clock boundaries, an anonymous fetch, browser CORS behavior, and an assertion that log events never contain the returned URL. Short expirations do not revoke a URL already copied. Use a proxy when immediate revocation, per-request inspection, or transformation is a hard requirement.

## Capacity planning is the actual comparison

Before comparing object storage offerings, build a workload sheet with stored bytes by age, p50 and p95 object size, writes, reads, deletes, geographical distribution, expected egress, retention, and restore volume. Then run three models: a normal month, a customer export, and a reconnect-driven retry surge. An estimate based only on stored gigabytes is decorative accounting.

For each candidate, collect current terms on the same date and retain the assumptions with the architecture decision. Calculate storage, operations, delivered bytes, replication or lifecycle work, migration traffic, and the human cost of the path. The last item is uncomfortable because it resists a clean unit price, but it is still real: a self-operated data plane needs patching, capacity headroom, failure-domain design, restore drills, and named on-call ownership. Review the calculation at the same cadence as capacity planning, because a document corpus has an awkward shape: its retained bytes generally grow monotonically, while reads can arrive in sharp, tenant-specific bursts. A tenant exporting years of attachments is not a representative average request. Nor is a retrying browser population after a network interruption. The plan needs a peak transfer assumption, a concurrent-signing assumption, a backpressure decision, and enough observational data to distinguish an authorization denial from a slow object read. Without those, the comparison is a pricing worksheet pretending to be an operating model.

Model the tails.

The SLO should distinguish link issuance from byte transfer. Link issuance belongs to the application and signer path; transfer performance depends on the storage and network path. Define availability and latency targets for both, plus a recovery point objective and recovery time objective for document metadata and objects. Capacity is not just free disk. It includes enough headroom to restore, migrate, or survive a burst while the service is still meeting its objective.

| Delivery design | Works well when | The catch | Proof before rollout |
|---|---|---|---|
| Managed private storage plus direct signed URLs | Documents can leave the app after authorization | Signing, lifecycle, and portability still need testing | Tenant isolation, expiry, checksum, CORS, and restore tests |
| Application or edge proxy | Every request needs live inspection or policy | The team owns bandwidth, backpressure, latency, and availability for every byte | Load test at export traffic and an explicit data-plane SLO |
| Self-operated object storage | Control or locality requirements fund a storage practice | Repairs, upgrades, replication, and recovery move onto the team's pager | Failure-domain review, restore drills, and spare-capacity policy |

## When does direct object delivery stop being suitable?

Direct delivery is not suitable when a download must be revoked immediately after issuance, every byte requires a current entitlement check, content needs request-time transformation, or a compliance control requires application-level inspection. The catch is that a proxy can satisfy those requirements only by moving bandwidth, backpressure, latency, and availability for every delivered byte into infrastructure the team operates. Stick with a proxy when those controls are mandatory; otherwise, direct delivery keeps the application out of the bulk-data path.

No universal winner exists.

Self-operation has a similar boundary. It can be rational where data locality, sovereignty, or sustained scale justifies dedicated storage expertise. It is a poor bargain when a small team has no tested recovery process or cannot assign durable ownership for upgrades and repair. Buy-versus-build decisions should be recorded against SLOs and recovery objectives, not a feature checklist.

Portability also needs an exercise, not a promise. Write checksummed fixtures, read them through the user path, export object keys and metadata, import them into a second compatible implementation, and compare bytes alongside authorization records. Compatibility labels are a starting hypothesis; signing behavior, metadata, multipart uploads, CORS, lifecycle rules, and errors deserve contract tests. Measure migration time using realistic bandwidth and compare it to the stated recovery objective.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
