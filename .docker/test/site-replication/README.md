# Site Replication Docker Compose Test

## Purpose

This directory contains a local RustFS site replication check. It is intended to verify the admin site-replication flow against real containers:

- three independent RustFS sites start successfully
- the `mc admin replicate add` command configures all three sites
- a bucket and objects written to the first three sites are replicated across the three-site set
- an optional fourth site can be started after data exists, joined to the replication set, and verified for back-fill and new object replication
- `mc admin replicate status` can read the resulting site-replication state

The compose file uses named volumes so the test does not require preparing host bind-mount directories.
Because Docker named volumes usually share the same physical device in local desktop environments, this test compose defaults `RUSTFS_UNSAFE_BYPASS_DISK_CHECK=true`. Keep that setting limited to local test and CI environments.

For the user-facing site replication topology rules, see `docs/operations/site-replication.md`.

## Files

- `docker-compose.yml`: three default RustFS sites, an optional fourth site, a volume permission helper, and one-shot setup/check containers
- `run-object-flow-check.sh`: host-side upload/download verification for replicated 10 MiB to 100 MiB objects

## Ports

Default host endpoints:

- Site 1 API: `http://127.0.0.1:9000`
- Site 1 Console: `http://127.0.0.1:9001`
- Site 2 API: `http://127.0.0.1:9010`
- Site 2 Console: `http://127.0.0.1:9011`
- Site 3 API: `http://127.0.0.1:9020`
- Site 3 Console: `http://127.0.0.1:9021`
- Site 4 API: `http://127.0.0.1:9030` (only with the `new-site` profile)
- Site 4 Console: `http://127.0.0.1:9031` (only with the `new-site` profile)

Default credentials are `rustfsadmin` / `rustfsadmin`. These are for local testing only.

## Run

From the repository root:

```bash
docker compose -f .docker/test/site-replication/docker-compose.yml up
```

To test a locally built image instead of Docker Hub `rustfs/rustfs:latest`, set `RUSTFS_SITE_REPL_IMAGE`:

```bash
docker build -f Dockerfile.source -t rustfs-site-repl-local:latest .
RUSTFS_SITE_REPL_IMAGE=rustfs-site-repl-local:latest \
docker compose -f .docker/test/site-replication/docker-compose.yml up
```

For a detached run:

```bash
docker compose -f .docker/test/site-replication/docker-compose.yml up -d
docker compose -f .docker/test/site-replication/docker-compose.yml logs -f site-replication-setup
```

To test adding a new site after the initial three-site set already contains objects, first let `site-replication-setup` complete, then start the `new-site` profile:

```bash
docker compose -f .docker/test/site-replication/docker-compose.yml up -d
docker compose -f .docker/test/site-replication/docker-compose.yml logs -f site-replication-setup
docker compose -f .docker/test/site-replication/docker-compose.yml --profile new-site up -d --no-deps rustfs-site4
docker compose -f .docker/test/site-replication/docker-compose.yml --profile new-site up --no-deps \
  --abort-on-container-exit --exit-code-from site-replication-expand-site4 \
  site-replication-expand-site4
```

## Test Flow

The compose stack performs these steps:

1. `site-replication-volume-permission-helper` fixes ownership on all named volumes for the RustFS runtime user.
2. `rustfs-site1`, `rustfs-site2`, and `rustfs-site3` start as separate RustFS sites.
3. Each site exposes its S3 API and Console on a unique host port.
4. Health checks wait for `/health/ready` on each RustFS container.
5. `site-replication-setup` configures `mc` aliases for all three sites.
6. The setup container waits until `mc admin info` succeeds for all sites.
7. It runs:

```bash
mc admin replicate add site1 site2 site3
```

8. It creates the test bucket on site 1 and uploads pre-site4 objects from site 1, site 2, and site 3.
9. It polls the other two sites for each object until the replicated objects are visible.
10. It prints `mc admin replicate status site1`.

The setup container exits with status `0` only after the object replication check passes.

## New Site Join Check

The `new-site` profile starts `rustfs-site4` only after the default three-site setup has completed. The `site-replication-expand-site4` container then:

1. Configures aliases for all four sites.
2. Runs:

