---

# Istio – Service Mesh

---

### Component Summary

* **Title:** Istio service mesh

* **Context:**
  Istio is the platform-wide service mesh providing secure, observable and controllable L4–L7 traffic between workloads. It standardizes ingress and egress, enables zero-trust (mTLS, AuthN/Z), and provides common observability and traffic-management primitives for all tenants.

* **Purpose of component:**
  `"Platform functionality" | "Platform stability" | "Platform operations" | "Platform observability"`

* **Design Owners:**
  `{{ Platform Service Mesh Owner }}, {{ Platform Network/Security Owner }}`

* **On-call:**
  Confluence escalation page: `<link to Istio on-call and escalation runbook>`

* **Stakeholders:**
  `{{ Platform DEV/SRE | Security | Networking | Tenant X }}`

* **Customers:**
  Platform services, Platform Development teams, Platform SRE, Tenant Dev teams, Tenant clients (internal users and calling systems).

* **Status:**
  `{{ PoC | UAT | PROD | Decommission }}` – *set per environment; main document assumes PROD*.

* **Doc Version:** `{{ v1.0.0 }}`

* **Component Version (optional):** `{{ Istio <major.minor.patch> }}`

* **Last reviewed:** `YYYY-MM-DD`

* **Next review due to:** `YYYY-MM-DD` (recommended: every 6 months or after any major Istio/Kubernetes upgrade).

* **Links:**

  * Git Repo URLs:

    * `{{ https://git…/platform/istio-control-plane.git }}`
    * `{{ https://git…/platform/istio-gateways.git }}`
  * CI/CD (optional):

    * `{{ https://ci…/platform/istio-pipelines }}`
  * Dashboards (Grafana/Kiali/Tracing):

    * `{{ https://grafana…/d/istio-control-plane }}`
    * `{{ https://grafana…/d/istio-gateways }}`
    * `{{ https://kiali… }}`
    * `{{ tracing UI (Jaeger/Tempo/Zipkin) }}`
  * Alerts:

    * `{{ https://alertmanager…/#/alerts?search=istio }}`
  * Runbooks:

    * `{{ Istio – Ingress 5xx Runbook }}`
    * `{{ Istio – Control-plane degraded Runbook }}`
    * `{{ Istio – Egress policy changes Runbook }}`
  * ArgoCD Apps:

    * `{{ DEV-argocd-app-url / istio-dev }}`
    * `{{ UAT-argocd-app-url / istio-uat }}`
    * `{{ PROD-argocd-app-url / istio-prod }}`

---

### Design Document Lifecycle

* **Purpose of document**
  This document describes the design, deployment and operation of the Istio service mesh on the platform. It is intended for Platform DEV/SRE, Security, and Tenant teams who need to understand how to use, extend, or troubleshoot the mesh.

* **Change process**
  All changes to this document are done via Pull Requests to the `docs/` folder of the Istio Git repository with mandatory reviewers: `<list of roles/names, e.g. "Service Mesh Owner, Security Architect, Platform SRE">`.

* **Frequency of validation**
  The design must be reviewed at least every `<N>` months (recommended: every 6 months) and after any major Istio or Kubernetes upgrade. The responsible owner is `<role/person, e.g. "Service Mesh Product Owner">`.

* **Criteria for the document’s status**

  * *Draft* – content under active development, not yet reviewed.
  * *In-Review* – document is under review.
  * *Approved* – design signed off by required stakeholders.
  * *Active* – implemented and used in production according to this design.
  * *Deprecated* – Istio decommission is planned and a sunset plan exists.

* **Changelog**
  Bullet list of key changes with dates, e.g.:

  * `2025-01-10 – v1.0.0 – Initial Istio control-plane + ingress/egress + Kiali + East-West design`
  * `…`

---

## 1. What are we building and for whom? (Business goals)

* **Summary**

  Istio provides a standardized service mesh layer for all Kubernetes workloads across bare-metal and cloud clusters. It offers:

  * mTLS between services, with workload identity based on Kubernetes ServiceAccounts.
  * Centralized ingress and egress via Envoy gateways.
  * Traffic management (routing, retries, timeouts, circuit-breaking, canaries).
  * Observability via telemetry (metrics, logs, distributed tracing) and UI (Kiali).
  * Policy enforcement (AuthN/Z and, potentially, fine-grained traffic policies).

* **Primary users**

  * Platform DEV/SRE teams operating shared platform services.
  * Tenant DEV teams deploying microservices onto the platform.
  * Security / Network teams defining traffic and egress policies.
  * Observability / Operations teams using metrics, traces and Kiali to diagnose issues.

