# Correct the correlator namespace, and keep the sqlserver password out of state

**Branch** `update-namespace-iam`

## Commit message

    Correct the correlator Workload Identity namespace, and stop storing the
    sqlserver password in state

    The grant named qa-correlator. The cluster namespace is ns-qa-correlator,
    which is what the chart creates and what the principal has to match. The
    binding is name based, so the mismatch produced a secret the pod could not
    read and a mount failure that reads as a missing secret.

    The contained user passwords already avoided state through an ephemeral
    resource and write-only arguments. The instance's built-in sqlserver login
    did not: random_password with secret_data put it in terraform.tfstate in
    plaintext. It now uses the same pattern, with root_password_wo on the
    instance and secret_data_wo on the secret, both keyed to
    root_password_version so they are written in the same apply and stay in
    step.

## Files

| File | Change |
|---|---|
| `secrets/npr/iam-correlator.yaml` | `qa-correlator` to `ns-qa-correlator` |
| `modules/cloudsql-mssql/main.tf` | ephemeral root password, write-only on instance and secret |
| `modules/cloudsql-mssql/variables.tf` | `root_password_version` |
| `variables.tf`, `main.tf` | plumb it through `sql_instances` |
| `tfvars/npr/data.tfvars` | `root_password_version = 1` |
| `provider.tf` | google floor to 7.40 for `root_password_wo` |
| `secrets/README.md` | namespace format, and the root password now covered |

## Rotating the sqlserver login

Bump `root_password_version` in tfvars. One apply sets the new password on the
instance and adds a new secret version. Terraform cannot tell you the value:
read it from `<instance>-sqlserver-password`.

## Expected plan

1 IAM member added on `enr-app-corr-db-q01_correlator`, 1 secret version, 1
instance updated in place. **0 to destroy.** Anything destroying is wrong.

`random_password.root` leaves state. That is a state-only removal, no API call.

## One thing to know before approving

**This apply rotates the sqlserver password.** Moving from `root_password` to
`root_password_wo` with version 1 is a change from unset, so the provider writes
a new password. Anything holding the current one stops working. The new value is
in Secret Manager immediately.

If the apply fails between the instance update and the secret version, the two
diverge. Recovery is to bump `root_password_version` again and re-apply, which
rewrites both.
