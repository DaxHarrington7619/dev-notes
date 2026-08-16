# Node.js SaaS rules for private presigned object storage

Short answer: for a Node.js SaaS handling user avatars, use private object storage with short-lived presigned upload and download URLs only after proving that browser CORS rules and the authorization path fit together; the durable design keeps bytes off the application servers while keeping the decision to issue a URL in the application.

That sounds modest because it is. Avatar storage is rarely a capacity-planning problem, but it can become an availability and privacy problem when a tiny file path is allowed to borrow the login API's latency budget, turn deployment nodes into stateful machines, or make a long-lived bearer URL the real access-control system. The useful decision is about operational boundaries, not a console screenshot.

## What should a Node.js SaaS require from private avatar upload and download URLs?

Start with a control plane and a data plane. The control plane authenticates the user, checks tenant and object ownership, chooses a server-generated key, applies size and media-type policy, and creates a narrowly scoped presigned operation. The data plane transfers the image directly between browser and object storage. The bucket remains private. A signed URL is a temporary capability, so it belongs in a response body and not in a database record, log line, analytics event, or HTML cache.

No public bucket.

The incident lesson in a design review is usually mundane: a team treats "private bucket" as the whole policy, then discovers that its browser cannot upload, its API is now relaying every image, or an authorization check occurs only when an object is created and not when it is read. Each choice works in isolation. The invariant is that every transfer needs a current authorization decision before the capability is minted, and every capability needs an expiry appropriate to the object and audience.

Keep the signing endpoint boring. It should accept an opaque avatar intent rather than a client-selected object path, reject unsupported types and sizes before it grants a URL, and record an audit event without recording the URL itself. For downloads, authenticate and authorize again, then mint a separate short-lived read URL. Don't turn the storage credential into a browser credential.

Here is the kind of policy boundary worth preserving in Go, even when the service that calls it is written in Node.js. The storage-specific signer sits behind this validation rather than defining the business rules itself.

```go
package avatarpolicy

import "fmt"

const maxAvatarBytes int64 = 2 << 20

type UploadIntent struct {
	TenantID    string
	UserID      string
	ContentType string
	Size        int64
}

func Validate(intent UploadIntent) error {
	if intent.TenantID == "" || intent.UserID == "" {
		return fmt.Errorf("avatar upload requires tenant and user identity")
	}
	if intent.Size <= 0 || intent.Size > maxAvatarBytes {
		return fmt.Errorf("avatar size must be within policy")
	}
	switch intent.ContentType {
	case "image/jpeg", "image/png", "image/webp":
		return nil
	default:
		return fmt.Errorf("avatar content type is not allowed")
	}
}
```

One small path. It is also the path that lets the platform team count attempted uploads, rejected policy decisions, issued write capabilities, and completed uploads as different signals. An SLO for URL issuance cannot prove that the browser completed a transfer; an object inventory or a completion event closes that gap. Alerting on all four as one metric creates noise, while separating them shows whether an incident is authorization, CORS, signing, or transfer behavior.

## The browser boundary is where otherwise good storage designs fail

CORS is a browser protocol decision, not an application-server setting. MDN describes CORS as an HTTP-header mechanism through which a server indicates that a browser may allow a web application from another origin to access the response. A browser upload may therefore cause the object endpoint to handle a preflight request before the upload itself. The allowed origin, method, request headers, and response headers must match what the browser actually uses.

This matters for direct uploads because the API cannot repair a rejected preflight after it has returned a valid URL. The verification sequence deserves more care than it usually gets: use the actual browser origin rather than a same-origin local shortcut, create an upload intent as an authenticated user, compare the exact request method and request headers against the storage CORS rule, run the preflight, upload an allowed image, inspect only the response headers the client is permitted to read, and then repeat the request with a disallowed origin and a disallowed header. Persist the object key only after a successful transfer; otherwise a retry can make a database row look like an uploaded avatar when no object exists. Then test a second user in the same tenant and a user from another tenant, because storage-key namespacing is not an authorization decision by itself. In deployment review, collect the browser-side failure reason, the signing decision, and the final object result under a correlation identifier that does not include the signed URL. This produces a traceable failure boundary without making the capability itself part of telemetry. Keep development origins explicit instead of broadening the production rule until the test passes.

The same boundary affects private reads. A presigned URL moves object bytes outside the API process, which protects the API's concurrency budget, but it does not replace application authorization. The URL holder can use it until expiry. Cache policy, referrer handling, client telemetry, and expiration are therefore part of the security design, not cleanup work for a later sprint.

## Buy, build, or keep the bytes on the app nodes?

For avatars, write the capacity estimate down before choosing an architecture: expected users, retained versions per user, upper-bound image size after processing, peak upload rate, read-to-write ratio, and the retention period. The purpose is not a false-precise forecast. It identifies which component owns durability, backup verification, regional placement, and the on-call response when an object is unavailable.

| Approach | Operational owner | Main advantage | Boundary to accept |
| --- | --- | --- | --- |
| Managed object storage | Storage service and platform team | Direct browser transfer with a standard object interface | Credential and policy configuration still need review |
| Self-hosted object storage | Your platform team | Placement and operational control | Disks, replication, upgrades, backup recovery, and paging are yours |
| Files on application nodes | Application team | Minimal first deployment | Horizontal scaling and replacement turn into data-management work |
| API-proxied reads and writes | Application team | Per-request authorization can be centralized | Object traffic competes with application latency and egress capacity |

The catch is that direct presigned delivery is not suitable when a read must be revoked immediately for one recipient, or when a policy must be evaluated continuously while bytes are flowing. In those cases, keep the object behind an authorizing proxy or another access layer that can enforce the required control, and explicitly budget for its throughput and failure domain. Public avatars are a different case: if they are intentionally public, a private-object design adds ceremony without providing privacy.

Self-hosting can be the right call for a hard placement requirement or for a team that already operates storage as a discipline. It is not a free simplification. A small product team should compare the recurring work of recovery drills and capacity headroom with the lock-in and operational constraints of a managed service, then choose the failure mode it can actually staff.

## Multipart is a scaling tool, not a default for avatars

Amazon S3's multipart-upload documentation describes an object upload as initiated, uploaded in parts, and completed, with the service assembling the parts after completion. That workflow solves a real problem for large objects and interrupted transfers, because parts can be retried independently and an application can recover from a partial transfer with more control.

For a bounded avatar policy, those extra states also create operational work: track upload IDs, decide when to abort abandoned uploads, reconcile completed objects with application records, and test failure paths between completion and metadata persistence. Use multipart when expected object size, unreliable networks, or transfer duration justify it. For small avatars, a single signed upload has fewer states to observe and fewer cleanup rules to get wrong.

This is where a deployment test earns its keep. Run an end-to-end test from a browser origin that is different from the API origin; create an upload URL, upload an allowed image, persist the resulting key only after the transfer succeeds, obtain a read URL as the permitted user, and confirm that an unauthorized user cannot obtain one. Add expiry and abandoned-transfer cases. The test is more valuable than a unit test that only proves a URL-shaped string was returned.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
