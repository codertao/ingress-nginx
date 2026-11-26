# Maintenance Fork

The original ingress-nginx was retired.

This fork uses AI to make bare minimum updates to keep NGINX up to date.

Work happens on the `public-v1.16` branch.

## Current state

| Component | Version | Where it's set |
|---|---|---|
| NGINX | 1.31.10 | `NGINX_VERSION` in `images/nginx/rootfs/build.sh` |
| OpenResty patch set | v1.31.9.1 | `images/nginx/rootfs/patches/` |
| NGINX base image | `docker.io/codertao2/nginx:1.31.10` (amd64 only) | `NGINX_BASE` (pinned by digest) |
| Controller | v1.16.0 | `TAG` |

## How the pieces fit

1. **NGINX base image** (`images/nginx/`): NGINX compiled from source with OpenResty's core patches, Lua, ModSecurity, OpenTelemetry, njs, etc. Built from `images/nginx/rootfs/build.sh`.
2. **Controller image** (repo root): the Go controller, built `FROM` the base image named in `NGINX_BASE`.

An NGINX update changes the base image, then the controller is rebuilt on top of it.

## Updating NGINX

1. **Pick a version that OpenResty supports.** Our NGINX core patches come from OpenResty, so check <https://openresty.org/en/download.html> for the newest `1.X.Y.Z` release. If NGINX has a newer patch release than OpenResty, it's usually fine to use OpenResty's patches for the previous NGINX patch version (1.31.10 currently uses the 1.31.9.1 patches), as long as they apply cleanly.
2. **Update the patches** in `images/nginx/rootfs/patches/`:
   - Take the patches from `https://github.com/openresty/openresty/tree/v<OPENRESTY_VERSION>/patches/nginx/<NGINX_VERSION>`.
   - Only use the patches that OpenResty's `util/mirror-tarballs` actually applies for that NGINX version. The directory also contains stale patches that are already in NGINX core.
   - Keep the `NN_` prefixes so patches apply in OpenResty's order, and keep OpenResty's file names.
   - `00_drop-alias-root.patch` is ours, not OpenResty's (it removes the `root`/`alias` directives). Rebase it by hand if it stops applying.
3. **Bump versions in `images/nginx/rootfs/build.sh`:** set `NGINX_VERSION`, and set the Lua modules (lua-nginx-module, stream-lua, lua-resty-core, luajit2, headers-more, etc.) to the versions listed in that OpenResty release's changelog. ngx_devel_kit comes from OpenResty's fork (`openresty/ngx_devel_kit`), not vision5's repo, because only the fork supports NGINX ≥ 1.31.3.
4. **Bump `images/nginx/TAG`.** This is the base image's own version number, unrelated to the NGINX version.

## Building and publishing the base image

```bash
cd images/nginx/rootfs
docker build --platform linux/amd64 -t docker.io/codertao2/nginx:<NGINX_VERSION> .
docker push docker.io/codertao2/nginx:<NGINX_VERSION>
```

A build takes a while because OpenTelemetry and ModSecurity are compiled from source. Watch the log for `Hunk FAILED` or `.rej`, which mean a patch didn't apply.

Then pin `NGINX_BASE` to the pushed digest:

```bash
docker buildx imagetools inspect docker.io/codertao2/nginx:<NGINX_VERSION>   # shows "Digest: sha256:..."
echo "docker.io/codertao2/nginx:<NGINX_VERSION>@sha256:<DIGEST>" > NGINX_BASE
```

Only amd64 images are built today. Multi-arch (`make -C images/nginx push`) needs QEMU/binfmt or native arm builders.

## Building the controller image

From the repo root:

```bash
make build image
docker push docker.io/codertao2/ingress-nginx:$(cat TAG)
```

`make build` compiles the Go binaries inside a container, so host Go doesn't matter. `make image` builds `$(CONTROLLER_IMAGE):$(TAG)` on top of `NGINX_BASE`; `CONTROLLER_IMAGE` defaults to `docker.io/codertao2/ingress-nginx`, and the chroot variant (`make build image-chroot`) is `$(CONTROLLER_IMAGE)-chroot`. Override `TAG=...` or `CONTROLLER_IMAGE=...` as needed. Setting `REGISTRY` switches back to the upstream `$(REGISTRY)/controller` naming, which the e2e scripts rely on. Don't use `make release`, which needs a multi-arch base image.

