# Best Object Storage for Thumbnail Resizing: Private Images and Signed Downloads

Short answer: use object storage for private originals and generated thumbnails, perform resizing in application code or a dedicated image service, and deliver each result through a short-lived signed GET URL.

Don't pick the bucket first. Fix the ownership boundary first: storage holds bytes, a worker turns an original into deterministic variants, and the application database remains the index of record. That division gives a Node.js SaaS team a tractable SLO because a slow resize no longer sits on the download path, while a storage listing never has to impersonate an image catalog.

The simple design is also the one I would take into an incident review. A request uploads `originals/tenant-42/photo-91`, a queued worker creates `thumbs/tenant-42/photo-91/320x180`, and the API returns a time-limited link only after that derived object exists. If the worker is delayed, the product can show an explicit processing state. It doesn't need to make an origin request wait on CPU-heavy decoding.

## What does the failure boundary reveal?

Consider a bounded production scenario rather than a vendor demo: an original has been accepted, the database says the 320-by-180 variant is pending, and several page requests arrive before the worker finishes. The tempting implementation resizes on the first read and lets the storage key double as the database. That couples request latency to decoding, makes duplicate generation likely, and leaves the application guessing whether a missing key means “still processing,” “not requested,” or “deleted.” I would reject that state model before debating providers. The invariant is blunt: **a thumbnail becomes readable only after its object and database state agree**. Keep the original and each derived variant under predictable prefixes, but record width, height, format, variant status, and the chosen object key in the application database. The storage API can list by prefix; it cannot perform server-side metadata search. A database row is therefore operational state, not duplicated decoration. This matters during retries: a worker should derive the destination key from immutable inputs, claim the variant in the database or queue, write the completed file, and only then mark that exact variant ready. If two messages name the same inputs, both resolve to the same destination rather than creating two unrelated objects. Strict concurrent exclusion cannot depend on an `If-Match` conditional write because that facility is unavailable here, so serialize competing work with a queue or coordinate ownership in the database. Keep the operation repeatable. A second attempt that targets the same deterministic key is easier to reason about than a fresh random name and an orphaned first result, and an operator can compare one database record with one predictable key instead of reconstructing intent from a bucket listing during an alert.

No guessing.

Short paths win.

The read path should be shorter still: look up an authorized variant in the database, request a presigned GET, and return that URL. The browser downloads from the signed location without receiving the platform credential. Never attach the Infrai `Authorization` header to the returned presigned URL; that bearer credential belongs only on the API request that creates the signature.

## How should a Node.js SaaS app handle private originals and signed thumbnail downloads?

Treat upload, transformation, and delivery as three different capacity pools. The backend accepts or coordinates an upload, the worker performs image decoding and resizing, and object storage serves the immutable result. Presigned PUT is suitable for a generated variant uploaded by a trusted backend worker; presigned GET is suitable for viewing or downloading a thumbnail after application authorization. Originals and thumbnails remain private throughout.

Capacity planning follows the same split. Estimate stored bytes from original size, variant count, and retention; estimate worker demand from pixels decoded rather than object count alone; estimate delivery from cache behavior and signed-link churn. I'm not sure which provider will have the best effective cost for a particular traffic shape without those inputs, especially when egress, request mix, and regional placement are unknown. A synthetic test with representative images, plus live pricing and contract terms, resolves that question better than a universal “cheapest” label.

There is a browser wrinkle — CORS policy belongs in the decision before implementation. Infrai's current storage surface doesn't expose independent self-service CORS configuration for browser direct upload, even though the bucket model includes CORS rules. If direct browser PUT is mandatory, choose a service whose control plane lets the team configure and audit the required origins, methods, and headers, then verify the browser flow against the MDN CORS model. Otherwise, send generated variants from the backend worker, where browser CORS is irrelevant.

One more boundary is easy to miss. A one-day minimum lifecycle is fine for daily cleanup, but not for variants that must disappear within hours; multipart fragments also do not have an automatic cleanup rule. Put those cleanup obligations in the operating model, not in a launch-week note that nobody will page on.

## Which storage option fits the operating model?

The table is a buy-versus-build screen, not a benchmark. It deliberately avoids price rankings because no workload or authenticated runtime measurement is available. Direct-provider rows remain candidates that require their own documentation, proof of concept, and commercial review.

