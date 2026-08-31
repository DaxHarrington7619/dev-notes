# Tenant-Isolated Object Storage for US/EU App Data Backup Retention

Short answer: for an e-commerce system storing training artifacts alongside app data backups, the decision rests on tenant isolation and a reproducible restore test; the cheapest archive is irrelevant if one tenant can enumerate another tenant's keys or the retention rule cannot be demonstrated.

The constraint changes the design. This is not a bucket-selection exercise. A training artifact may be regenerated from a course release, while an application backup is the recovery copy for an actual customer workload. They can share a storage service, but they should not share an authorization assumption, key prefix, or deletion path. Isolation first.

## How should private object storage handle US/EU app backup retention?

Start with the tenant boundary, then calculate capacity. Give each tenant an explicit namespace such as `tenant-482/training/` and `tenant-482/backups/`, keep the bucket private, and make the application server decide which tenant an operator may access before it creates a signed download link. A prefix is an organization aid; it is not, by itself, an authorization policy. The authorization check belongs in the trusted service that knows the tenant identity.

Region selection is a separate gate. Record whether a dataset belongs in the US or EU, and reject a placement that does not satisfy the application's residency requirement. Then forecast retained bytes as protected bytes per day multiplied by retention days, with headroom for growth, failed runs, and one complete restore. Include write and list requests, signed-link requests, and restore egress in the review. A storage rate alone is not a recovery budget.

Keep the retention rule reproducible. Store the object key, tenant ID, source checksum, byte count, completion time, and intended expiry in an external catalog. Lifecycle expiration can clean up ordinary copies, but day-level expiry is not a substitute for an hourly legal or operational hold. If a tenant needs a hold, record the exception outside the ordinary sweep and make its removal an explicit, authorized action.

One boundary matters immediately: a signed link is temporary access to one object, not a public bucket. The link should be minted only after the tenant and restore request have been checked, and it should expire quickly enough to fit the runbook. A permanent URL would turn a recovery artifact into a distribution surface.

## What fails when tenant isolation is treated as a prefix convention?

The failure mode is quiet. A list operation scoped to the wrong prefix can expose names even when object bodies remain unreadable; an overly broad service credential can make every tenant look like the same customer; and a deletion job that accepts a user-controlled prefix can erase more than its owner intended. These are design failures, not exotic storage failures.

Use separate tests for read, list, write, delete, and link creation. The expected result for a cross-tenant request is denial, while the expected result for an authorized request is access only to the selected object. Test both the application authorization layer and the storage credential's scope. A green upload test proves very little.

For the e-commerce case, the artifact catalog should carry tenant and release identity separately from the object key. That lets a restore operator answer “which training release produced this archive?” without trusting a filename, while a backup job can still use deterministic keys. Keep the catalog outside the bucket so a mistaken lifecycle rule cannot erase the evidence needed to select a restore. In a real runbook, this record also needs an owner, the source dataset's recovery point, the policy revision that produced the expiry, and a link to the verification result. Without those fields, an operator may find an object that looks current but cannot prove that it belongs to the requested tenant, was created under the current policy, or was ever restored successfully. The extra bookkeeping is not busywork: it is the difference between locating bytes and selecting a defensible recovery point during a busy incident.

This is where the on-call cost appears. The team must know who can approve a restore, how a revoked link is handled, and what alert fires when the newest successful copy ages past its SLO. I'm not sure any compatibility label can answer those questions; a permission test and a restore drill can.

## A small Go control loop for retention evidence

The storage API can remain behind a narrow interface. That keeps provider-specific signing and object calls out of the retention decision, makes the policy unit-testable, and prevents a future migration from changing tenant rules. The example records evidence rather than pretending that a successful upload equals a successful backup.

```go
package retention

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"time"
)

type ObjectStore interface {
	Put(ctx context.Context, key string, body []byte) error
	Delete(ctx context.Context, key string) error
}

type Evidence struct {
	TenantID   string
	ObjectKey  string
	Checksum   string
	Bytes      int
	Completed  time.Time
	ExpiresAt  time.Time
}

func SaveBackup(ctx context.Context, store ObjectStore, tenantID, key string, body []byte, expiresAt time.Time) (Evidence, error) {
	if tenantID == "" || key == "" {
		return Evidence{}, fmt.Errorf("tenant and object key are required")
	}
	if expiresAt.Before(time.Now().UTC()) {
		return Evidence{}, fmt.Errorf("retention expiry is in the past")
	}

	digest := sha256.Sum256(body)
	objectKey := tenantID + "/backups/" + key
	if err := store.Put(ctx, objectKey, body); err != nil {
		return Evidence{}, err
	}

	return Evidence{
		TenantID:  tenantID,
		ObjectKey: objectKey,
		Checksum:  hex.EncodeToString(digest[:]),
		Bytes:     len(body),
		Completed: time.Now().UTC(),
		ExpiresAt: expiresAt.UTC(),
	}, nil
}
```

The production version should validate that the caller's tenant context matches `tenantID`, use a canonical key builder rather than string concatenation, and persist the evidence transactionally with the backup-run state. The point is the order: establish identity, build a bounded key, write the object, and retain proof that the intended copy exists. Do not accept a browser-supplied tenant prefix as authority.

For large archives, use multipart upload and define what happens to incomplete parts. The AWS multipart overview describes the upload as a sequence of part uploads followed by completion; that model is useful even when the implementation is S3-compatible. Completion must be followed by checksum verification and cataloging. An abandoned upload is capacity with no recoverable artifact, so the runbook needs a cleanup policy and an alert for unfinished work.

## How should teams verify, roll back, and compare the storage choice?

The acceptance test is a restore, not a `200` from the writer. On a schedule, select the newest checksum-verified object for one tenant, issue a fresh signed download link, download it into an isolated restore workspace, compare the checksum, and measure elapsed time against the recovery SLO. Repeat the test for both US and EU placement where both are part of the service. Track newest-success age, protected bytes versus forecast, cross-tenant denial results, link expiry, and restore-check success.

Rollback has two meanings. If the backup data is wrong, stop the writer, select the latest verified catalog entry, and restore through the application's documented procedure. If the storage design fails an isolation or residency review, keep the old copy available, copy retained objects into the replacement system, verify restores there, and retire the old path only after the evidence is complete. Plan the migration's capacity and elapsed time; a policy document cannot make a cross-provider transfer instantaneous.

The catch is that generic object storage is not suitable when the requirement is immutable, ransomware-resistant retention, application-aware recovery, or a separately audited WORM control. Stick with a dedicated backup system or a storage configuration that supplies those controls when they are mandatory. Choose the simpler object-store path when private objects, scoped signed downloads, day-level lifecycle cleanup, and an owned restore runbook are enough.

My buy-versus-build table is intentionally boring:

| Choice | Fits when | Accept the trade-off |
|---|---|---|
| Generic S3-compatible storage | The team can own tenant authorization, cataloging, retention tests, and restores | More operational code and more responsibility for controls outside the object API |
| A managed backup service | Immutable retention, cataloged recovery, or application-aware restores are required | Agent, catalog, and platform lock-in |
| Self-hosted object storage | Residency, network placement, or provider independence dominates | The team owns capacity, upgrades, durability design, and on-call coverage |

The answer is not a universal cheapest provider. It is the option whose failure modes fit the team's SLO and staffing model, with evidence that a tenant-scoped restore works before the first incident.

## References

- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [AWS S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