## Running the e2e tests

```bash
K8S_VERSION=v1.34.11 make kind-e2e-test
```

This creates a kind cluster, builds the controller and e2e images, and runs the full suite (around 45 minutes). The JUnit results end up in `test/junitreports/report-e2e-test-suite.xml`.

One-time setup on the build host:

- **kind and kubectl** aren't in apt. Install the binaries into `~/.local/bin`: `go install sigs.k8s.io/kind@latest`, and kubectl from `https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl`.
- **ginkgo** must be on the host `PATH`. The script otherwise tries to `go install` it with the host Go, which is too old. Copy it out of the test-runner image:
  ```bash
  IMG=$(grep -oE 'registry.k8s.io/ingress-nginx/e2e-test-runner:[^"]+' test/e2e-image/Makefile)
  id=$(docker create "$IMG"); docker cp "$id":/go/bin/ginkgo ~/.local/bin/ginkgo; docker rm "$id"
  ```
- **`~/.kube` must be owned by you** (`sudo chown -R "$USER:$USER" ~/.kube`). Docker bind mounts can create it as root.
- **Raise the inotify limits:** `sudo sysctl fs.inotify.max_user_watches=524288 fs.inotify.max_user_instances=512`.
- **Kubernetes 1.34 is required** (`K8S_VERSION=v1.34.11`) while the host uses cgroup v1. Kubernetes 1.35's kubelet refuses to start on cgroup v1.

Expected noise: the `use-proxy-protocol should enable PROXY Protocol for TCP` spec takes around 10 minutes and prints Ginkgo progress reports showing `IO wait`. That's normal, because it waits for the 600s stream `proxy_timeout`. Check the JUnit report for actual failures.


# Ingress NGINX Retirement

## Retiring