* **Business goals, success metrics (SLI/SLO), acceptance criteria**

  Example SLOs (adjust to your platform):

  * **Control-plane availability:**

    * SLI: `istiod` Deployment availability in PROD.
    * SLO: ≥ 99.9% per month.
  * **Ingress success rate:**

    * SLI: HTTP success rate (2xx+3xx) for ingress traffic (per critical service).
    * SLO: ≥ 99.5% per month.
  * **Mesh mTLS coverage:**

    * SLI: percentage of intra-mesh traffic using mutual TLS.
    * SLO: ≥ 99% of traffic encrypted.
  * **Config safety:**

    * No more than `N` production incidents per quarter attributed to invalid Istio configuration.

  Acceptance criteria include:

  * Tenants can onboard services with sidecar injection through a documented process.
  * Standard ingress patterns exist (HTTP(S) with OIDC, TCP where needed).
  * Egress is controlled via policies (ServiceEntry + AuthorizationPolicy / external DNS allow lists).
  * Kiali and dashboards are available in all PROD clusters.

---

## 2. Design

### 2.1 High-level architecture (within a cluster)

**Namespaces**

* `istio-d` – Istio control-plane:

  * `istiod` Deployment(s)
  * Istio telemetry addons (only if installed here: `prometheus`, `istio-telemetry`, etc.)
  * CRDs (installed cluster-wide)
* `sm-gateways` – Envoy gateway components:

  * `istio-ingressgateway` Deployment + Service
  * `istio-egressgateway` Deployment + Service
  * (Optional) `istio-eastwestgateway` for cross-cluster traffic

**Control plane vs data plane**

* **Control plane (istio-d)**

  * `istiod` manages:

    * Service discovery (Kubernetes Services/endpoints)
    * xDS configuration for Envoy sidecars and gateways
    * Certificate issuance (workload mTLS certificates via Istio CA)
    * Pilot, Galley and typical pilot functions (config distribution, validation)
  * CRDs:

    * `Gateway`, `VirtualService`, `DestinationRule`
    * `ServiceEntry`, `Sidecar`
    * `EnvoyFilter`
    * `PeerAuthentication`, `RequestAuthentication`, `AuthorizationPolicy`
    * (Optionally) Gateway API `GatewayClass`, `HTTPRoute`, etc., if adopted.

* **Data plane (per namespace + sm-gateways)**

  * **Sidecar proxies**: Envoy containers injected into tenant Pods via auto-injection (namespace label `istio-injection=enabled` or sidecar-injection webhook). They:

    * Terminate and initiate mTLS.
    * Apply traffic rules (routing, retry, timeout, fault injection).
    * Collect telemetry (metrics, access logs, traces).

  * **Ingress Gateway (`istio-ingressgateway`)**:

    * Namespace: `sm-gateways`.
    * Exposed externally via `Service` of type `LoadBalancer` or `NodePort` + external L4 load balancer (F5/HAProxy/MetalLB/nginx, depending on bare-metal design).
    * Owns external IPs / hostnames for tenant applications.

  * **Egress Gateway (`istio-egressgateway`)**:

    * Namespace: `sm-gateways`.
    * Used for controlled outbound traffic to external systems (DBs, SaaS, APIs).
    * Egress policies route traffic via `ServiceEntry` + `VirtualService`.

  * **East-West gateway (optional now, required for multi-cluster)**:

    * Namespace: `sm-gateways`.
    * Exposes mesh services between clusters over secure mTLS links.

**Textual “diagram”**

For a single cluster, logical data flow:

1. Client → external L4 LB (bare-metal LB / cloud LB).
2. LB → `istio-ingressgateway` Service (`sm-gateways` namespace).
3. `istio-ingressgateway` Envoy → target service sidecars using `VirtualService` + `DestinationRule`.
4. Service A sidecar → Service B sidecar inside mesh via mTLS.
5. Service sidecar → `istio-egressgateway` (if egress policy requires) → external destination (SaaS, DBs, etc.).
6. For multi-cluster: Service sidecar in cluster A → `istio-eastwestgateway` A → `istio-eastwestgateway` B → sidecar in cluster B.

If you later want a concrete diagram, you can render this as:

* One box per namespace.
* Control-plane box (`istio-d`) at the top.
* Data-plane boxes below per tenant namespace, plus `sm-gateways` with ingress/egress/east-west gateways.
* Arrows representing xDS config from `istiod` to Envoy instances; data flow arrows between clients, gateways and workloads.

---

### 2.2 Dependencies

#### 2.2.1 In-cluster dependencies

Critical in-cluster services Istio depends on:

* **Kubernetes core components**

  * API server and etcd (CRDs, config storage).
  * DNS (`kube-dns` / `CoreDNS`) for service discovery.
* **CNI plugin**

  * Must support the iptables/envoy interception model used by Istio or Istio CNI.
* **Cert manager / PKI** (if used)

  * Optionally issues root/intermediate CAs for Istio.
* **Gateway API CRDs** (if adopted)

  * `gateway.networking.k8s.io/*` resources.
* **Observability stack**

  * Prometheus scraping Envoy and `istiod` metrics.
  * Grafana for dashboards.
  * Jaeger/Tempo/Zipkin for tracing.
  * Loki/ELK for logs (Envoy access logs, `istiod` logs).
* **Kiali**

  * UI for Istio configuration and traffic topology.
