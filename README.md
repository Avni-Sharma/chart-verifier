# chart-verifier

`chart-verifier` is a tool that certifies a Helm chart against a configurable list of checks; those checks can be
whitelisted or blacklisted through command line options.

Each check is independent and order is not and will not be guaranteed, and its input will be informed through options in
the command line interface; currently the only input is the required `uri` option.

The following checks have been implemented:

| Name | Description
|---|---
| `is-helm-v3` | Checks whether the given `uri` is a Helm v3 chart.
| `has-readme` | Checks whether the Helm chart contains a `README.md` file.
| `contains-test` | Checks whether the Helm chart contains at least one test file.
| `has-minkubeversion` | Checks whether the Helm chart's `Chart.yaml` includes the `minKubeVersion` field.
| `readme-contains-values-schema` | Checks whether the Helm chart `README.md` file contains a `values` schema section.
| `not-contains-crds` | Check whether the Helm chart does not include CRDs.

The following checks are being implemented and/or considered:

| Name | Description
|---|---
| `keywords-are-openshift-categories` | Checks whether the Helm chart's `Chart.yaml` file includes keywords mapped to OpenShift categories.
| `is-commercial-chart` | Checks whether the Helm chart is a Commercial chart.
| `is-community-chart` | Checks whether the Helm chart is a Community chart.
| `not-contains-infra-plugins-and-drivers` | Check whether the Helm chart does not include infra plugins and drivers (network, storage, hardware, etc)
| `can-be-installed-without-manual-prerequisites` |
| `can-be-installed-without-cluster-admin-privileges` |

## Architecture

This tool is part of a larger process that aims to certify Helm charts, and its sole responsibility is to ingest a Helm
chart URI (`file://`, `https?://`, etc)
and return either a *positive* result indicating the Helm chart has passed all checks, or a *negative* result indicating
which checks have failed and possibly propose solutions.

The application is separated in two pieces: a command line interface and a library. This is handy because the command
line interface is specific to the user interface, and the library can be generic enough to be used to, for example,
inspect Helm chart bytes in flight.

One positive aspect of the command line interface specificity is that its output can be tailored to the methods of
consumption the user expects; in other words, the command line interface can be programmed in such way it can be
represented as either *YAML* or *JSON* formats, in addition to a descriptive representation tailored to human actors.

The interpretation of what is considered a certified Helm chart depends on which checks the chart has been submitted to,
so this information should be present in the certificate as well.

Primitive functions to manipulate the Helm chart should be provided, since most checks involve inspecting the contents
of the chart itself; for example, whether a `README.md` file exists, or whether `README.md` contains the `values`'
specification, implicating in offering a cache API layer is required to avoid downloading and unpacking the charts for
each test.

## Getting chart-verifier

Available is currently available in Quay: https://quay.io/repository/redhat-certification/chart-verifier.

## Building chart-verifier

To build `chart-verifier` locally, please execute `hack/build.sh` or its PowerShell alternative.

To build `chart-verifier` container image, please execute `hack/build-image.sh` or its PowerShell alternative:

