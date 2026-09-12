# check-fk-cascade composite action

## Problem

Foreign keys added to `supabase/data-base/tables/*.sql` across consumer
projects sometimes forget `ON UPDATE CASCADE ON DELETE CASCADE`, which is the
project convention for keeping dependent rows in sync instead of leaving
orphans or failing writes. Nothing in CI caught this — it was only found by
manual review.

## Goal

A reusable composite action, callable from `supabase_ci.yml`, that inspects
every foreign key in the `public` schema of the running database container
and fails the job if any is missing `ON UPDATE CASCADE` or
`ON DELETE CASCADE`, unless its table is explicitly listed in a skip file.

## Design

New composite action: `.github/actions/check-fk-cascade/action.yml`

**Input:**
- `skip_file` (optional, string, default
  `supabase/data-base/fk_cascade_skip.txt`) — newline-separated list of table
  names whose foreign keys are intentionally excluded from the check. Blank
  lines and `#`-prefixed comment lines are ignored.

**Behavior:**
- Queries `pg_constraint` in the already-running `supabase-test-db`
  container for every `contype = 'f'` row in the `public` namespace.
- Flags any constraint where `confupdtype <> 'c'` or `confdeltype <> 'c'`
  (i.e. not `CASCADE`), excluding tables named in `skip_file`.
- Prints each offending `table.constraint` with its current update/delete
  action, then exits 1.
- Exits 0 with a success message when nothing is flagged.

**Wiring:** added as a step in `supabase_ci.yml` right after the security
advisor linter, gated on the existing `validate_sql` input, with a new
`fk_cascade_skip_file` workflow input (default
`supabase/data-base/fk_cascade_skip.txt`) passed through.

**Consumer setup:** a project opts a table out of the check by adding its
name to `supabase/data-base/fk_cascade_skip.txt`. An empty file (present but
with no entries) means every foreign key in the schema must cascade on both
update and delete.