* **Policy engines (optional but recommended)**

  * OPA/Gatekeeper or Kyverno for enforcing policy on Istio CRDs (e.g., no wildcard hosts, restricted external egress, etc.).

#### 2.2.2 Out-of-cluster dependencies

External systems used by Istio and by mesh traffic:

> Example table – adjust to your environment.

| Service / Resource | Provider       | Network Path                     | AuthN/AuthZ                                  | SLA   | Backup option                |
| ------------------ | -------------- | -------------------------------- | -------------------------------------------- | ----- | ---------------------------- |
| Internal PKI / CA  | Security / PKI | Interconnect / FW allow          | mTLS with client certs                       | 99.9% | PKI’s own backup process     |
| OIDC IdP           | IAM / SSO      | Egress via `istio-egressgateway` | OIDC/JWT                                     | 99.9% | n/a (IdP managed externally) |
| Secrets backend    | e.g. GSM/Vault | Egress via `istio-egressgateway` | IAM / token                                  | 99.9% | Read-only mode in DR region  |
| SMTP relay         | `smtp.company` | Egress / FW allow from egress GW | TLS + credentials in Secret / ExternalSecret | 99.9% | Secondary SMTP relay         |
| Central logging    | ELK / Loki     | Egress via gateway or direct     | mTLS / token                                 | 99.5% | Clustered; backup snapshots  |

---

### 2.3 Data flows & ports

**Ingress (North-South)**

* External client → external LB → `istio-ingressgateway` (sm-gateways):

  * Ports (example):

    * `80/TCP` – HTTP
    * `443/TCP` – HTTPS (TLS passthrough or termination at Envoy)
    * `15443/TCP` – mTLS termination (SNI routing for cross-mesh traffic)
* `istio-ingressgateway` → service sidecars:

  * mTLS on cluster-internal ports (service ports, typically 80/8080/…)
* Kiali, tracing, Prometheus access:

  * Usually internal only, via cluster ingress domains.
  * Ports: 20001 (Kiali), 16686 (Jaeger), etc., depending on deployment.

**Egress**

* Service sidecar → `istio-egressgateway` (sm-gateways) for controlled outbound traffic:

  * Typically `TLS` from sidecar to egress gateway, mTLS inside mesh.
  * `istio-egressgateway` → external destination (Internet/partner networks, DBs).
* For allowed direct egress (if any) envoys connect directly to external endpoints; NetworkPolicies and firewalls must reflect this.

**Control-plane**

* `istiod` <-> Envoy sidecars/gateways using xDS on:

  * `15010` (plain gRPC – usually disabled)
  * `15012` (secure gRPC)
  * Admin/health:

    * `15014` (control-plane metrics)
* Envoy admin and metrics:

  * Sidecar admin: `15000` (local, not exposed)
  * Envoy Prometheus metrics: `15090` (scraped by Prometheus via sidecar/gateway ServiceMonitor).

**Useful table format (populate in repo)**

`Source → Destination → Port/Protocol → Direction → Encryption (mTLS/TLS) → Authentication → NetworkPolicy rule`

Examples:

* `External client` → `istio-ingressgateway` → `443/TCP` → ingress → TLS → OIDC (via upstream IDP) → `<external LB / FW>`
* `istio-ingressgateway` → `service-a` sidecar → `8080/TCP` → internal → mTLS → SPIFFE ID based on SA → `allow-istio-gateways-to-tenant` NetworkPolicy
* `service-a` sidecar → `istio-egressgateway` → `443/TCP` → internal → mTLS → SPIFFE ID → `allow-tenant-egress-via-gw` NetworkPolicy

---

### 2.4 Failure model & HA strategy

* **Control-plane HA**

  * `istiod` replicated (≥3 replicas) across nodes / failure domains (AZ/DC where available).
  * `PodDisruptionBudget` with `minAvailable: 1` for `istiod` and each gateway.
  * `topologySpreadConstraints` to avoid co-locating all replicas on one node.
  * Leader-election for components that require it (via `coordination.k8s.io/Lease`).

* **Gateway HA**

  * `istio-ingressgateway` and `istio-egressgateway` with at least 2 replicas in PROD.
  * External L4 LB configured with health checks pointing to Envoy health endpoints (`15021`/`/healthz/ready`).
  * If bare-metal, NodePort + external HA pair of L4 proxies (e.g. keepalived + HAProxy) with failover.

* **Sidecars**

  * Injected into all mesh workloads; if injection fails for new Pods, traffic may fail. Admission webhook availability is critical.
  * Webhook has PDB + resources to avoid being evicted or throttled.

* **Network failures**

  * Mesh relies on cluster DNS; partial DNS outage impacts service discovery.
  * Loss of connectivity to PKI/IdP/tracing backends should degrade gracefully (traffic continues; telemetry may be partial).

Document typical failure modes and behavior:

* `istiod` down – existing connections keep working until Envoy config expires; new config changes cannot be applied.
* Gateway Pod loss – LB routes to remaining Pods; if all down, ingress/egress traffic fails.
* Misconfigured `VirtualService`/`DestinationRule` – may cause 404/503; config vetting in CI and Kiali config validation is required.