```bash
mc admin replicate add site1 site2 site3 site4
```

3. Waits for the objects that existed before site 4 joined to appear on site 4.
4. Uploads a new object to site 4 and waits for it on sites 1, 2, and 3.
5. Uploads a new object to site 1 and waits for it on site 4.
6. Prints replication status from site 1 and site 4.

The expansion container exits with status `0` only after both back-fill and post-join replication checks pass.

## Object Flow Check

After the compose setup succeeds, run the larger object flow check from the repository root:

```bash
.docker/test/site-replication/run-object-flow-check.sh
```

The script creates five local files and uploads them from different sites:

- 10 MiB from site 1
- 25 MiB from site 2
- 50 MiB from site 3
- 75 MiB from site 1
- 100 MiB from site 2

For each uploaded object, the script waits for replication to the other two sites, downloads the object from those sites, and verifies both byte size and SHA-256 checksum. It uses a temporary `mc` config directory, so it does not overwrite existing host aliases.

The default bucket is `site-repl-flow-check`. Override it when needed:

```bash
RUSTFS_SITE_REPL_FLOW_BUCKET='site-repl-large-flow' \
.docker/test/site-replication/run-object-flow-check.sh
```

The script keeps uploaded objects under a timestamped prefix. Override the prefix for repeatable runs:

```bash
RUSTFS_SITE_REPL_FLOW_PREFIX='manual-check-001' \
.docker/test/site-replication/run-object-flow-check.sh
```

If replication is slow on the local machine, increase polling:

```bash
RUSTFS_SITE_REPL_WAIT_ATTEMPTS=180 \
RUSTFS_SITE_REPL_WAIT_SLEEP_SECONDS=2 \
.docker/test/site-replication/run-object-flow-check.sh
```

## Optional Settings

Override local test credentials:

```bash
RUSTFS_SITE_REPL_ACCESS_KEY='localadmin' \
RUSTFS_SITE_REPL_SECRET_KEY='localadmin-secret' \
docker compose -f .docker/test/site-replication/docker-compose.yml up
```

Use a different test bucket:

```bash
RUSTFS_SITE_REPL_BUCKET='site-repl-check' \
docker compose -f .docker/test/site-replication/docker-compose.yml up
```

Enable ILM expiry rule replication during site setup:

```bash
RUSTFS_SITE_REPL_ENABLE_ILM_EXPIRY=true \
docker compose -f .docker/test/site-replication/docker-compose.yml up
```

Use this only when the test needs lifecycle expiry metadata included in site replication.

## Manual Checks

After the setup container succeeds, you can inspect the sites with `mc` from the host:

```bash
mc alias set site1 http://127.0.0.1:9000 rustfsadmin rustfsadmin
mc alias set site2 http://127.0.0.1:9010 rustfsadmin rustfsadmin
mc alias set site3 http://127.0.0.1:9020 rustfsadmin rustfsadmin
mc alias set site4 http://127.0.0.1:9030 rustfsadmin rustfsadmin

mc admin replicate info site1
mc admin replicate status site1
mc stat site2/site-repl-demo/pre-site4/from-site1.txt
mc stat site3/site-repl-demo/pre-site4/from-site1.txt
mc stat site4/site-repl-demo/pre-site4/from-site1.txt
mc stat site4/site-repl-demo/post-site4/from-site1.txt
```

After larger object flow checks, replication should converge without a growing queue:

```bash
mc admin replicate status site1
mc admin replicate status site2
mc admin replicate status site3
```

Useful Docker commands:

```bash
docker compose -f .docker/test/site-replication/docker-compose.yml ps
docker compose -f .docker/test/site-replication/docker-compose.yml logs --no-color --tail=200
docker compose -f .docker/test/site-replication/docker-compose.yml logs --no-color site-replication-setup
docker compose -f .docker/test/site-replication/docker-compose.yml logs --no-color site-replication-expand-site4
```

## Cleanup

Remove containers and named volumes:

```bash
docker compose -f .docker/test/site-replication/docker-compose.yml down -v
```

Use `down -v` before rerunning the full setup from scratch. Site replication state is persisted in the named volumes, so rerunning without deleting volumes may attempt to add an already-configured replication topology.
