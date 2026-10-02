# iperf3

Container images with [iperf3](https://software.es.net/iperf/), ESnet's tool for measuring TCP and UDP throughput between two hosts. iperf3 is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when ESnet publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by ESnet or Lawrence Berkeley National Laboratory, which develop iperf3. Report problems with the image in this repository and iperf3 bugs at [esnet/iperf](https://github.com/esnet/iperf/issues).

## Quick start

Start a server:

```sh
docker run --rm --network host ghcr.io/randomcontainers/iperf3 -s
```

On another host, run a 10-second TCP test against it, with the server's name or IP address in place of `<server>`:

```sh
docker run --rm --network host ghcr.io/randomcontainers/iperf3 -c <server>
```

The entrypoint runs `iperf3` under `tini`, so iperf3's options go after the image name. A few more commands:

```sh
# Measure the other direction, from the server to this host
docker run --rm --network host ghcr.io/randomcontainers/iperf3 -c <server> -R

# UDP at 500 Mbit/s, which also reports jitter and packet loss
docker run --rm --network host ghcr.io/randomcontainers/iperf3 -c <server> -u -b 500M

# Four parallel streams for 30 seconds, with the results as JSON
docker run --rm --network host ghcr.io/randomcontainers/iperf3 -c <server> -P 4 -t 30 -J > result.json
```

The [iperf3 documentation](https://software.es.net/iperf/invoking.html) explains the options, and running the image without arguments prints the ones its version has. Run the same iperf3 version on both hosts. `3.21` also follows 3.21.x patch releases, so pin a digest when results must stay comparable over time.

## What is in the image

- `iperf3` in `/usr/local/bin`. Its library, libiperf, is linked in statically, so apart from the distro's C library it loads only OpenSSL.
- Support for [authentication](#authentication), which needs OpenSSL, and for UDP GSO and GRO (`--gsro`). `iperf3 --version` lists the optional features that were compiled in.

Not included: SCTP (`--sctp`), the libiperf library and its header, and the man pages. The configure flags are in `/usr/local/share/randomcontainers/iperf3/buildinfo`.

## Default or slim

iperf3's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is an iperf3 release such as `3.21`, or `3.21.1` for a patch release. `<minor>` and `<major>` are its shorter forms, `3.21` and `3`, and follow the newest release in that series. For a release without a patch number, such as `3.21`, `<version>` and `<minor>` are the same tag. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current iperf3 version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

iperf3 reads or writes files only for options that name one, such as `--logfile`, `-F` and the key and user files for authentication. Mount the directory that holds them:

```sh
docker run --rm --network host --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/iperf3 -c <server> -J --logfile result.json
```

## Networking

With `--network host` the container uses the host's network interfaces, so iperf3 measures the host's own path. On Docker's default bridge network, publish the server's port instead. iperf3 uses TCP port 5201 for the control connection and TCP tests, and UDP port 5201 for UDP tests:

```sh
docker run --rm -p 5201:5201 -p 5201:5201/udp ghcr.io/randomcontainers/iperf3 -s
```

Traffic to a published port passes through Docker's NAT and a virtual Ethernet pair, which costs CPU time per packet and can lower the result on fast links. Use `--network host` when the numbers should describe the host. On Docker Desktop, containers run in a virtual machine, so the results include that machine's network path even with `--network host`. The client needs no published ports, because it opens every connection, also in reverse mode (`-R`).

A server started with `-s` runs a test for any client that can reach its port, until you stop it. Pass `-1` to exit after one test, `-B <address>` to listen on one address only, or require a login as described below.

Run a long-lived server with `docker run -d` rather than iperf3's `-D`, which sends iperf3 to the background and makes the container exit. Because the image runs as UID 1000, a server on the host network can't use a port below 1024 unless you add `--user 0:0`.

## Authentication

A server can require a username and password. The client encrypts them with the server's RSA public key, and the server checks them against a file of SHA-256 hashes. Only the credentials are encrypted, not the test traffic. Create the key pair and the users file on the host:

```sh
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
printf 'alice,%s\n' "$(printf '%s' '{alice}<password>' | openssl dgst -sha256 -r | cut -d ' ' -f 1)" > users.csv
```

Start the server with the private key and the users file:

```sh
docker run --rm --network host --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/iperf3 -s --rsa-private-key-path private.pem --authorized-users-path users.csv
```

The client reads the password from `IPERF3_PASSWORD`, or asks for it when the container runs with `-it`:

```sh
docker run --rm --network host --user "$(id -u):$(id -g)" -v "$PWD:/work" -e IPERF3_PASSWORD \
  ghcr.io/randomcontainers/iperf3 -c <server> --username alice --rsa-public-key-path public.pem
```

The private key must not have a passphrase. The clocks of the two hosts must agree within 10 seconds, or the server needs a larger `--time-skew-threshold`.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new iperf3 release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/iperf3:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends iproute2 \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

`iproute2` adds `ip` and `ss`, for looking at interfaces and open connections next to a test. On Alpine, start from `slim-alpine` and use `apk add --no-cache iproute2`. The entrypoint is `["tini", "--", "iperf3"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new iperf3 releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/iperf3/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages iperf3 needs at run time. On Ubuntu the list is empty, because the base image already has OpenSSL.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/iperf3:latest \
  --repo randomcontainers/iperf3 --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/iperf3:latest --format '{{ json .SBOM }}'
```

ESnet publishes a SHA-256 checksum with each release but no signature. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`. After compiling, it runs iperf3's own test suite, which includes a round trip through the authentication code.

## Updates

The project checks the releases of [esnet/iperf](https://github.com/esnet/iperf/releases) every 15 minutes and follows the 3.x series. A release is picked up once it is 24 hours old. Its tarball is checked against the `.sha256` file ESnet publishes on downloads.es.net and the digest GitHub recorded for it, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t iperf3:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

iperf3 is licensed under a three-clause BSD license from the University of California, through Lawrence Berkeley National Laboratory, with an added paragraph about improvements you choose to share (SPDX `BSD-3-Clause-LBNL`). Some of its source files come from other projects and keep their own terms:

- MIT: cJSON (`src/cjson.c`), and the code from Russ Cox's libtask in `src/net.c` and `src/net.h`.
- A permissive notice from Lucent Technologies, which has no SPDX identifier: the older library that the libtask code in `src/net.c` and `src/net.h` builds on.
- BSD-3-Clause: `src/queue.h` from the University of California, and the code by Eric Jackson in `src/net.c`.
- BSD-2-Clause: the DSCP name table in `src/dscp.c`, from OpenSSH.
- NCSA: the unit conversion and message code from iperf 2 in `src/units.c` and `src/iperf_locale.c`, from the University of Illinois.
- Public domain: `src/portable_endian.h`.

All of these are compiled into `iperf3`. iperf3's `LICENSE` file has every notice above except the one in `src/dscp.c`, which the build copies to `COPYING.dscp`. Both files are in `/usr/local/share/randomcontainers/iperf3/licenses/`, and the source tarball URL is in `/usr/local/share/randomcontainers/iperf3/source`. The image's license label is `BSD-3-Clause-LBNL AND BSD-2-Clause AND BSD-3-Clause AND MIT AND NCSA`. The Ubuntu and Alpine packages in the image keep their own licenses, for example Apache-2.0 for OpenSSL. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