---

### 2.5 Resource model

Per-component requests/limits (example starting points; tune from load tests):

| Component              | Namespace     | CPU request | CPU limit | Mem request | Mem limit | QoS                    |
| ---------------------- | ------------- | ----------- | --------- | ----------- | --------- | ---------------------- |
| `istiod`               | `istio-d`     | 500m        | 2 vCPU    | 1 Gi        | 4 Gi      | Burstable / Guaranteed |
| `istio-ingressgateway` | `sm-gateways` | 500m        | 2 vCPU    | 512 Mi      | 2 Gi      | Burstable              |
| `istio-egressgateway`  | `sm-gateways` | 250m        | 1 vCPU    | 256 Mi      | 1 Gi      | Burstable              |
| Sidecar per workload   | Various       | 100m        | 500m      | 128 Mi      | 512 Mi    | Burstable              |

Additional notes:

* Consider `CPUManagerPolicy=static` and `TopologyManagerPolicy=best-effort`/`restricted` for latency-sensitive gateways.
* Monitor sidecar overhead relative to application containers to size nodes.
* Avoid oversubscribing nodes that run both control-plane and high traffic gateways.

---

### 2.6 Security model

* **Pod security**

  * All Istio Pods run with:

    * `runAsNonRoot: true`
    * `readOnlyRootFilesystem: true` where possible
    * Minimal Linux capabilities
    * Images from trusted registries only (signed where possible).

* **mTLS & identity**

  * Mesh-wide mTLS enabled by default (`PeerAuthentication` with `STRICT`).
  * Workload identity derived from Kubernetes ServiceAccount → mapped to SPIFFE IDs (`spiffe://<trust-domain>/ns/<namespace>/sa/<serviceaccount>`).

* **AuthN/Z**

  * Ingress:

    * HTTP(S) ingress integrates with OIDC via Envoy filters / RequestAuthentication + AuthorizationPolicy.
    * JWT validation at the gateway (id_token or access_token).
  * In-mesh:

    * Authentication: mTLS with workload identity.
    * Authorization: `AuthorizationPolicy` resources, e.g. allow only `ns A/sa payment-api` to call `ns B/sa billing-api`.
  * Egress:

    * Authorization policies restrict which workloads can use `istio-egressgateway`.
    * DNS / ServiceEntry allow-lists restrict external endpoints.

* **Supply-chain & image security**

  * Istio images pinned to specific versions (no `:latest`).
  * Images signed and verified (e.g. cosign) where supported.
  * Policies (Kyverno/Gatekeeper) to enforce:

    * Allowed registries.
    * No unpinned tags.
    * Mandatory `runAsNonRoot`.

---

### 2.7 Secrets handling

* CA root and intermediate keys:

  * Stored in `istio-d` namespace as Kubernetes Secrets or in external secret store.
  * Rotated via defined procedure; shortish cert lifetimes (e.g. 24h for workload certs).

* OIDC client credentials / TLS keys for gateways:

  * Stored as Kubernetes Secrets or mounted via `ExternalSecrets` from central secret manager.
  * Rotation policy documented (e.g. every 90 days or on incident).

* Encryption at rest:

  * Kubernetes secrets encrypted via KMS-backed `EncryptionConfiguration` (where cluster allows).
  * External secret store has its own at-rest encryption.

---

### 2.8 NetworkPolicy

Baseline policies:

* Each Istio namespace (including `istio-d`, `sm-gateways`) has default-deny ingress/egress.
* Explicit allows:

  * `istio-d` to communicate with kube-apiserver, metrics backend, tracing backend.
  * Gateways to talk to:

    * `istiod` (xDS).
    * Tenant namespaces (ingress) or external networks (egress).
  * Tenant namespaces allowing:

    * `istiod` webhooks.
    * Sidecar → gateway communications (if multi-namespace).
* Document exact policies in a table (`name`, `from`, `to`, `purpose`).

---

### 2.9 Data classification & encryption

* **In-transit**

  * All in-mesh traffic: mTLS by default with strong ciphers; TLS versions aligned with security policy.
  * Ingress: TLS from client to gateway. Optionally TLS termination at Envoy and mTLS internal.
  * Egress: TLS from gateway to external endpoints where supported; certificate pinning for sensitive systems.

* **At-rest**

  * Logs, traces and metrics from Envoy are stored in central observability systems (Loki/ELK, Jaeger/Tempo, Prometheus/Thanos) with at-rest encryption according to platform policy.
  * K8s secrets (certs/keys) encrypted via KMS if available.

---

### 2.10 Threat model (top risks & mitigations)

1. **Misconfiguration leading to traffic outage (e.g., VirtualService or DestinationRule error).**

   * Mitigations:

     * CI validation (istioctl `analyze`, `kubectl apply --dry-run`, schema validation).
     * Policy to require peer review for Istio config in PROD.
     * Kiali used to validate configuration before rollout.
     * Canary/Blue-Green deployments for risky config changes.

