# `persql-action`

GitHub Action: claim a TTL-bounded branch on a [PerSQL](https://persql.com)
database for an ephemeral PR preview environment. One step, no
teardown required — the branch is automatically reaped when its TTL
elapses.

## Usage

```yaml
# .github/workflows/preview-db.yml
name: Preview DB
on:
  pull_request:

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Claim PerSQL branch
        id: persql
        uses: persql/preview-db-action@v1
        with:
          token: ${{ secrets.PERSQL_TOKEN }}
          database: acme/orders
          branch: pr-${{ github.event.pull_request.number }}
          ttl-seconds: 86400      # 1 day, default

      # Anything in this job can now target the branch via its scoped token.
      - name: Apply migrations + smoke test
        env:
          PERSQL_TOKEN: ${{ steps.persql.outputs.token }}
        run: |
          pnpm dlx @persql/cli@latest db migrate
          pnpm test:e2e

      - name: Comment branch URL on PR
        uses: actions/github-script@v8
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🌿 Preview DB: \`${{ steps.persql.outputs.branch-ref }}\` (expires ${{ steps.persql.outputs.expires-at }})`,
            });
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `token` | yes | — | PerSQL bearer token with `claim_branch` permission on `database`. Usually `${{ secrets.PERSQL_TOKEN }}`. |
| `database` | yes | — | Database path as `<namespace>/<database-slug>`. |
| `branch` | yes | — | Branch ref to claim. Idempotent — re-running on the same ref refreshes the lease and resets the branch to the parent's current schema. |
| `ttl-seconds` | no | `86400` | Lease length in seconds (max 30 days). Branch is auto-reaped after this. |
| `role` | no | `write` | Role of the returned scoped token: `read`, `write`, or `admin`. |
| `api-url` | no | `https://api.persql.com` | API base URL — point at staging for testing. |

## Outputs

| Output | Description |
| --- | --- |
| `branch-ref` | The ref of the claimed branch. |
| `token` | Scoped bearer token for the branch (masked in logs via `::add-mask::`). |
| `expires-at` | ISO-8601 timestamp when the lease expires. |
| `outcome` | `created` on first claim, `reset` when reclaiming an existing ref. |

## Why a branch, not a fresh database?

A PerSQL branch carries the parent's **schema** — your tables and
indexes, with empty data — so it provisions in milliseconds and you
skip re-applying your migrations on every PR. Writes are isolated from
`main`; seed test data with a fixture or migration step in the same
job. You pay only for the requests the branch serves and the rows you
write into it. Operations are uniform across the SDK and `/v1`, so the
same code works against a branch or the parent.

## Cleanup

The TTL handles it. The default 24-hour lease covers most PRs; bump
for long-running ones, or pass `ttl-seconds: 0` to keep the branch
until you explicitly drop it via `DELETE /v1/db/:ns/:db/branches/:ref`.

If you want immediate cleanup on PR close, add a second job
conditional on `github.event.action == 'closed'` that calls the delete
endpoint. For most teams the TTL pattern is enough.

## Pin to a major version

Tags follow semver. `@v1` follows the latest non-breaking 1.x release
— recommended. Pin to a specific tag (`@v1.0.0`) if you'd rather
upgrade explicitly.

## License

MIT.
