# reference-service

The reference service of the [homelab platform](https://github.com/Karppimc/homelab-platform): a small Go HTTP service created from [service-template](https://github.com/Karppimc/service-template) and deployed by Argo CD. It exists to show, and to keep proving, that a service built from the template runs on the platform without any extra work: no Kubernetes manifests, no policy exceptions.

## What it does

| Endpoint | Purpose |
|---|---|
| `/` | JSON with the service name, the version (the commit the image was built from) and the pod that answered |
| `/healthz` | Liveness probe |
| `/readyz` | Readiness probe; fails during startup and shutdown |
| `/metrics` | Prometheus metrics: request count and duration by route, Go runtime and process metrics |

## How it reaches the cluster

1. A pull request is merged to `main`. CI tests the code, builds the image, runs it read-only as a non-root user, scans it with Trivy and pushes it to `ghcr.io/karppimc/reference-service:<commit SHA>`.
2. A second pull request sets `image.tag` in [`deploy/values.yaml`](deploy/values.yaml) to that commit SHA. This is the release.
3. Argo CD on the platform renders the platform's service chart with `deploy/values.yaml` and rolls the new version out into the namespace `reference`.

The Deployment, Service and ServiceMonitor are not in this repository: the platform's chart renders them with the labels, security context and probes its policies require. The CI job `deploy` runs the platform's own check on `deploy/values.yaml`, so a value the platform would refuse fails here, in the pull request.

## Develop

```bash
go test ./...
go run .                      # http://localhost:8080

docker build --build-arg VERSION=dev -t reference-service:dev .
docker run --rm --read-only --cap-drop ALL --user 65532:65532 -p 8080:8080 reference-service:dev
```

## CI

| Job | What it checks |
|---|---|
| `test` | `gofmt`, `go vet`, `go test` |
| `image` | Build, smoke test, Trivy scan (fails on HIGH and CRITICAL vulnerabilities that have a fix); pushes the image on `main` |
| `deploy` | The platform's chart schema, Kubernetes schemas and Kyverno policies against `deploy/values.yaml` |
| `validate` | Passes only when the jobs above passed; the check the branch ruleset requires |