2. **Compromise of gateway or sidecar Pods.**

   * Mitigations:

     * Hardened containers (non-root, minimal capabilities).
     * PodSecurity admission enforcing restricted profiles.
     * Image scanning + signing.
     * Fine-grained RBAC; no extraneous K8s API permissions.

3. **Unauthorized access to external systems via egress.**

   * Mitigations:

     * Egress strictly controlled via ServiceEntry + AuthorizationPolicy.
     * Default deny for external egress; only documented endpoints allowed.
     * Logs and alerts on new external hostnames/ports.

(Additional risks can be added: MITM on egress, compromise of CA, etc.)

---

### 2.11 Operational characteristics

* Typical traffic: large volume of short-lived HTTP/gRPC calls between services, plus ingress/egress HTTP(S).
* Latency overhead: expected 2–5 ms per hop under normal load. Must be measured and monitored.
* Startup behavior:

  * Sidecar must be ready before application to avoid initial 503s; use init-containers and `holdApplicationUntilProxyStarts` / `proxyConfig`.
* Dependencies:

  * Strong dependence on DNS; resolution failures show as 5xx in Envoy with upstream errors.

---

### 2.12 SLOs & error budgets

Examples (per PROD cluster):

* **Istiod availability** – 99.9% / month.
* **Ingress HTTP 5xx rate** – < 0.5% of total requests per month (excluding client errors and upstream app failures, where possible).
* **Config propagation delay** – < 30 seconds for 95% of config changes (from `kubectl apply` to Envoy receiving it).

Document how error budget is “spent”:

* New features / experiments (e.g. new filters, advanced routing) must not exceed a defined % of error budget.
* When budget is exhausted, freeze new Istio features and only deploy critical fixes.

---

### 2.13 Lifecycle & upgrades

* **Version strategy**

  * Run a supported Istio LTS version compatible with Kubernetes version.
  * Keep max N-1 version lag compared to latest supported version (define N).

* **Upgrade process**

  1. Test upgrade in DEV cluster using ArgoCD manifest changes (new Helm chart version).
  2. Validate istioctl `x precheck` and `x analyze`.
  3. Upgrade control-plane (`istiod`) using canary revision (new Istio revision, gradually migrate workloads).
  4. Upgrade gateways (ingress, egress, east-west).
  5. Shift traffic gradually to new revision; monitor metrics and logs.
  6. Once stable, remove old revision.

* **Backward compatibility**

  * CRD conversion webhooks used where necessary; maintain a conversion window where both old/new API versions exist.
  * Ensure that application sidecars are upgraded with the control-plane; avoid very old proxies connected to new control-planes.

---

### 2.14 Release cadence & change management

* Regular maintenance window (e.g. once per quarter) for major Istio upgrades.
* Minor/patch releases (security fixes) applied as needed, with shorter change lead times.
* Breaking changes:

  * Communicated via release notes to tenants.
  * Feature flags or per-namespace gradual rollout if needed.
  * Deprecation window documented (e.g., 3–6 months).

---

### 2.15 Rollout strategy

* Control-plane:

  * Use **canary revisions**: deploy new `istiod` with a new revision label, then label namespaces to opt-in.
* Gateways:

  * Rolling update with surge; at least one Pod always available.
* Tenant safely:

  * Start in DEV/UAT; once stable, apply to PROD.
  * For risky features (e.g., new Envoy filters), use Blue/Green gateway deployments: old gateway remains as fallback; external LB capable of routing to old or new.

Rollback plan must be documented with exact manifests / ArgoCD app versions to revert to.

---

### 2.16 Backup & restore requirements

What must be backed up:

* Istio CRD *definitions* and *instances* (export via `kubectl get istio-io/*` as YAML).
* Helm/ArgoCD configuration repositories (the desired state).
* Any external configuration stored outside Git (ideally there is none).
* If you manage etcd (bare-metal control-plane clusters): etcd snapshots following Kubernetes backup practice.

Restore validation:

* Regular DR tests simulate a new cluster being created and Istio restored from Git + backups.
* Validation steps:

  * All key CRDs present and in expected versions.
  * Control-plane and gateways become Ready.
  * Sample traffic (smoke tests) passes through ingress/egress; Kiali shows mesh topology.

---

### 2.17 East-West (multi-cluster) design

*(Adapt when you implement; this describes the target model.)*

* **Use case**
  Connect multiple Kubernetes clusters (on-prem bare-metal and cloud) into one logical mesh, allowing service-to-service traffic across clusters with mTLS and service discovery.

* **East-West gateway**

  * For each cluster, deploy `istio-eastwestgateway` in `sm-gateways`.
  * Expose it via cluster-local LoadBalancer / NodePort + external L4 routing between DCs.
  * Gateways use dedicated mesh network names (Istio `meshNetworks` config) to route across clusters.

* **Trust and identity**

  * Shared root CA across clusters or properly federated trust domains.
  * Each cluster has its own intermediate CA, but all trust each other.

