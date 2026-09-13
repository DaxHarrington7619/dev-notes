# Node.js Support Attachments: Private PDF/DOCX Delivery with Compatible Object Storage

Keep customer-support attachments private, authorize every download against a tenant-owned database record, and issue a short-lived signed URL only after that check. Treat the object key as an address, not as proof of ownership.

That rule keeps the access-control decision in application data while leaving byte delivery to object storage. It also gives the platform team a useful SLO boundary: authorization latency belongs to the API, while transfer latency and availability belong to the storage path.

The bucket is private.

## What should a Node.js document upload store for each tenant?

For a support ticket, create a document row with `tenant_id`, `ticket_id`, `document_id`, `object_key`, `content_type`, `size_bytes`, `checksum`, `original_filename`, and a state such as `pending`, `available`, or `quarantined`. The database row is the source of truth for ownership. Object metadata can repeat the tenant and user identifiers for investigation, but it is not an authorization query.

Use an opaque, predictable key that does not contain a customer email or other personal data: `tenants/t_42/tickets/k_981/attachments/d_7/source.docx`. A key prefix helps cleanup and reconciliation; it does not replace a row-level access check. Normalize the extension from an allowlist, and derive the content type from validated intake rather than trusting a browser header. PDF and DOCX are file classes, not permission classes. Keep the key boring. Future operators should be able to identify the tenant, ticket, and revision without decoding a user-controlled string, while the database should still decide whether anyone may read it.

The upload endpoint should take the authenticated tenant context from the session or token. It must never accept an arbitrary bucket name and then treat the submitted key as safe.

A 403 is the correct outcome for a cross-tenant request; a missing record can be a 404 if that is the API's deliberate anti-enumeration policy. Pick one policy and document it. I would test that choice with a tenant that knows a document ID but does not own it, because authorization leaks often hide in the distinction between a missing row and a forbidden row.

## How can Node.js issue signed download URLs for private PDF and DOCX objects?

The download flow is deliberately boring: load the document by `tenant_id` and `document_id`, verify that its state is `available`, verify the caller's ticket permission, then ask the storage adapter for a URL with a short expiry. Return that URL once; don't save it as the permanent document link. When it expires, repeat authorization and mint another one.

Here is a provider-neutral Go interface and handler shape. The adapter can target an S3-compatible endpoint, but the application never constructs a public URL or exposes storage credentials to a browser.

```go
package attachments

import (
    "context"
    "errors"
    "net/http"
    "time"
)

var ErrNotFound = errors.New("attachment not found")

type Store interface {
    Put(ctx context.Context, key, contentType string, body []byte, metadata map[string]string) error
    SignGet(ctx context.Context, key string, expires time.Duration) (string, error)
}

type Attachment struct {
    TenantID, DocumentID, ObjectKey, State string
}

func DownloadURL(ctx context.Context, store Store, tenantID, documentID string) (string, error) {
    attachment, err := loadAttachment(ctx, tenantID, documentID)
    if err != nil || attachment.State != "available" {
        return "", ErrNotFound
    }
    return store.SignGet(ctx, attachment.ObjectKey, 10*time.Minute)
}

func DownloadHandler(store Store) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        tenantID := authenticatedTenant(r.Context())
        documentID := r.PathValue("documentID")
        url, err := DownloadURL(r.Context(), store, tenantID, documentID)
        if errors.Is(err, ErrNotFound) {
            http.NotFound(w, r)
            return
        }
        if err != nil {
            http.Error(w, "download unavailable", http.StatusBadGateway)
            return
        }
        http.Redirect(w, r, url, http.StatusFound)
    }
}
```

The `loadAttachment` query must include tenant scope, and `authenticatedTenant` must come from verified identity context. Those functions stay outside the storage adapter: mixing them into signing code makes a future provider change an authorization change. I would rather review that boundary than a handler accepting `bucket`, `key`, and `user_id` from the request.

