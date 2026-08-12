# Architecture

## Overview

RunOnceDurationOverride Operator is an OpenShift operator that deploys and manages a mutating admission webhook server. The webhook overrides the `activeDeadlineSeconds` field on pods whose `restartPolicy` is set to `Never` or `OnFailure`, preventing "run-once" pods (like Jobs and build pods) from running indefinitely.

The operand is the [run-once-duration-override](https://github.com/openshift/run-once-duration-override) webhook server, deployed as a DaemonSet on master nodes using host networking.

The operator's primary responsibilities:
- Watch the `RunOnceDurationOverride` CR and reconcile the operand lifecycle
- Deploy and manage the webhook DaemonSet with TLS certificates and configuration
- Generate and rotate self-signed TLS certificates for the webhook
- Observe cluster-level TLS settings and inject them into the webhook
- Manage the MutatingWebhookConfiguration with the correct CA bundle
- Support namespace-level opt-in via labels

## Data Flow

```text
  RunOnceDurationOverride CR (operator.openshift.io/v1)
  (name: cluster, cluster-scoped)
              │
              ▼
  ┌──────────────────────────────────────────────────────┐
  │    TargetConfigController (pkg/operator)              │
  │  (handler chain: Validate → Config → Certs →         │
  │   DaemonSet → Webhook → Availability)                │
  └──────────────────┬───────────────────────────────────┘
                     │
      ┌──────────────┼──────────────────┐
      ▼              ▼                  ▼
  DaemonSet     ServiceAccount     MutatingWebhook
  (webhook      + RBAC             Configuration
   on masters)                     (CA bundle)
      │              ▲
      ▼              │
  Webhook Pod(s)  TLS Certs
  (hostNetwork,   (self-signed,
   port 9448)      1yr validity)
      │
      ▼
  Pods with restartPolicy: Never/OnFailure
  in labeled namespaces get activeDeadlineSeconds set
```

Users create a `RunOnceDurationOverride` CR specifying the desired `activeDeadlineSeconds` value. The operator deploys and manages the webhook as a DaemonSet in the `openshift-run-once-duration-override-operator` namespace. Namespaces must opt in with the label `runoncedurationoverrides.admission.runoncedurationoverride.openshift.io/enabled: "true"`.

## Operator Startup

Entry point: `cmd/run-once-duration-override-operator/main.go` → `pkg/cmd/operator/cmd.go` → `pkg/operator/start.go`.

Startup sequence:
1. Create clients (Kubernetes, dynamic, OpenShift config, operator CR client)
2. Set up informers for the operator namespace and cluster-wide resources
3. Create an `OperatorClientWrapper` (adapts custom CR client to `v1helpers.OperatorClient`)
4. Start three controllers:
   - **ResourceSyncController** — syncs secrets/configmaps between namespaces
   - **ConfigObserver** — observes TLS security profile from the cluster `APIServer` config
   - **TargetConfigController** — the main reconciliation controller (handler chain)
5. Serve health check on port 8080 at `/healthz`
6. Block until context cancellation

The operator uses OpenShift's `library-go` `controllercmd` framework, which provides leader election, health checks, and graceful shutdown.

## Custom Resource

The `RunOnceDurationOverride` CRD (`operator.openshift.io/v1`) is cluster-scoped (short name: `rodoo`) and defines:

- **Spec fields** (embeds `operatorv1.OperatorSpec`):
  - `runOnceDurationOverride.spec.activeDeadlineSeconds` (int64) — the maximum active deadline to enforce on run-once pods
  - Standard operator fields: `managementState`, `logLevel`, `unsupportedConfigOverrides`, `observedConfig`
- **Status fields** (embeds `operatorv1.OperatorStatus` plus custom fields):
  - `conditions[]`, `generations[]`, `observedGeneration`
  - `hash` — tracks hashes for configuration, serving cert, and observed config (change detection)
  - `resources` — references to managed resources (ConfigMap, Service, Secret, DaemonSet, MutatingWebhookConfiguration, etc.)
  - `image` — current operand image
  - `certsRotateAt` — timestamp when TLS certificates should next be rotated

The CR must be named `cluster`.

## TargetConfigController — Handler Chain

`pkg/operator/targetconfigcontroller/` implements the main reconciliation loop using a handler chain pattern. On each sync, handlers execute sequentially:

1. **AvailabilityHandler** (`handler_availability.go`) — checks if the webhook DaemonSet is available and sets the `Available` condition
2. **ValidationHandler** (`handler_validation.go`) — validates the CR spec (e.g., `activeDeadlineSeconds >= 0`); sets `InstallReadinessFailure` condition on invalid input
3. **ConfigurationHandler** (`handler_configuration.go`) — ensures the ConfigMap containing webhook configuration exists and is current
4. **CertGenerationHandler** (`handler_cert_generation.go`) — generates self-signed CA and serving certificates if missing or expiring within 48 hours; stores in a Secret
5. **CertReadyHandler** (`handler_cert_ready.go`) — verifies TLS certificates are populated in the Secret before proceeding
6. **DaemonSetHandler** (`handler_deploy.go`) — constructs and applies the webhook DaemonSet with:
   - Operand image from `RELATED_IMAGE_OPERAND_IMAGE` env var
   - Config hash and cert hash annotations for rolling updates on changes
   - TLS cipher suites and min TLS version from observed config
7. **DeploymentReadyHandler** (`handler_deployment_ready.go`) — checks DaemonSet readiness (desired vs. available pods)
8. **WebhookConfigurationHandler** (`handler_webhook.go`) — ensures MutatingWebhookConfiguration exists with the current CA bundle
9. **AvailabilityHandler** (final) — re-checks availability after all resources are reconciled

Each handler receives an operand context and can set conditions, update status fields, or halt the chain.

## Config Observer

`pkg/operator/configobservation/configobservercontroller/` observes the cluster-level `APIServer` configuration for TLS security profile settings. When the cluster admin changes the TLS profile, the observer updates the operator's `observedConfig`, which triggers the DaemonSet handler to update the webhook container args with the new `--tls-cipher-suites` and `--tls-min-version` values.

## Certificate Management

`pkg/cert/` provides self-signed certificate generation:

- **CA certificate** — self-signed root CA used to sign the serving certificate
- **Serving certificate** — used by the webhook server for TLS
- **Validity:** 365 days (`DefaultCertValidFor`)
- **Rotation threshold:** 48 hours before expiry (`DefaultCertRotateThreshold`)
- **Organization:** `Red Hat, Inc.`
- **Storage:** CA and serving certs stored in a Secret; CA bundle stored in a ConfigMap for injection into MutatingWebhookConfiguration

The operator does not use OpenShift's service-ca-operator because the webhook runs on host networking and uses URL-based webhook client config rather than service references.

## Programmatic Asset Construction

Unlike operators that use embedded YAML templates, all operand Kubernetes resources are constructed programmatically in Go via `pkg/asset/`:

| Resource | Constructor | Purpose |
|----------|-------------|---------|
| DaemonSet | `pkg/asset/` + `pkg/deploy/` | Webhook server on master nodes, hostNetwork on port 9448 |
| ServiceAccount | `pkg/asset/` | Identity for the webhook pods |
| ClusterRole / ClusterRoleBinding | `pkg/asset/` | RBAC for reading namespaces, pods, and webhook configs |
| Service | `pkg/asset/` | Metrics endpoint |
| ConfigMap | `pkg/asset/` | Webhook configuration (activeDeadlineSeconds value) |
| Secret | `pkg/asset/` | TLS certificates (CA + serving) |
| MutatingWebhookConfiguration | `pkg/asset/` | Webhook registration with CA bundle, namespace selector |

The DaemonSet runs `/usr/bin/run-once-duration-override` with `--tls-cert-file`, `--tls-private-key-file`, `--tls-cipher-suites`, `--tls-min-version`, and `--v` (log verbosity) arguments. It uses the `restricted-v2` SCC and `hostNetwork: true`.

## Build System

Uses `build-machinery-go`. Key targets:

| Target | Description |
|--------|-------------|
| `make build` | Build operator binary |
| `make test-unit` | Run unit tests (`./pkg/... ./cmd/...`) |
| `make test-e2e` | Run E2E tests (requires cluster, 3h timeout) |
| `make verify` | Formatting, vetting, golang version checks |
| `make regen-crd` | Regenerate CRD from Go types using `controller-gen` |
| `make generate` | Run all codegen (CRD schema + client generation) |

**Base image:** `ubi9/ubi-minimal`. **Go version:** see `go.mod`.

## Testing

**Unit tests**: Co-located `*_test.go` files in `pkg/operator/`. Coverage includes operator client wrapper, controller startup, handler conditions, and DaemonSet readiness checks.

**E2E tests** (`test/e2e/`): Deployed via the OpenShift Tests Extension (OTE) framework. Tests include:
- Operator pod readiness in `openshift-run-once-duration-override-operator` namespace
- Webhook DaemonSet pod readiness
- MutatingWebhookConfiguration creation
- Pod mutation: creates a pod with `restartPolicy: OnFailure` in a labeled namespace and verifies `activeDeadlineSeconds` is set to the configured value (800 in tests)

The OTE test binary is built alongside the operator and shipped (gzipped) in the operator image at `/usr/bin/run-once-duration-override-operator-tests-ext.gz`.

Run E2E with: `make test-e2e` (requires `KUBECONFIG`, `RELEASE_IMAGE_LATEST`, `NAMESPACE` env vars).

## Namespace

The operator and operand run in `openshift-run-once-duration-override-operator` (constant `OperatorNamespace` in `pkg/operator/operatorclient/`). The operator Deployment, webhook DaemonSet, RBAC, Service, ConfigMap, Secret, and health check all live here.

## Directory Structure

| Directory / File | Purpose |
|-----------------|---------|
| `cmd/run-once-duration-override-operator/` | Main operator entry point |
| `cmd/run-once-duration-override-operator-tests-ext/` | OTE test binary entry point |
| `cmd/testutil/` | JSON/YAML conversion utilities |
| `pkg/apis/runoncedurationoverride/v1/` | RunOnceDurationOverride CRD types |
| `pkg/apis/reference/` | Object reference helper |
| `pkg/asset/` | Programmatic operand resource construction |
| `pkg/cert/` | Self-signed certificate generation and rotation |
| `pkg/cmd/operator/` | Cobra command factory |
| `pkg/deploy/` | DaemonSet deployment logic |
| `pkg/operator/` | Core operator logic (startup, health check) |
| `pkg/operator/targetconfigcontroller/` | Handler chain reconciliation controller |
| `pkg/operator/operatorclient/` | Operator client adapter and v1helpers wrapper |
| `pkg/operator/configobservation/` | TLS config observer |
| `pkg/runtime/` | Operand context, enqueuer, ownership utilities |
| `pkg/generated/` | Auto-generated clientset, informers, listers |
| `pkg/version/` | Build version info |
| `deploy/` | Manual (non-OLM) deployment manifests |
| `manifests/` | OLM bundle manifests (CSV, CRD) |
| `metadata/` | OLM metadata annotations |
| `hack/` | Code generation and helper scripts |
| `test/e2e/` | End-to-end test suite |
| `test/e2e/bindata/` | Embedded YAML assets for e2e deployment |
| `vendor/` | Vendored dependencies (don't modify directly) |

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| `library-go` controller framework | Consistent with other OpenShift operators; provides battle-tested leader election, health checks, config observation |
| Handler chain pattern | Decompose reconciliation into discrete, testable steps with clear ordering; each handler manages one concern |
| Programmatic resource construction | Type-safe resource building in Go via `pkg/asset/`; avoids template parsing but requires Go changes for manifest updates |
| Self-signed certificates | Webhook uses host networking with URL-based client config, which is incompatible with service-ca-operator's service-based cert injection |
| DaemonSet on master nodes with hostNetwork | Ensures webhook is available on every master; host networking avoids service routing complexity for admission webhooks |
| URL-based webhook client config | Required because of host networking — no ClusterIP service to reference |
| Hash-based change detection | Configuration, cert, and observed config hashes stored in CR status trigger DaemonSet updates only when content actually changes |
| Namespace label opt-in | Prevents unintended mutation of pods; critical namespaces (`run-level: 0` or `1`) are always excluded |
| OperatorClientWrapper | Adapts custom CR client to `v1helpers.OperatorClient` interface for library-go compatibility while preserving custom status fields |
| OTE test binary shipped in operator image | Enables CI to extract and run e2e tests without a separate test image |