* **Service discovery**

  * Either:

    * **Primary/remote** model (one control-plane for many clusters), or
    * **Multi-primary** model (one control-plane per cluster, each with replicated config).
  * Services exported across clusters via Istio `ServiceExport` / `ServiceEntry` or via built-in multi-cluster features.

* **Routing**

  * Local first: traffic prefers local endpoints; if none healthy, can fail over to remote cluster.
  * Policy to avoid cross-cluster traffic for data-sensitive services if regulations forbid it.

* **Observability**

  * Kiali configured to show multiple clusters; or separate Kiali per cluster with cross-links.
  * Tracing correlated across clusters via consistent trace IDs.

Document precise East-West flows once topology is finalized (which clusters, which DCs, L3 peerings, etc.).

---

## 3. Deploy

### 3.1 Delivery mechanism

* **GitOps with ArgoCD + Helm** (example target)

  * Git repos:

    * `platform/istio-control-plane` – Helm chart / Kustomize for `istiod`, CRDs, base mesh config.
    * `platform/istio-gateways` – Helm chart for ingress/egress/east-west gateways.
    * `platform/mesh-config` – tenant-level VirtualServices, DestinationRules, AuthorizationPolicies.

  * ArgoCD Applications:

    * `istio-base-<env>` – installs CRDs and base config.
    * `istio-controlplane-<env>` – installs/updates `istiod`.
    * `istio-gateways-<env>` – installs/updates ingress/egress/east-west gateways.
    * `mesh-config-<env>` – mesh-wide policies & default routes.

### 3.2 Environments

* `dev` – development cluster

  * Fast iteration, broader debug logging.
  * Tenants may experiment with new Istio features.
* `uat` – user acceptance testing

  * Mirrors prod configuration; less experimentation.
* `prod` – production

  * Strict approval process; config via PR only.
  * Full SLO/alert coverage.

Document differences per env:

* Gateway replica counts.
* Default mesh policies (e.g., allow plain HTTP in dev, strict mTLS in prod).
* Access to Kiali (open to dev teams in dev/uat, restricted in prod).

### 3.3 Step-by-step deployment (beyond “click Sync”)

*New Istio version example:*

1. **Prepare manifests**

   * Update Helm chart version in `istio-control-plane` repo.
   * Update values: global.mtls, meshNetworks, meshConfig, etc.
   * Run `istioctl x precheck` and `istioctl analyze` against target cluster.

2. **Deploy to DEV**

   * ArgoCD `Sync` `istio-base-dev`, then `istio-controlplane-dev`, then `istio-gateways-dev`.
   * Validate Pods Ready.
   * Run smoke tests: sample app accessible through mesh.

3. **Deploy to UAT**

   * Same as DEV, with extra tests (tenant regression tests).

4. **Prod canary**

   * Create new Istio revision (`istioctl install --set revision=<rev>` via GitOps).
   * Label one or two non-critical namespaces with `istio.io/rev=<rev>`.
   * Monitor for errors (Kiali, Grafana, logs).

5. **Prod rollout**

   * Gradually relabel more namespaces to new revision.
   * Once stable, switch default revision and remove old one.

6. **Gateways**

   * Upgrade `istio-gateways-<env>` ArgoCD apps after control-plane stable.
   * Observe LB health checks and traffic metrics.

7. **Post-deployment checks**

   * Validate all SLO dashboards show healthy state.
   * Confirm mTLS, AuthZ policies still enforced.

### 3.4 Post-deploy validation

* Smoke tests / scripts:

  * `curl` from test Pods to key services via mesh.
  * End-to-end traffic via ingress domain.
  * Egress HTTP(S) calls via egress gateway to known external endpoint.

* Readiness criteria:

  * All Istio components Ready.
  * No new Critical alerts fired for Istio.
  * Kiali reports no major configuration errors.

### 3.5 Backout / rollback plan

* For Helm/ArgoCD:

  * Revert to previous Git commit (`git revert` or ArgoCD “Rollback” to previous Revision).
  * Re-sync ArgoCD Application.
* For revision-based upgrades:

  * Relabel namespaces back to old revision.
  * Ensure old `istiod` deployment still present until rollback completed.
* For misbehaving gateway config:

  * Change route weights in `VirtualService` to route 100% traffic back to stable backend.
  * If necessary, scale down new gateway deployment and re-point external LB to old one.

Document exact commands, e.g.:

```bash
kubectl label namespace payments istio.io/rev=old-rev --overwrite
argocd app rollback istio-controlplane-prod <revision-id>
```

---

## 4. Operate

### 4.1 Observability – dashboards & Kiali

* **Kiali**

  * URL: `{{ https://kiali… }}`.
  * Auth: via corporate IdP (OIDC SSO).
  * Typical uses:

    * Mesh graph per namespace: detect missing connections or 4xx/5xx spikes.
    * Config validation: check VirtualService/DestinationRule conflicts.
    * Workload view for latency and error rate.

