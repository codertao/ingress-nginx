# CLAUDE.md

A maintenance fork of the retired kubernetes/ingress-nginx. The goal is **minimal changes to keep NGINX current and patched**: no feature work, no refactors. Branch: `public-v1.16`. Never push upstream or reference upstream PR numbers as ours.

The step-by-step process for updating NGINX, building images and running e2e lives in the **"Maintenance Fork" section at the top of `README.md`**. Follow it, and keep its "Current state" table updated whenever versions change.

## Key files

- `images/nginx/rootfs/build.sh`: NGINX and module versions (`export *_VERSION=...`). Each export has a `# Check for recent changes: .../compare/<ver>...` comment that must be bumped along with it.
- `images/nginx/rootfs/patches/`: NGINX core patches, applied in `ls` order with `patch -p1`.
- `images/nginx/TAG`: base image version. Bump on every base image change.
- `NGINX_BASE`: base image the controller builds on, pinned `image:tag@sha256:digest`.
- `TAG`, `Makefile`: controller image tag and name. `CONTROLLER_IMAGE` defaults to `docker.io/codertao2/ingress-nginx`. If `REGISTRY` is set, the Makefile falls back to upstream's `$(REGISTRY)/controller`. The e2e and dev scripts (`test/e2e/run-*.sh`, `build/dev-env.sh`) export `REGISTRY=ingress-controller` and depend on that, so keep the fallback.
- `rootfs/etc/nginx/template/nginx.tmpl`, `rootfs/etc/nginx/lua/`: generated NGINX config and Lua. Check these when upstream behaviour changes.

## NGINX update rules

- **OpenResty is the source of truth for patches and Lua module versions.** Use the OpenResty release whose NGINX core matches the target version, or the closest earlier patch release. Confirm with the user before targeting a version with no matching OpenResty release.
- **Derive the patch list from OpenResty's `util/mirror-tarballs`** at the release tag, not from the `patches/nginx/<ver>/` directory listing. The directory contains stale patches; for example, CVE patches already merged into core are only applied for older versions. Ignore `server_header`, `builtin_error_page_footer` and `no_pool`, which we deliberately don't carry.
- Keep OpenResty's patch file names (`NN_nginx-<openresty-nginx-ver>-<name>.patch`), numbered in OpenResty's apply order. `00_drop-alias-root.patch` is ours: it removes the `root`/`alias` directives and may need a manual rebase.
- `ngx_devel_kit` must come from `openresty/ngx_devel_kit`, because vision5's version breaks on NGINX ≥ 1.31.3.
- Before building, apply the full patch set to a pristine NGINX tarball in a temp directory. It's much faster than a docker build and catches rejects early.
- The `get_src` sha256 hashes in `build.sh` aren't verified (checksum check is disabled), so leave them alone.

## Verifying a base image

After `docker build` in `images/nginx/rootfs`, smoke-test the image before the user publishes it:
- `nginx -V` shows the expected version.
- Every `/etc/nginx/modules/*.so` loads with `nginx -t`.
- A Lua request (`content_by_lua_block`) and `more_set_headers` work.
- A stream server with `proxy_protocol` passes the config test.
- An `alias` directive fails with "unknown directive".

OTLP "empty endpoint" errors during the smoke test are expected, because no collector is configured.

## Build host quirks

- The host Go (`/usr/local/go`, 1.20) is too old for this repo (needs 1.26). All real builds run in containers (`build/run-in-docker.sh`), so don't use host `go build` or `go test`.
- The host uses cgroup v1, so kind needs `K8S_VERSION=v1.34.11`.
- No QEMU/binfmt is available, so images are amd64 only.
- The e2e spec `use-proxy-protocol should enable PROXY Protocol for TCP` takes about 10 minutes by design and prints progress reports. That isn't a failure; check `test/junitreports/report-e2e-test-suite.xml` for actual results.

## Conventions

- Don't commit, push or publish images unless asked. The user pushes images to Docker Hub.
- Downloaded upstream sources (tarballs, patches) go in fresh temp directories, never into the repo directly.
