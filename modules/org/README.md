# modules/org

Connects an entire GCP organization to Aikido using Workload Identity Federation.

The Workload Identity Pool lives in a designated host project; IAM is granted at the organization level so all projects in the organization are covered.

It supports:

- the base GCP organization connection
- optional Artifact Registry access
- optional organization-scoped GCP VM scanning IAM for the Aikido-managed scanner service account

Set `aikido_region` to choose the default Aikido AWS principals used by the Workload Identity Provider:

- `eu` (default, `app.aikido.dev`)
- `us` (`app.us.aikido.dev`)
- `me` (`app.me.aikido.dev`)
- `au` (`app.au.aikido.dev`)

When `enable_vm_scanning = true`, the module:

- keeps the existing host-project WIF setup unchanged
- creates the custom role `aikidoSecurityVmScannerRole`
- creates the custom role `aikidoSecurityVmScannerSnapshotDeleteRole`
- binds the provided managed scanner service account to both roles at the organization level
- applies a condition on the delete role binding so it only matches Aikido-managed snapshots

## Authoritative IAM bindings

If you manage a role with `google_organization_iam_binding`, remove it from
`org_roles` and include `org_principal_members` in your existing binding's
complete member list. Do not manage the same role with both binding and member
resources. The output is available even when `org_roles = []` and waits for the
Workload Identity Pool to exist.

```hcl
module "aikido" {
  source = "github.com/AikidoSec/gcp-onboarding-terraform-module//modules/org"

  organization_id = "1234567890"
  project_id      = "my-host-project"
  project_number  = "123456789"
  org_roles      = ["roles/viewer", "roles/resourcemanager.folderViewer"]
}

# Update your existing binding rather than creating a second binding for this role.
resource "google_organization_iam_binding" "security_reviewers" {
  org_id = "1234567890"
  role   = "roles/iam.securityReviewer"
  members = concat(
    ["group:security@example.com"], # Preserve all existing members here.
    module.aikido.org_principal_members,
  )
}
```

When migrating an existing connection, review the plan together with your
existing authoritative binding so it retains Aikido's principals as the module's
individual member resources are removed.