* **Prometheus & Grafana**

  * Dashboards:

    * `Istio / Control plane` – `istiod` CPU/memory, xDS push errors, config reloads.
    * `Istio / Mesh traffic` – per-service request rate, latency, error rate.
    * `Gateways / Ingress/Egress` – requests by host/path, 4xx/5xx, TLS handshake errors.
  * Documentation should note:

    * Key panels per dashboard.
    * Normal ranges (e.g., p95 latency < 150ms, `istiod` CPU < 60%).
    * Thresholds that, when exceeded, indicate incidents.

* **Tracing (Jaeger/Tempo)**

  * Used for:

    * Debugging cross-service latency.
    * Finding retries/timeouts injected by Envoy.
  * Standard tracing headers: `x-request-id`, `x-b3-*` or W3C `traceparent`.

* **Logs**

  * Envoy access logs collected centrally (e.g. via Fluent Bit).
  * `istiod` logs for config errors and TLS issues.

### 4.2 Alerts

Maintain a table of alert rules; example structure:

* `name` – e.g., `IstiodDown`, `IstioIngressHigh5xx`, `IstioConfigRejected`.
* `expr` – PromQL expression.
* `severity` – `critical` / `warning` / `info`.
* `page or ticket` – On-call (PagerDuty/Opsgenie) vs Jira ticket.

Example alerts:

1. `IstiodDown`

   * `expr`: `kube_deployment_status_replicas_available{deployment="istiod",namespace="istio-d"} < 1`
   * `severity`: `critical`
   * `page or ticket`: `page on-call`

2. `IstioIngressHigh5xx`

   * `expr`: `sum(rate(istio_requests_total{reporter="destination",destination_workload="istio-ingressgateway",response_code=~"5.."}[5m])) / sum(rate(istio_requests_total{reporter="destination",destination_workload="istio-ingressgateway"}[5m])) > 0.02`
   * `severity`: `warning`/`critical` with different thresholds.

3. `IstioConfigRejected`

   * `expr`: `sum(rate(pilot_total_xds_rejects[5m])) > 0`
   * `severity`: `warning`

Each alert must link to a **runbook** with:

* Context of the alert.
* Step-by-step checks.
* Criteria for “resolved”.

### 4.3 Common operations & runbooks

Short procedures for:

* **Scaling gateways**

  * When to scale (`CPU > 70%` or `requests per second > X`).
  * `kubectl scale deploy istio-ingressgateway -n sm-gateways --replicas=N`.
* **Rotating TLS certificates / keys**

  * For ingress hostnames and egress endpoints.
  * Process to update Secrets and reload gateways (usually automatic on volume change).
* **Handling misbehaving webhooks**

  * If Istio sidecar injector webhook is slow/unavailable:

    * Check Pod health in `istio-d`.
    * Temporarily disable `failurePolicy: Fail` → `Ignore` only as last resort, with clear rollback steps.
* **Handling CRD skew**

  * If some clusters run different Istio CRD versions:

    * Ensure all clusters upgrade CRDs to same version.
    * Validate stored versions in etcd.
* **Draining nodes**

  * How to drain nodes with many Istio components using PDBs and `kubectl drain --ignore-daemonsets --delete-emptydir-data`.

### 4.4 Incident triage

Use the standard funnel:

1. **Symptoms**

   * Increased 5xx in ingress/mesh dashboards.
   * Latency spikes.
   * Tenants report failing calls.

2. **Initial checks**

   * Kiali for topology and error-rate hot spots.
   * `kubectl get pods -n istio-d,sm-gateways`.
   * Check alerts for `istiod` or gateway issues.

3. **Actions**

   * Roll back latest config changes (VirtualService, DestinationRule, AuthorizationPolicy).
   * Scale gateways or `istiod` if resource bottleneck.
   * Temporarily route traffic around problematic clusters/endpoints.

4. **Verify**

   * Confirm metrics return to normal.
   * Confirm no new errors in logs.

5. **Comms**

   * Update incident channel/status page.
   * Notify affected tenants.

6. **Postmortem**

   * Document root cause, contributing factors, and mitigation actions (e.g., add validations, new alerts).

### 4.5 Maintenance & capacity

* **Regular maintenance**

  * Certificate rotation calendar.
  * Webhook timeout checks to avoid API blocking.
  * Registry, log store, and trace store compaction/cleanup.

* **Capacity & cost**

  * Monitor:

    * Number of workloads in mesh.
    * Sidecar resource consumption.
    * Gateway throughput and CPU.
  * Plan for:

    * etcd object growth due to Istio CRDs.
    * Prometheus metric cardinality (labels like `destination_workload`).

### 4.6 Security operations

* **CVE handling**

  * Track Istio and Envoy CVEs.
  * For high severity, patch within policy (e.g., 7 days).
* **Policy updates**

  * Regularly refine AuthorizationPolicies, NetworkPolicies, and OPA/Gatekeeper rules based on new findings.
* **Image signing/verification**

  * Enforce signatures for Istio and gateway images.

### 4.7 DR & backups

