# AI Agent Guide for RunOnceDurationOverride Operator

This file provides guidance for AI agents working with the OpenShift RunOnceDurationOverride Operator repository.

## Overview

**What is RunOnceDurationOverride Operator?**
An OpenShift operator that deploys and manages the RunOnceDurationOverride Admission Webhook Server. The webhook is a mutating admission webhook that overrides the `activeDeadlineSeconds` field on pods whose `restartPolicy` is set to `Never` or `OnFailure`, preventing "run-once" pods (like Jobs, build pods, etc.) from running indefinitely.

The operand (the webhook server from [openshift/run-once-duration-override](https://github.com/openshift/run-once-duration-override)) runs as a DaemonSet on master nodes using host networking. Namespaces must opt in via label `runoncedurationoverrides.admission.runoncedurationoverride.openshift.io/enabled: "true"`.

The operator is installed via the Operator Lifecycle Manager (OLM) and reconciles a `RunOnceDurationOverride` CR to deploy and manage the webhook operand. It uses OpenShift's `library-go` controller framework (not `controller-runtime` or `operator-sdk`).

## Build and Test

```bash
make build        # Build all binaries
make test-unit    # Unit tests (pkg/... cmd/...)
make verify       # Formatting, vetting, golang version checks
make test-e2e     # E2E tests (requires cluster, 3h timeout)
make generate     # Run all codegen (CRD schema + client generation)
make regen-crd    # Regenerate CRD from Go types using controller-gen
```

Go version: see `go.mod`.

## Project Structure

| Directory / File | Purpose |
|-----------------|---------|
| `cmd/run-once-duration-override-operator/` | Main operator binary entry point |
| `cmd/run-once-duration-override-operator-tests-ext/` | OpenShift Tests Extension (OTE) binary for e2e tests |
| `cmd/testutil/` | JSON/YAML conversion utilities |
| `pkg/apis/runoncedurationoverride/v1/` | `RunOnceDurationOverride` CRD type definitions |
| `pkg/apis/reference/` | Object reference helper |
| `pkg/asset/` | Programmatic operand resource construction (DaemonSet, Service, RBAC, ConfigMap, Secret, Webhook config) |
| `pkg/cert/` | Self-signed certificate generation and rotation |
| `pkg/cmd/operator/cmd.go` | Cobra command factory, wires `controllercmd` framework |
| `pkg/deploy/` | DaemonSet deployment logic |
| `pkg/operator/start.go` | `RunOperator()` — creates clients, informers, starts all controllers |
| `pkg/operator/targetconfigcontroller/` | Main reconciliation controller with handler chain pattern |
| `pkg/operator/operatorclient/` | Operator client adapter — constants (`OperatorNamespace`, `OperatorName`), custom v1helpers wrapper |
| `pkg/operator/configobservation/` | TLS config observer — watches cluster `APIServer` config for TLS settings |
| `pkg/runtime/` | Operand context, enqueuer, ownership utilities |
| `pkg/generated/` | Auto-generated clientset, informers, listers — **do not modify directly** |
| `pkg/version/version.go` | Build version info |
| `deploy/` | Manual (non-OLM) deployment manifests (numbered `00_` through `09_`) |
| `manifests/` | OLM bundle manifests (CSV, CRD) |
| `metadata/` | OLM metadata annotations |
| `test/e2e/` | E2E test suite — webhook injection, DaemonSet readiness tests |
| `test/e2e/bindata/` | Embedded YAML assets for e2e test deployment |
| `hack/` | Code generation and helper scripts |
| `vendor/` | Vendored dependencies — **do not modify directly** |
| `.tekton/` | Tekton/Konflux CI pipeline definitions |
| `Dockerfile` | CI container image build |
| `Dockerfile.rhel7` | Production container image build (uses RHEL 9 base despite the name) |
| `Makefile` | Build targets |
| `go.mod` | Go module dependencies |

## Controller Pattern

The operator runs three controllers, all wired in `pkg/operator/start.go` via the library-go controller framework:

**`TargetConfigController`** — the main reconciliation controller using a **handler chain pattern**. Each reconciliation executes a series of handlers in order:

1. **AvailabilityHandler** — checks if the webhook DaemonSet is available
2. **ValidationHandler** — validates the CR spec (e.g., `activeDeadlineSeconds >= 0`)
3. **ConfigurationHandler** — ensures webhook ConfigMap exists and is up-to-date
4. **CertGenerationHandler** — generates/rotates self-signed TLS certificates (1-year validity, 48h rotation threshold)
5. **CertReadyHandler** — verifies certificates are populated in the Secret
6. **DaemonSetHandler** — ensures webhook DaemonSet is created/updated with correct image, config hash, cert hash, TLS settings
7. **DeploymentReadyHandler** — checks DaemonSet readiness
8. **WebhookConfigurationHandler** — ensures MutatingWebhookConfiguration exists with correct CA bundle
9. **AvailabilityHandler** (final) — final availability check

**`ConfigObserver`** — watches cluster `APIServer` config and observes TLS security profile changes, updating `observedConfig` on the CR.

**`ResourceSyncController`** — syncs secrets/configmaps between namespaces.

**Key Concepts:**
- **Handler chain:** Each handler gets a context with the current CR, performs its work, and returns a result. Handlers can set conditions, update status, and signal whether reconciliation should continue.
- **Programmatic asset construction:** Unlike operators using embedded YAML templates, all operand resources (DaemonSet, Service, RBAC, etc.) are built programmatically in Go via `pkg/asset/`.
- **Self-signed certificates:** The operator manages its own TLS certificates rather than relying on OpenShift's service-ca-operator.
- **Host networking:** The webhook DaemonSet uses `hostNetwork: true` on port 9448, with the MutatingWebhookConfiguration using a URL client config.

## Key Conventions

- **Namespace:** The operator and operand both run in `openshift-run-once-duration-override-operator`. Constants live in `pkg/operator/operatorclient/client.go`.
- **CR name:** Must be `cluster` (constant `DefaultCR` / `OperatorConfigName`).
- **Operator name:** `runoncedurationoverride` (constant `OperatorName`).
- **Short name:** `rodoo` (CRD short name).
- **Namespace opt-in label:** `runoncedurationoverrides.admission.runoncedurationoverride.openshift.io/enabled: "true"`.
- **Logging:** `k8s.io/klog/v2` with verbosity levels.
- **Error handling:** Wrap with `fmt.Errorf("context: %w", err)`; return errors for retry, return `nil` for non-retriable conditions.
- **CRD changes:** Modify `pkg/apis/runoncedurationoverride/v1/override_types.go`, then run `make regen-crd` and `make generate-clients`.
- **Build tags:** `strictfipsruntime` is set for all Go builds.
- **Owner tracking:** Resources are tracked via `runoncedurationoverride.operator.openshift.io/owner` annotation and OwnerReferences.

## Critical Rules

### DO NOT
1. **Don't modify CRD definitions** in `pkg/apis/runoncedurationoverride/v1/` without understanding backward compatibility implications
2. **Don't modify `vendor/`** — always use `go mod tidy && go mod vendor`
3. **Don't modify `pkg/generated/`** — always use `make generate-clients`
4. **Don't modify `zz_generated.deepcopy.go`** — always use `make generate`
5. **Don't skip `make verify`** before considering work complete
6. **Don't log secrets** — TLS certificates, private keys, or auth tokens must never appear in logs
7. **Don't modify OWNERS files** without explicit direction from maintainers
8. **Don't bypass the handler chain pattern** — add new reconciliation logic as a new handler, not inline in the controller

### DO
1. **Run `make verify`** before submitting any changes
2. **Run `make test-unit`** to ensure tests pass
3. **Use structured logging** via klog with appropriate verbosity levels
4. **Follow Kubernetes API conventions** for CRD status conditions
5. **Handle errors gracefully** and return meaningful error messages
6. **Use the library-go controller factory pattern** — do not introduce controller-runtime controllers
7. **Keep `deploy/` and `test/e2e/bindata/` in sync** — CRD is copied across these locations (use `make regen-crd`)
8. **Document architectural decisions** in ARCHITECTURE.md

## Non-Obvious Internals

- **`controllercmd` framework:** The entry point chain (`cmd/` → `pkg/cmd/operator/` → `pkg/operator/start.go`) passes through library-go's `controllercmd.ControllerCommandConfig`, which handles leader election, signal handling, health checks, and serving info.
- **Handler chain pattern:** The reconciler doesn't use a single `Reconcile()` method. Instead, `pkg/operator/targetconfigcontroller/` defines a chain of handlers that execute sequentially. Each handler receives an operand context and returns a `reconcile.Result`. This is different from the template-based approach used by some other OpenShift operators.
- **Programmatic resource construction:** All operand Kubernetes resources are built in Go code via `pkg/asset/` rather than from embedded YAML templates. This gives type-safe resource construction but means changes to operand manifests require Go code changes.
- **Self-signed certificate management:** The operator generates its own CA and serving certificates (`pkg/cert/`). Certificates are valid for 1 year and rotated 48 hours before expiry. The rotation timestamp is tracked in the CR status (`CertsRotateAt`).
- **Host network webhook:** The webhook DaemonSet runs with `hostNetwork: true` on port 9448. The MutatingWebhookConfiguration uses a `URL` client config (`https://localhost:9448/...`) instead of a service reference, because of host networking.
- **Namespace exclusion:** Namespaces with `openshift.io/run-level` set to `0` or `1` are excluded from webhook mutation, protecting critical control plane namespaces.
- **OperatorClientWrapper:** The custom wrapper in `pkg/operator/operatorclient/wrapper.go` adapts the CR client to implement `v1helpers.OperatorClient`, preserving custom status fields (Hash, Resources, Image, CertsRotateAt) during updates.
- **OTE test binary:** The e2e tests are compiled into a separate binary (`cmd/run-once-duration-override-operator-tests-ext/`) using the OpenShift Tests Extension framework and shipped gzipped inside the operator image. CI extracts and runs it.
- **CRD lives in three places:** `manifests/` is the source of truth. `make regen-crd` copies to `deploy/` and `test/e2e/bindata/assets/`.
- **Hash-based change detection:** The operator stores hashes of configuration, serving cert, and observed config in the CR status. Changes to these hashes trigger DaemonSet updates.

### Updating Dependencies

1. Update `go.mod`: `go get <module>@<version> && go mod tidy`
2. Vendor: `go mod vendor`
3. Verify: `make verify && make test-unit`

## Testing

- **Unit tests:** Co-located `*_test.go` files in `pkg/operator/`, table-driven, run with `make test-unit` or `go test ./pkg/... ./cmd/...`.
- **E2E tests:** `test/e2e/` — deploys the operator to a real cluster, creates a `RunOnceDurationOverride` CR, verifies that pods with `restartPolicy: OnFailure` get `activeDeadlineSeconds` injected. Run with `make test-e2e` (requires cluster, `KUBECONFIG`, `RELEASE_IMAGE_LATEST`, `NAMESPACE` env vars).

## Additional Resources

- [ARCHITECTURE.md](ARCHITECTURE.md) — Complete system design, components, and technical details
- [README.md](README.md) — User-facing documentation and getting started guide
- [OpenShift library-go](https://github.com/openshift/library-go) — Controller factory patterns and operator helpers
- [run-once-duration-override](https://github.com/openshift/run-once-duration-override) — The webhook server operand
- [Kubernetes Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) — Mutating admission webhook documentation
