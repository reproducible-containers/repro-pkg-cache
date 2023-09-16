# Dockerfile examples for reproducing package cache (e.g., `/etc/apk/cache`)

This repo contains Dockerfile examples to reproduce package cache
with specific versions, by pushing the cache to an image registry.

See [`Dockerfile.alpine`](./Dockerfile.alpine):
```dockerfile
ARG PACKAGES="gcc neofetch"

# PKG_CACHE defaults to the "pkg-cache-local" stage in this image.
# Can be overridden to a custom image for reproducible builds.
ARG PKG_CACHE=pkg-cache-local

ARG BASE=alpine:3.18.3@sha256:7144f7bab3d4c2648d7e59409f15ec52a18006a128c733fcff20d3a4a54ba44a

FROM ${BASE} AS base

FROM base AS pkg-cache-local-base
ARG PACKAGES
RUN mkdir -p /etc/apk/cache && \
  apk update && \
  apk cache download --available --add-dependencies ${PACKAGES}

FROM scratch AS pkg-cache-local
COPY --from=pkg-cache-local-base /etc/apk/cache /etc/apk/cache

# pkg-cache is the stage to collect package cache files.
# This stage can be pushed for the sake of reproducible builds.
FROM ${PKG_CACHE} AS pkg-cache

FROM base
ARG PACKAGES
RUN \
  --mount=from=pkg-cache,source=/etc/apk/cache,target=/etc/apk/cache,rw \
  --network=none \
  apk add --no-network ${PACKAGES}
# The package signatures are verified by apk
```

Push:
```bash
docker build . -f Dockerfile.alpine \
  --push -t example.com/example-alpine:v1.2.3

docker build . -f Dockerfile.alpine \
  --push -t example.com/example-alpine:v1.2.3-pkg-cache \
  --target pkg-cache
```

Repro:
```bash
docker build . -f Dockerfile.alpine \
  -t example-alpine:v1.2.3 \
  --build-arg PKG_CACHE=example.com/example-alpine:v1.2.3-pkg-cache
```

Other examples:
- [`Dockerfile.archlinux`](./Dockerfile.archlinux): ArchLinux (:warning: Discouraged. See the `WARNING` at the end of the file)
- [`Dockerfile.debian`](./Dockerfile.debian): Debian and Ubuntu (:warning: Discouraged. See the `WARNING` at the end of the file)
- [`Dockerfile.fedora`](./Dockerfile.fedora): Fedora, CentOS Stream, Rocky Linux, and AlmaLinux
- [`Dockerfile.opensuse`](./Dockerfile.opensuse): openSUSE

## Related project
<https://github.com/reproducible-containers/repro-sources-list.sh>
configures `/etc/apt/sources.list` and similar files for installing packages from a past snapshot
like <http://snapshot.debian.org/archive/debian/20230101T000000Z>.

|Project                                                           |Cache location                          |Best for                             |
|------------------------------------------------------------------|----------------------------------------|-------------------------------------|
|<https://github.com/reproducible-containers/repro-sources-list.sh>|Distros' permanent snapshot servers (*1)|Debian, Ubuntu, ArchLinux            |
|<https://github.com/reproducible-containers/repro-pkg-cache>      |Your own permanent image registry       |Alpine, Fedora, Rocky, openSUSE, etc.|

(*1): The packages can be also ephemerally cached on GitHub Actions to reduce loads on distros' snapshot servers.
See <https://github.com/reproducible-containers/buildkit-cache-dance>.

## Alternatives
- There is a proposal by [Tõnis Tiigi](https://github.com/tonistiigi) to extend BuildKit to attach package materials to an image provenance.
  See [his comment on 2023-09-15 in `moby/buildkit#4238`](https://github.com/moby/buildkit/pull/4238#issuecomment-1721859753).
