# Site Replication User Guide

This guide describes the user-facing topology rules for RustFS site replication when using the `mc admin replicate` workflow.

## Core Rules

When configuring site replication, RustFS applies these topology rules:

- When site replication is enabled for the first time, all sites may be empty. If buckets already exist, only the site receiving the `mc admin replicate add` request may contain buckets.
- When adding a site to an existing replication topology, the request must include every site that is already part of replication and at least one new site.
- A newly added site must be empty. Sites that already belong to the replication topology may contain buckets because replicated data is expected to exist there.

These rules prevent two independent non-empty datasets from being merged accidentally while still allowing an existing replication topology to grow safely.

## First-Time Setup

Run `mc admin replicate add` from the site that already contains data.

For example, if only `site1` has buckets:

```bash
mc admin replicate add site1 site2 site3
```

This request is valid when:

- `site1` is the site receiving the request and contains the existing buckets.
- `site2` and `site3` are empty.
- Every site reports a unique deployment ID.
- All sites have compatible IDP settings.

If all sites are empty, the request may be sent to any site included in the command.

Do not send the first setup request to an empty site while including another new remote site that already contains buckets. RustFS rejects that request and reports that the request should be sent to the site containing the data.

## Adding A Site

If `site1`, `site2`, and `site3` already form a site replication topology and you want to add an empty `site4`, the request must include all existing sites plus the new site:

```bash
mc admin replicate add site1 site2 site3 site4
```

This request is valid when:

- The current site replication state already contains `site1`, `site2`, and `site3`.
- The request includes every existing replicated site.
- `site4` has a new deployment ID.
- `site4` is empty.
- All sites have compatible IDP settings.

It is expected that `site1`, `site2`, and `site3` may already contain buckets. Existing replicated sites are not treated as conflicting data sources.

After the new site is added, RustFS synchronizes replication metadata and back-fills existing buckets and objects to the new site.

## Rejected Requests

RustFS rejects unsafe or ambiguous `add` requests, including:

- The request contains duplicate deployment IDs.
- The request does not include the local deployment.
- Site IDP settings are incompatible.
- During first-time setup, the local site and a new remote site both contain buckets.
- During first-time setup, the request is sent to an empty site while another new remote site already contains buckets.
- During expansion, the request omits an existing replicated site.
- During expansion, the request does not include any new site.
- During expansion, a new site already contains buckets.

These checks run before RustFS creates or updates site replication service accounts, sends peer join requests, or persists the new site replication state.

## Recommended Workflow

1. Identify the site that contains existing data.
2. Ensure every new site is empty.
3. Configure site aliases with `mc alias set`.
4. Run `mc admin replicate add` from the site containing data.
5. When expanding an existing topology, always include every existing replicated site and every new empty site.
6. Check replication state after the operation completes:

```bash
mc admin replicate status site1
mc admin replicate info site1
```

## Local Docker Validation

The repository includes a Docker Compose scenario that validates this behavior:

- Start three sites.
- Configure site replication.
- Write objects into the replicated sites.
- Start a fourth empty site.
- Add the fourth site to the replication topology.
- Verify that existing objects are back-filled to `site4` and new objects replicate in both directions.

See `.docker/test/site-replication/README.md` for the commands.
