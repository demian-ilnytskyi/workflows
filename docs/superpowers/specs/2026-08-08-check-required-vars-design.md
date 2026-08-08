# check-required-vars composite action

## Problem

Azure Pipelines workflows in `codinghouse/workflows/azure-pipelines` repeat an
inline `check_required_var` bash function in nearly every step that needs a
secret or pipeline variable (deploy steps, coverage push, mobile builds,
integration tests). It checks that a named variable is non-empty and fails
fast with a clear error if not. This repo's GitHub Actions workflows have no
equivalent — missing secrets currently fail late and unclearly (e.g. deep
inside a `psql` or `supabase` CLI call).

## Goal

A single reusable composite action, callable from any workflow in this repo,
that validates a list of required environment variables are set and
non-empty, failing the job early with a clear message if not.

## Design

New composite action: `.github/actions/check-required-vars/action.yml`

**Input:**
- `vars` (required, string) — newline-separated list of variable names to
  validate, e.g.:
  ```yaml
  with:
    vars: |
      SUPABASE_PROJECT_ID
      SUPABASE_DB_PASSWORD
  ```

**Behavior:**
- Composite `bash` step iterates over each non-blank line in `vars`.
- For each name, uses bash indirect expansion (`${!name}`) to read the
  current value of that variable from the step's environment.
- Collects all missing/empty names (not fail-fast per name) so a caller sees
  every problem in one run instead of fixing them one at a time.
- If any are missing, prints one `Error: <NAME> is EMPTY or NOT set!` line
  per missing var, then exits 1.
- If all are present, the step succeeds silently.

**Scope of values:** GitHub Actions composite action steps only see
environment variables set at job-level `env:` or on the exact step that
invokes the action — not variables exported by an earlier sibling step. This
action does not attempt to work around that. Callers must already have the
required secrets in the job's `env:` block (this is already the existing
convention in workflows like `supabase_deploy.yml`), or set them directly on
the step invoking this action.

**Out of scope:** Azure's version also detects an unresolved `$(...)`
literal, which signals a variable group not linked to the pipeline job. GH
Actions secrets/vars have no equivalent failure mode — a missing secret
simply resolves to an empty string — so this check is omitted.

## Example usage

```yaml
- name: Check required secrets
  uses: ./.github/actions/check-required-vars
  with:
    vars: |
      SUPABASE_PROJECT_ID
      SUPABASE_DB_PASSWORD
      SUPABASE_POOLER_REGION
```

Placed early in `supabase_deploy.yml`'s `deploy` job (after checkout, before
the DB/functions deploy steps) and in other workflows that currently assume
secrets are present without checking.

## Testing

Manual: run a workflow with a secret deliberately unset (e.g. via
`workflow_dispatch` on a branch/environment missing the secret) and confirm
the job fails at the check step with a clear message, before reaching the
step that would otherwise fail obscurely.