[What You Need to Know about Ingress NGINX Retirement](https://www.kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/):

* Best-effort maintenance will continue until March 2026.
* Afterward, there will be no further releases, no bugfixes, and no updates to resolve any security vulnerabilities that may be discovered.
* Existing deployments of Ingress NGINX will not be broken.
  * Existing project artifacts such as Helm charts and container images will remain available.

# Ingress NGINX Controller

[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/5691/badge)](https://bestpractices.coreinfrastructure.org/projects/5691)
[![Go Report Card](https://goreportcard.com/badge/github.com/kubernetes/ingress-nginx)](https://goreportcard.com/report/github.com/kubernetes/ingress-nginx)
[![GitHub license](https://img.shields.io/github/license/kubernetes/ingress-nginx.svg)](https://github.com/kubernetes/ingress-nginx/blob/main/LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/kubernetes/ingress-nginx.svg)](https://github.com/kubernetes/ingress-nginx/stargazers)

## Overview

ingress-nginx was an Ingress controller for Kubernetes using [NGINX](https://www.nginx.org/) as a reverse proxy and load
balancer.

[Learn more about Ingress on the Kubernetes documentation site](https://kubernetes.io/docs/concepts/services-networking/ingress/).

## Usage warnings

If you are not already using ingress-nginx, you should not be deploying it as it is [not being developed](#retiring). Instead you should identify a [Gateway API](https://gateway-api.sigs.k8s.io/guides/) implementation and use it.

Do not use in multi-tenant Kubernetes production installations. This project assumes that users that can create Ingress objects are administrators of the cluster. See the [FAQ](https://kubernetes.github.io/ingress-nginx/faq/#faq) for more.

## Troubleshooting

If you encounter issues, review the [troubleshooting docs](docs/troubleshooting.md),
[search for an issue](https://github.com/kubernetes/ingress-nginx/issues), or talk to us on the
[#ingress-nginx-users channel](https://kubernetes.slack.com/messages/ingress-nginx-users) on the Kubernetes Slack server.

## Changelog

See [the list of releases](https://github.com/kubernetes/ingress-nginx/releases) for all changes.
For detailed changes for each release, please check the [changelog-$version.md](./changelog) file for the release version.
For detailed changes on the `ingress-nginx` helm chart, please check the changelog folder for a specific version.
[CHANGELOG-$current-version.md](./charts/ingress-nginx/changelog) file.

### Supported Versions table

Supported versions for the ingress-nginx project mean that we have completed E2E tests, and they are passing for
the versions listed. Ingress-Nginx versions **may** work on older versions, but the project does not make that guarantee.

| Supported | Ingress-NGINX version | k8s supported version         | Alpine Version | Nginx Version | Helm Chart Version |
| :-------: | --------------------- | ----------------------------- | -------------- | ------------- | ------------------ |
|    🔄     | **v1.16.0**           | 1.35, 1.34, 1.33, 1.32, 1.31  | 3.23.3         | 1.31.8        | 4.15.1             |
|           | **v1.15.1**           | 1.35, 1.34, 1.33, 1.32, 1.31  | 3.23.3         | 1.27.1        | 4.15.1             |
|           | **v1.15.0**           | 1.35, 1.34, 1.33, 1.32, 1.31  | 3.23.3         | 1.27.1        | 4.15.0             |
|           | **v1.14.4**           | 1.34, 1.33, 1.32, 1.31, 1.30  | 3.23.3         | 1.27.1        | 4.14.4             |
|           | **v1.14.3**           | 1.34, 1.33, 1.32, 1.31, 1.30  | 3.23.2         | 1.27.1        | 4.14.3             |
|           | **v1.14.2**           | 1.34, 1.33, 1.32, 1.31, 1.30  | 3.23.2         | 1.27.1        | 4.14.2             |
|           | **v1.14.1**           | 1.34, 1.33, 1.32, 1.31, 1.30  | 3.22.2         | 1.27.1        | 4.14.1             |
|           | **v1.14.0**           | 1.34, 1.33, 1.32, 1.31, 1.30  | 3.22.2         | 1.27.1        | 4.14.0             |
|           | **v1.13.8**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.23.3         | 1.27.1        | 4.13.8             |
|           | **v1.13.7**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.23.2         | 1.27.1        | 4.13.7             |
|           | **v1.13.6**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.23.2         | 1.27.1        | 4.13.6             |
|           | **v1.13.5**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.2         | 1.27.1        | 4.13.5             |
|           | **v1.13.4**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.2         | 1.27.1        | 4.13.4             |
|           | **v1.13.3**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.1         | 1.27.1        | 4.13.3             |
|           | **v1.13.2**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.1         | 1.27.1        | 4.13.2             |
|           | **v1.13.1**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.1         | 1.27.1        | 4.13.1             |
|           | **v1.13.0**           | 1.33, 1.32, 1.31, 1.30, 1.29  | 3.22.0         | 1.27.1        | 4.13.0             |
|           | v1.12.8               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.22.2         | 1.25.5        | 4.12.8             |
|           | v1.12.7               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.22.1         | 1.25.5        | 4.12.7             |
|           | v1.12.6               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.22.1         | 1.25.5        | 4.12.6             |
|           | v1.12.5               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.22.1         | 1.25.5        | 4.12.5             |
|           | v1.12.4               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.22.0         | 1.25.5        | 4.12.4             |
|           | v1.12.3               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.21.3         | 1.25.5        | 4.12.3             |
|           | v1.12.2               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.21.3         | 1.25.5        | 4.12.2             |
|           | v1.12.1               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.21.3         | 1.25.5        | 4.12.1             |
|           | v1.12.0               | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.21.0         | 1.25.5        | 4.12.0             |
|           | v1.12.0-beta.0        | 1.32, 1.31, 1.30, 1.29, 1.28  | 3.20.3         | 1.25.5        | 4.12.0-beta.0      |
|           | v1.11.8               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.22.0         | 1.25.5        | 4.11.8             |
|           | v1.11.7               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.21.3         | 1.25.5        | 4.11.7             |
|           | v1.11.6               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.21.3         | 1.25.5        | 4.11.6             |
|           | v1.11.5               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.21.3         | 1.25.5        | 4.11.5             |
|           | v1.11.4               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.21.0         | 1.25.5        | 4.11.4             |
|           | v1.11.3               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.3         | 1.25.5        | 4.11.3             |
|           | v1.11.2               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.11.2             |
|           | v1.11.1               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.11.1             |
|           | v1.11.0               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.11.0             |
|           | v1.10.6               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.21.0         | 1.25.5        | 4.10.6             |
|           | v1.10.5               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.3         | 1.25.5        | 4.10.5             |
|           | v1.10.4               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.10.4             |
|           | v1.10.3               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.10.3             |
|           | v1.10.2               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.20.0         | 1.25.5        | 4.10.2             |
|           | v1.10.1               | 1.30, 1.29, 1.28, 1.27, 1.26  | 3.19.1         | 1.25.3        | 4.10.1             |
|           | v1.10.0               | 1.29, 1.28, 1.27, 1.26        | 3.19.1         | 1.25.3        | 4.10.0             |
|           | v1.9.6                | 1.29, 1.28, 1.27, 1.26, 1.25  | 3.19.0         | 1.21.6        | 4.9.1              |
|           | v1.9.5                | 1.28, 1.27, 1.26, 1.25        | 3.18.4         | 1.21.6        | 4.9.0              |
|           | v1.9.4                | 1.28, 1.27, 1.26, 1.25        | 3.18.4         | 1.21.6        | 4.8.3              |
|           | v1.9.3                | 1.28, 1.27, 1.26, 1.25        | 3.18.4         | 1.21.6        | 4.8.*              |
|           | v1.9.1                | 1.28, 1.27, 1.26, 1.25        | 3.18.4         | 1.21.6        | 4.8.*              |
|           | v1.9.0                | 1.28, 1.27, 1.26, 1.25        | 3.18.2         | 1.21.6        | 4.8.*              |
|           | v1.8.4                | 1.27, 1.26, 1.25, 1.24        | 3.18.2         | 1.21.6        | 4.7.*              |
|           | v1.7.1                | 1.27, 1.26, 1.25, 1.24        | 3.17.2         | 1.21.6        | 4.6.*              |
|           | v1.6.4                | 1.26, 1.25, 1.24, 1.23        | 3.17.0         | 1.21.6        | 4.5.*              |
|           | v1.5.1                | 1.25, 1.24, 1.23              | 3.16.2         | 1.21.6        | 4.4.*              |
|           | v1.4.0                | 1.25, 1.24, 1.23, 1.22        | 3.16.2         | 1.19.10†      | 4.3.0              |
|           | v1.3.1                | 1.24, 1.23, 1.22, 1.21, 1.20  | 3.16.2         | 1.19.10†      | 4.2.5              |

See [Updating NGINX-Ingress to use the stable Ingress API (July 26, 2021)](https://kubernetes.io/blog/2021/07/26/update-with-ingress-nginx/)
to upgrade to the stable Ingress API before upgrading to Kubernetes 1.22.

## Get Involved

Thanks for taking the time to join our community and start contributing!

- This project adheres to the [Kubernetes Community Code of Conduct](https://git.k8s.io/community/code-of-conduct.md).
  By participating in this project, you agree to abide by its terms.
- **Contributing**: Documentation contributions are welcome.

  - Read [`CONTRIBUTING.md`](CONTRIBUTING.md) for information about the workflow that we
    expect and instructions on the developer certificate of origin that we require.
  - Join our Kubernetes Slack channel for developer discussion : [#ingress-nginx-dev](https://kubernetes.slack.com/archives/C021E147ZA4).
  - Submit GitHub issues for documentation problems.
    - Please make sure to read the [Issue Reporting Checklist](https://github.com/kubernetes/ingress-nginx/blob/main/CONTRIBUTING.md#issue-reporting-guidelines) before opening an issue. Issues not conforming to the guidelines **may be closed immediately**.

- **Support**:

  - Join the [#ingress-nginx-users](https://kubernetes.slack.com/messages/CANQGM8BA/) channel inside the [Kubernetes Slack](http://slack.kubernetes.io/) to ask questions or get support from the maintainers and other users.
  - The [GitHub issues](https://github.com/kubernetes/ingress-nginx/issues) in the repository are **exclusively** for bug reports and feature requests.

## License

[Apache License 2.0](https://github.com/kubernetes/ingress-nginx/blob/main/LICENSE)