## Which failure modes should a private object-storage workflow expose?

Test the matrix, not just a successful upload. A second tenant must not obtain a URL for the first tenant's ticket; a signed URL must stop working after expiry; a replacement must not return the previous revision; and a `pending` or `quarantined` object must not redirect. Test both PDF and DOCX metadata, including a filename with path separators and a content type that disagrees with the extension.

The upload state machine should make partial work visible. Write `pending`, send the bytes, verify size and checksum, then mark `available`. A worker can move suspicious content to `quarantined`; the download path should treat that state as unavailable. Retries need an idempotency key derived from the logical document revision, otherwise a timeout can leave duplicate objects and an ambiguous database row. For example, if the client loses its connection after the storage service accepts the bytes but before the API commits the row, the retry must address the same revision rather than inventing a second attachment. A reconciliation job can then compare `pending` rows with object metadata, retain a matching object for a bounded period, and delete only after the database and audit log agree on its disposition. That sequence is slower to design than a direct upload followed by a success flag, but it gives an on-call engineer something precise to inspect during a partial failure: state, revision, checksum, and the last storage request ID.\n\nDo not let a successful `PUT` imply a user-visible document.

Observe separate measurements for authorization requests, signing requests, upload bytes, download bytes, and peak concurrent transfers. Alert on authorization failures by tenant, repeated checksum mismatches, stale `pending` rows, and a widening gap between database records and object listings. Capacity planning is not one number: stored bytes affect retention, transferred bytes affect egress, and concurrency affects connection pools and worker limits.

## How should teams compare storage controls, direct APIs, and self-hosting?

The buy-vs-build decision is mainly about controls and on-call ownership, not whether a URL can be signed. A managed object service usually reduces storage operations work; a direct integration can provide placement or governance that a generic adapter cannot. The contract should make that trade visible.

| Approach | Good fit | Not suitable when |
| --- | --- | --- |
| S3-compatible managed storage behind an adapter | Private objects, signed reads, and a familiar API contract are enough | Native retention, replication, or provider-specific governance is mandatory |
| Direct provider integration | Existing IAM, audit, lifecycle, and regional controls are hard requirements | Several backends must remain interchangeable without application changes |
| Self-hosted object storage | Data placement and operational control outweigh maintenance effort | The team cannot staff upgrades, disk failure response, and recovery tests |

The catch is that S3 compatibility does not mean identical behavior everywhere. Verify signed URL semantics, checksum handling, lifecycle rules, versioning, multipart cleanup, and policy evaluation against the actual target. Your mileage may vary on browser-direct uploads, especially when CORS administration and preflight behavior are owned outside the application team.

Keep the application-facing interface small: put an object, inspect its metadata, create a signed read, and delete or supersede a revision. If legal hold, WORM retention, automatic cross-region replication, or public edge caching is part of the SLO, make it an explicit requirement and choose a boundary that owns it. Don't hide it behind a generic `Store`.

## How do you verify and roll back tenant-scoped document delivery?

A release check should create two tenants, upload one PDF and one DOCX per tenant, request allowed and denied downloads, wait through signing expiry, and compare database state with object metadata. Run it in a disposable bucket with production-like policy evaluation. Log tenant ID, document ID, revision, authorization result, and storage request ID; never log the signed URL.

For replacements, prefer a new revision key and an atomic database pointer update. Roll back by pointing the row at the previous known-good key, then rerun the authorization smoke test. Overwrite-in-place removes that clean rollback path unless storage versioning is enabled and tested, so it is a poor default for records that matter to a support dispute.

This pattern fits ordinary private support documents where tenant isolation and simple delivery are the main axis. Choose a different storage design when the required SLO depends on immutable retention, provider-native governance, or browser-direct control that this boundary cannot guarantee.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://cloud.google.com/storage/docs
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://www.rfc-editor.org/rfc/rfc9111.html
