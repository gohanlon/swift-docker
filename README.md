# gohanlon/swift-docker

> [!IMPORTANT]
> * For Ubuntu 20.04 (focal) images, use the official [Swift Docker Hub repository](https://hub.docker.com/_/swift/).
> * Starting with Swift 5.7.2, the official [Swift Docker Hub repository](https://hub.docker.com/_/swift/) includes images for Ubuntu 22.04 (jammy).
> * If you need both Ubuntu 22.04 (jammy) and Swift 5.6.2, only then consider trying these images: [https://hub.docker.com/r/gohanlon/swift](https://hub.docker.com/r/gohanlon/swift)

# Overview

Apple provides official Docker images for:

* [Swift 5.6.2 on Ubuntu 20.04 (focal)](https://github.com/apple/swift-docker/tree/main/5.6/ubuntu/20.04)
* [Swift nightly-main on Ubuntu 22.04 (jammy)](https://github.com/apple/swift-docker/tree/main/nightly-main/ubuntu/22.04)

While remaining as similar as possible to the official Swift images, this repo
provides unofficial images for:

* [Swift 5.6.2 on Ubuntu 22.04 (jammy)](https://github.com/gohanlon/swift-docker/tree/main/5.6/ubuntu/22.04)

**Important note:** These images install the Swift toolchain that's intended for
Ubuntu 20.04 into a 22.04 image. In my short experience using these images, this
works. Please keep in mind that this oddity may have unintended side effects.

## Usage example

``` Dockerfile
FROM gohanlon/swift:5.6.2-jammy as build

RUN apt-get --fix-missing update
RUN apt-get install -y cmake libpq-dev libssl-dev libz-dev openssl
RUN apt-get install --no-install-recommends libsqlite3-dev

# Fix:
#  /build/.build/checkouts/swift-nio-ssl/Sources/CNIOBoringSSL/include/CNIOBoringSSL_base.h:500:10:
#  fatal error: 'memory' file not found
#
# See also: https://github.com/apple/swift-nio-ssl/issues/105
# Note: installing build-essential brings in libstdc++-11-dev. We already have libstdc++-9-dev from the base image.
RUN apt-get install -y build-essential

WORKDIR /build

COPY Package.swift .
RUN swift package resolve
COPY Sources ./Sources
COPY Tests ./Tests

RUN swift build \
  --configuration release \
  --product server-vapor \
  -Xswiftc -g

FROM gohanlon/swift:5.6.2-jammy-slim

RUN apt-get --fix-missing update \
  && apt-get install -y libpq-dev libsqlite3-dev libssl-dev libz-dev openssl \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /run

COPY --from=build /build/.build/release /run

EXPOSE 8080
ENTRYPOINT ["./Run"]
CMD ["serve", "--env", "production", "--hostname", "0.0.0.0", "--port", "8080"]
```

# Quick reference

See the official Swift images: https://hub.docker.com/_/swift/

# Supported tags and respective Dockerfile links

* [5.6.2-jammy](https://github.com/gohanlon/swift-docker/blob/main/5.6/ubuntu/22.04/Dockerfile)
* [5.6.2-jammy-slim](https://github.com/gohanlon/swift-docker/blob/main/5.6/ubuntu/22.04/slim/Dockerfile)

# Compared to the official Swift Dockerfiles

## `swift:5.6.2-focal` vs. `gohanlon/swift:5.6.2-jammy`:

``` sh
% diff \
<(curl -s https://raw.githubusercontent.com/apple/swift-docker/main/5.6/ubuntu/20.04/Dockerfile) \
5.6/ubuntu/22.04/Dockerfile
```

``` diff
1,2c1,2
< FROM ubuntu:20.04
< LABEL maintainer="Swift Infrastructure <swift-infrastructure@forums.swift.org>"
---
> FROM ubuntu:22.04
> LABEL org.opencontainers.image.source https://github.com/gohanlon/swift-docker
```

## `swift:5.6.2-focal-slim` vs. `gohanlon/swift:5.6.2-jammy-slim`:

``` sh
% diff \
<(curl -s https://raw.githubusercontent.com/apple/swift-docker/main/5.6/ubuntu/20.04/slim/Dockerfile) \
5.6/ubuntu/22.04/slim/Dockerfile
```

``` diff
1,2c1,2
< FROM ubuntu:20.04
< LABEL maintainer="Swift Infrastructure <swift-infrastructure@forums.swift.org>"
---
> FROM ubuntu:22.04
> LABEL org.opencontainers.image.source https://github.com/gohanlon/swift-docker
```

## `swiftlang/swift:nightly-main-focal` vs. `swiftlang/swift:nightly-main-jammy`:

Also, the above changes similar to the differences in the official Swift
`nightly-main` builds between Ubuntu 20.04 and 22.04, that is:

``` sh
% diff \
<(curl -s https://raw.githubusercontent.com/apple/swift-docker/main/nightly-main/ubuntu/22.04/Dockerfile) \
<(curl -s https://raw.githubusercontent.com/apple/swift-docker/main/nightly-main/ubuntu/20.04/Dockerfile)
```

``` diff
1c1
< FROM ubuntu:22.04
---
> FROM ubuntu:20.04
32c32
< ARG OS_MAJOR_VER=22
---
> ARG OS_MAJOR_VER=20
```

# How these images were built

## Preface

Ensure you've authenticated to Docker Hub:

``` bash
% docker login
```

Create a Docker [buildx
context](https://docs.docker.com/engine/reference/commandline/buildx_create/):

``` bash
% docker buildx create mycontext1
```

Docker produces rather unhelpful errors when the build cache gets too large,
e.g.:

```
> [linux/amd64 2/4] RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true && apt-get -q update &&     apt-get -q install -y     binutils     git     gnupg2     libc6-dev     libcurl4-openssl-dev     libedit2     libgcc-9-dev     libpython3.8     libsqlite3-0     libstdc++-9-dev     libxml2-dev     libz3-dev     pkg-config     tzdata     zlib1g-dev     && rm -r /var/lib/apt/lists/*:
#0 3.558   At least one invalid signature was encountered.
#0 3.606 Reading package lists...
#0 3.718 W: GPG error: http://security.ubuntu.com/ubuntu jammy-security InRelease: At least one invalid signature was encountered.
#0 3.718 E: The repository 'http://security.ubuntu.com/ubuntu jammy-security InRelease' is not signed.
[...]
```

Prune your preexisting build context to prevent or resolve this error (and
likely countless other similarly unintuitive errors):

``` bash
% docker builder prune
```

## Build and push to Docker Hub

``` bash
% export DOCKERHUB_USER=gohanlon
% docker buildx build \
  --progress=plain \
  --platform linux/arm64/v8,linux/amd64 \
  -t $DOCKERHUB_USER/swift:5.6.2-jammy\
  --push \
  5.6/ubuntu/22.04 \
  && docker buildx build \
    --progress=plain \
    --platform linux/arm64/v8,linux/amd64 \
    -t $DOCKERHUB_USER/swift:5.6.2-jammy-slim \
    --push \
    5.6/ubuntu/22.04/slim
```