* What’s backed up:

  * Git repos with Istio config (source of truth).
  * Cluster control-plane (if self-managed): etcd snapshots.
  * Observability data (optional, depending on retention policy).

* Frequency:

  * etcd snapshots – several times per day.
  * Git repos – standard VCS redundancy.
  * DR tests – at least once per year or after significant topology change.

* RTO/RPO:

  * Define target times for mesh recovery and config restoration.
  * Use DR drills to prove targets are met.

---

## 5. Appendices

### 5.1 Port matrix (example)

| Component              | Port  | Protocol | Direction         | Purpose                           |
| ---------------------- | ----- | -------- | ----------------- | --------------------------------- |
| `istiod`               | 15010 | gRPC     | inbound from mesh | xDS (unsecured, usually disabled) |
| `istiod`               | 15012 | gRPC/TLS | inbound from mesh | xDS (secured)                     |
| `istiod`               | 15014 | HTTP     | internal          | control-plane metrics / debug     |
| `istio-ingressgateway` | 80    | HTTP     | external ingress  | HTTP traffic                      |
| `istio-ingressgateway` | 443   | HTTPS    | external ingress  | HTTPS traffic                     |
| `istio-ingressgateway` | 15021 | HTTP     | health checks     | Envoy health                      |
| `istio-ingressgateway` | 15090 | HTTP     | Prometheus scrape | Envoy metrics                     |
| `istio-egressgateway`  | 443   | HTTPS    | outbound          | external TLS                      |
| Sidecars               | 15000 | HTTP     | local only        | Envoy admin                       |
| Sidecars               | 15090 | HTTP     | Prometheus scrape | sidecar metrics                   |

Extend this with all ports actually used in your deployment.

### 5.2 Config parameters table (key tunables)

| Parameter                                | Scope         | Default (example) | Valid range / notes                        |
| ---------------------------------------- | ------------- | ----------------- | ------------------------------------------ |
| `global.mtls.enabled`                    | mesh-wide     | `true`            | `true`/`false`; must be `true` in PROD     |
| `meshConfig.accessLogFile`               | mesh-wide     | `/dev/stdout`     | File vs stdout; adjust for logging backend |
| `meshConfig.defaultConfig.tracing`       | mesh-wide     | enabled           | Sampling rate (e.g. 1%)                    |
| `pilot.traceSampling`                    | control-plane | 1.0               | 0–100; percent of traces sampled           |
| `global.proxy.resources.requests.cpu`    | sidecar       | `100m`            | tune per workload                          |
| `global.proxy.resources.requests.memory` | sidecar       | `128Mi`           | tune per workload                          |
| `global.outboundTrafficPolicy.mode`      | mesh-wide     | `REGISTRY_ONLY`   | `REGISTRY_ONLY` or `ALLOW_ANY`             |
| `meshConfig.enableAutoMtls`              | mesh-wide     | `true`            | auto upgrade to mTLS when possible         |

Add other parameters as needed (timeouts, connection pool sizes, etc.).

### 5.3 Known issues & limitations (examples)

* High cardinality of metrics when many services (`destination_workload`, `source_workload`) – requires metrics relabeling.
* Sidecar injection may increase Pod startup time; sensitive apps must be tuned with proper probes and startup delays.
* Some legacy protocols (non-HTTP/TCP, custom UDP) are harder to proxy; may require exclusion from mesh.

### 5.4 FAQ (examples)

* **Q:** How do I onboard my namespace to the mesh?
  **A:** Label the namespace with `istio-injection=enabled` (or `istio.io/rev=<rev>` for revisioned upgrades) and redeploy workloads.

* **Q:** How do I expose my service externally?
  **A:** Create a `Gateway` and `VirtualService` resource bound to `istio-ingressgateway` with your host/path; follow ingress onboarding guide.

* **Q:** How do I call an external HTTP API?
  **A:** Request a new ServiceEntry + AuthorizationPolicy for the destination host, and route traffic via `istio-egressgateway`.

* **Q:** How can I see who calls my service?
  **A:** Use Kiali traffic graph or Grafana dashboards; filter by `destination_workload=<your-service>`.

### 5.5 Glossary

* **Mesh** – Logical network of services managed by Istio.
* **Control-plane** – `istiod` and CRDs managing mesh configuration.
* **Data-plane** – Envoy sidecars and gateways handling actual traffic.
* **Gateway** – Envoy proxy that handles edge (ingress/egress/east-west) traffic.
* **mTLS** – Mutual TLS; both client and server authenticate each other.
* **ServiceEntry** – Istio resource describing an external service.
* **VirtualService** – Istio resource defining routing rules.
* **DestinationRule** – Istio resource configuring subsets and traffic policies for a service.

### 5.6 ADR links

* `ADR-001 – Choice of Istio as service mesh`
* `ADR-002 – Mesh-wide mTLS policy`
* `ADR-003 – Egress strategy via dedicated gateway`
* `ADR-004 – East-West multi-cluster topology`

(Replace with actual ADRs in your repo.)

---