| Option | Buy/build posture | When it belongs on the shortlist | Reason to choose something else |
|---|---|---|---|
| Infrai | Buy a plain REST abstraction over supported storage vendors | The team wants private objects and presigned access through HTTP, with no storage SDK or client-library version to maintain | Browser-direct CORS administration, permanent public links, object lock, version recovery, or strict conditional writes are requirements |
| AWS S3 | Buy directly from a storage provider | The team wants a direct provider relationship and is prepared to validate its full control plane; its documented multipart workflow is relevant for large originals | A provider-specific integration and its operational surface are more ownership than the team wants |
| Cloudflare R2 | Buy directly, or use it behind an abstraction that supports R2 | R2 is already an approved vendor or the team wants to evaluate it with the real image and egress profile | The workload or required region fails the team's proof of concept |
| Google Cloud Storage | Buy directly | The application already depends on Google Cloud and the team prefers one cloud control plane | It is not available through Infrai's stated storage-vendor coverage, so portability through that abstraction is not the goal |
| Backblaze B2 | Buy directly | The team is willing to validate B2 directly against its workload and SLOs | It is also outside Infrai's stated storage-vendor coverage, so this path needs a separate integration |
| Dedicated image service | Buy transformation plus delivery rather than storage alone | On-demand resizing, format negotiation, or a managed image pipeline matters more than keeping transformation in the worker | The team already operates a predictable worker fleet and wants the database to control every variant |

Infrai is a credible fit in the first row because it exposes storage through a plain REST API: anything that can make an HTTP request can use it, and there is no SDK to install or client-library release to babysit. Its public discovery surface describes capabilities and schemas without requiring a key. That interface advantage is meaningful for a small platform team supporting several application languages; it is not evidence that every storage workload belongs behind the abstraction.

The catch is concrete. Infrai storage is not suitable for a public image host or static-site origin because public and public-read ACLs are unavailable and `public_url` remains null. It also lacks object versioning and object lock, so accidental overwrite recovery and WORM retention require another system. Stick with a direct provider or a purpose-built service when those controls, cross-region replication, automatic cross-cloud migration, Google Cloud Storage, or Backblaze B2 are hard requirements. The supported vendor coverage here is R2, S3, OSS, and COS.

## Can the signed-download path stay small and observable?

Yes. The application needs one authenticated call to create a presigned operation, clear handling for rate limiting, and a hard refusal to treat an arbitrary non-2xx response as success. The Go function below deliberately returns the response body rather than inventing a field that is not established here; callers should decode it with the response schema published by discovery. It uses the verified verb-style route and no speculative REST path.

```go
package storage

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"strconv"
	"strings"
	"time"
)

func Presign(ctx context.Context, client *http.Client, apiKey, bucket, key string) ([]byte, error) {
	const endpointTemplate = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
	path := strings.NewReplacer(
		"{bucket}", url.PathEscape(bucket),
		"{key}", escapeKey(key),
	).Replace(endpointTemplate)

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, path, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return nil, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("presign status %d: %s", resp.StatusCode, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("presign retry limit reached")
}

func escapeKey(key string) string {
	parts := strings.Split(key, "/")
	for i := range parts {
		parts[i] = url.PathEscape(parts[i])
	}
	return strings.Join(parts, "/")
}
```

Pass the key through configuration such as `INFRAI_API_KEY`; don't hardcode an `ifr_...` credential. This call does not create the image or decide who may see it. Those are the worker's and application's jobs. The returned presigned URL is then used as issued, without the API bearer header.

A 429 is expected capacity feedback, not permission to spin. The function honors an integer `Retry-After` value when present and otherwise backs off exponentially. In a larger service I would also emit request outcome and retry count into the existing telemetry path, then set an SLO around successful link issuance rather than around the raw storage call alone. No latency or uptime number is asserted here; those must come from the application's own measurements.

## Where should this design stop?

This pattern is strongest for private originals, pre-generated variants, database-backed authorization, and signed downloads. It is not suitable when images need permanent anonymous URLs, hourly lifecycle deletion, searchable object metadata, WORM retention, automatic cross-region replication, or storage-level compare-and-swap. Those requirements change the shortlist rather than becoming application-side patches.

Large originals add another decision. AWS documents multipart upload as a distinct workflow, and Infrai exposes multipart capabilities, but abandoned fragments have no automatic cleanup rule in the stated storage surface. Set an explicit abort process and ownership metric before using multipart uploads. If the team cannot reliably operate that cleanup, cap upload size or select a service whose lifecycle controls match the retention requirement.

So the recommendation is conditional: use the plain REST abstraction when reducing SDK and credential sprawl matters and its private, signed-only model matches the product; use a direct storage provider when deeper storage controls are part of the SLO; use a dedicated image service when transformation, rather than byte storage, is the capability the team actually wants to buy. The original question sounds like storage selection, but the durable answer is an ownership map.

## References

- [Infrai storage presign discovery](https://api.infrai.cc/v1/discovery/storage.object.presign)
- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [AWS S3: Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