```text
PS C:\Users\igors\GolandProjects\chart-verifier> .\hack\build-image.ps1
[+] Building 15.1s (15/15) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                 0.0s
 => => transferring dockerfile: 32B                                                                                                                                                                                                                                  0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                                                    0.0s
 => => transferring context: 2B                                                                                                                                                                                                                                      0.0s
 => [internal] load metadata for docker.io/library/fedora:31                                                                                                                                                                                                         1.4s
 => [internal] load metadata for docker.io/library/golang:1.15                                                                                                                                                                                                       1.3s
 => [build 1/7] FROM docker.io/library/golang:1.15@sha256:d141a8bca046ade2c96f89e864cd31f5d0ba88d5a71d62d59e0e1f2ecc2451f1                                                                                                                                           0.0s
 => CACHED [stage-1 1/2] FROM docker.io/library/fedora:31@sha256:ba4fe6a3da48addb248a16e8a63599cc5ff5250827e7232d2e3038279a0e467e                                                                                                                                    0.0s
 => [internal] load build context                                                                                                                                                                                                                                    0.5s
 => => transferring context: 43.06MB                                                                                                                                                                                                                                 0.5s
 => CACHED [build 2/7] WORKDIR /tmp/src                                                                                                                                                                                                                              0.0s
 => CACHED [build 3/7] COPY go.mod .                                                                                                                                                                                                                                 0.0s
 => CACHED [build 4/7] COPY go.sum .                                                                                                                                                                                                                                 0.0s
 => CACHED [build 5/7] RUN go mod download                                                                                                                                                                                                                           0.0s
 => [build 6/7] COPY . .                                                                                                                                                                                                                                             0.2s
 => [build 7/7] RUN ./hack/build.sh                                                                                                                                                                                                                                 12.5s
 => [stage-1 2/2] COPY --from=build /tmp/src/out/chart-verifier /app/chart-verifier                                                                                                                                                                                  0.1s
 => exporting to image                                                                                                                                                                                                                                               0.2s
 => => exporting layers                                                                                                                                                                                                                                              0.2s
 => => writing image sha256:7302e88a2805cb4be1b9e130d057bd167381e27f314cbe3c28fbc6cb7ee6f2a1                                                                                                                                                                         0.0s
 => => naming to quay.io/redhat-certification/chart-verifier:07e369d
```

The container image created by the build program is tagged with the commit ID of the working directory at the time of
the build: `quay.io/redhat-certification/chart-verifier:0d3706f`.

This container image can then be executed with the Docker client as `docker run -it --rm quay.io/redhat-certification/chart-verifier:0d3706f certify`,
like in the example below:

```text
PS C:\Users\igors\GolandProjects\chart-verifier> docker run -it --rm quay.io/redhat-certification/chart-verifier:b274864 certify --help
Certifies a Helm chart by checking some of its characteristics

Usage:
  chart-verifier certify <chart-uri> [flags]

Flags:
  -x, --disable strings   all checks will be enabled except the informed ones
  -e, --enable strings    only the informed checks will be enabled
  -h, --help              help for certify
  -o, --output string     the output format: default, json or yaml

Global Flags:
      --config string   config file (default is $HOME/.chart-verifier.yaml)
```

To verify a chart on the host system, the directory containing the chart should be mounted in the container; for http or
https verifications, no mounting is required:

```text
PS C:\Users\igors\GolandProjects\chart-verifier> docker run -it --rm quay.io/redhat-certification/chart-verifier:07e369d certify https://github.com/isutton/helmcertifier/blob/master/pkg/chartverifier/checks/chart-0.1.0-v3.valid.tgz?raw=true
chart: chart
version: 1.16.0
ok: true

is-helm-v3:
        ok: true
        reason: API version is V2 used in Helm 3
contains-test:
        ok: true
        reason: Chart test files exist
contains-values:
        ok: true
        reason: Values file exist
contains-values-schema:
        ok: true
        reason: Values schema file exist
has-minkubeversion:
        ok: true
        reason: Minimum Kubernetes version specified
not-contains-crds:
        ok: true
        reason: Chart does not contain CRDs
helm-lint:
        ok: true
        reason: Helm lint successful
has-readme:
        ok: true
        reason: Chart has README
```

## Usage

To certify a chart against all available checks:

```text
> chart-verifier ./chart.tgz
> chart-verifier ~/src/chart
> chart-verifier https://www.example.com/chart.tgz
```

To apply only the `is-helm-v3` check:

```text
> chart-verifier --enable is-helm-v3 https://www.example.com/chart.tgz
```

To apply all checks except `is-helm-v3`:

```text
> chart-verifier --disable is-helm-v3 https://www.example.com/chart.tgz
```

