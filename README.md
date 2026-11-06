# Cluster Platform Repository — Engineer's Guide

## Purpose

This repository is the approved source of truth for provisioning the two OCP management clusters and configuring their managed clusters. **Do not change the approved directory hierarchy or introduce a new platform technology without architectural approval.**

- **Alpha:** one management cluster (Argo CD, ACM/MCE, ACS) and three initial hosted clusters (virtualization, prod, nonprod).
- **Omega:** same arrangement as Alpha.
- **One Argo CD instance per management cluster** manages that site's management cluster and three initial hosted clusters. Both use this repository, with separate site targets.
- **AAP** provisions and bootstraps management clusters; **Argo CD** owns post-handover declarative configuration. **ACM** owns designated fleet compliance resources. Avoid multiple controllers owning the same Kubernetes resource.

## Where does a change belong?

| Change | Put reusable definition here | Select or instantiate here |
|---|---|---|
| Install/configure an operator | `gitops/components/operators/<approved-operator>/` | `gitops/clusters/<site>/<cluster>/operators/` |
| Configure storage | `gitops/components/storage/<approved-backend>/` | `gitops/clusters/<site>/<cluster>/storage/` |
| Change certificates or security settings | `gitops/components/security/` | `gitops/clusters/<site>/<cluster>/security/` |
| Configure common authentication/RBAC/monitoring | `gitops/components/common/` | `gitops/clusters/<site>/<cluster>/configuration/` |
| Add hosted control plane / worker pool | `gitops/hosted-clusters/base/` | `gitops/hosted-clusters/instances/<site>/<role>/` |
| Apply fleet compliance | `gitops/governance/acm/policies/` | `gitops/governance/acm/placements/` and `bindings/` |
| Manage ACS policy definitions | `gitops/governance/acs/` | ACS Central automation, not ordinary Kubernetes apply |
| Install/bootstrap management cluster | `ansible/` | AAP job templates |

The `components/` directory defines **what is available**. The `clusters/` overlays define **which cluster receives it**. The `hosted-clusters/` manifests create hosted control planes and NodePools on the management cluster. `governance/` contains centrally administered policy definitions.

## Approved structure (overview)

```text
cluster-platform/
├── README.md
├── ansible/
│   ├── inventories/{alpha,omega}/
│   ├── playbooks/
│   └── roles/
└── gitops/
    ├── bootstrap/
    ├── applicationsets/
    ├── components/
    │   ├── operators/{gitops,acm,acs,logging,virtualization,storage/hitachi-csi}/
    │   ├── common/{authentication,rbac,monitoring,networking}/
    │   ├── security/{scc,network-policies,pod-security,certificates,secrets,audit,compliance}/
    │   ├── storage/{hitachi,kubevirt-csi}/
    │   ├── management/
    │   ├── virtualization/
    │   └── workload/
    ├── hosted-clusters/
    │   ├── base/
    │   └── instances/{alpha,omega}/{virt,prod,nonprod}/
    ├── clusters/
    │   ├── alpha/{management,virtualization,prod,nonprod}/
    │   └── omega/{management,virtualization,prod,nonprod}/
    └── governance/
        ├── acm/{policies,placements,bindings}/
        └── acs/{policies,configuration}/
```

Braces abbreviate multiple actual directories; they are **not literal directory names**. Each cluster overlay has `kustomization.yaml`, `operators/`, `storage/`, `security/` (where used), and `configuration/` as applicable. Do not create new component families by copying these examples without review / approval.

## Kustomize

Kustomize **renders YAML**, it does not deploy it. A `kustomization.yaml` lists resources to include and patches to apply. Argo CD renders that path and deploys it to the cluster registered as its destination.

**Eg: include an existing, approved Logging Operator in Alpha production.**

`gitops/components/operators/logging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - operatorgroup.yaml
  - subscription.yaml
```

`gitops/clusters/alpha/prod/operators/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../components/operators/logging
```

The referenced component must contain valid, reviewed operator manifests. Keep other existing resources in the overlay. The cluster root `gitops/clusters/alpha/prod/kustomization.yaml` must include `operators/` (or an Argo CD Application must target that subpath); **never let two Applications own the same resources**.

Preview the result without applying it:

```bash
oc kustomize gitops/clusters/alpha/prod/operators
oc kustomize gitops/clusters/alpha/prod
```

**Example: production-only certificate reference.** The certificate's TLS Secret must already be provisioned securely in `openshift-ingress`; never commit private keys in plaintext.

`gitops/clusters/alpha/prod/security/certificates/ingress-patch.yaml`:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: alpha-prod-tls
```

In the relevant certificate overlay, include the approved base defining that same IngressController and add:

```yaml
patches:
  - path: ingress-patch.yaml
```

This patch only affects Alpha production when that overlay is rendered and deployed. The `defaultCertificate` Secret is in `openshift-ingress`, not the operator namespace.

## Workflow 1 — Normal configuration change

1. Create a feature branch from the protected branch.
2. Find the approved reusable component and the target cluster overlay.
3. Modify the component or add an overlay reference/patch; do not duplicate whole manifests.
4. Run `oc kustomize` for the affected overlay and verify other clusters are unchanged.
5. Open a pull request describing target clusters, expected changes, and rollback.
6. CI checks structure, manifests, rendered cluster impact and secrets; designated owners approve.
7. Merge only after required checks pass; the site's Argo CD reconciles the approved change.
8. Verify sync, health and the resulting resources; record evidence.

**Flow:** Branch → Edit → Render → PR → CI/approval → Merge → Argo CD → Verify.

## Workflow 2 — Add an approved operator to one cluster

1. Confirm the operator already exists under `gitops/components/operators/` and is supported on the target cluster.
2. Add its relative path to `gitops/clusters/alpha/prod/operators/kustomization.yaml` (example target).
3. Render that overlay; verify only Alpha production receives the new operator resources.
4. Raise PR; obtain platform approval; merge; verify Operator Subscription/CSV and workload readiness.

**To deploy to multiple clusters:** add the approved component to each eligible cluster overlay or an already-approved shared profile. A new directory under `components/operators/` is an **architecture change**, not a routine operator installation.

## Workflow 3 — Apply a compliance rule across the fleet

1. Create/reuse an ACM `Policy` under `gitops/governance/acm/policies/` on the appropriate management hub.
2. Define a `Placement` selecting the intended managed clusters using agreed labels; include management clusters only if they are registered/targeted as required.
3. Create a `PlacementBinding` connecting the policy to the Placement.
4. Verify the selected cluster list before enforcement; start with `inform` where appropriate, then seek security approval before `enforce`.
5. Argo CD applies the policy objects to the hub; ACM evaluates or enforces against the selected managed clusters.

**Flow:** Policy → Placement → PlacementBinding → Argo CD on hub → ACM evaluation/enforcement.

To avoid reconciliation conflicts - do not allow ACM and Argo CD to reconcile the same target resource independently.

## Workflow 4 — Add a hosted cluster

1. Confirm platform capacity, DNS, ingress, network, storage and supported HCP/NodePool design.
2. Create the site-specific HostedCluster and NodePool manifests under `gitops/hosted-clusters/instances/<site>/<role>/`, reusing approved bases.
3. Register that instance in the management cluster's GitOps composition; review cluster names and target site.
4. Merge after technical approval; Argo CD creates the HostedCluster/NodePool on the management cluster.
5. Wait for hosted control plane and worker readiness; register the new cluster with ACM and the site's Argo CD using the approved access mechanism.
6. Create/select its `gitops/clusters/<site>/<cluster>/` overlay and enable its platform configuration only after registration and readiness.

**Flow:** Prerequisites → HCP/NodePool PR → Management Argo CD → Ready → ACM/Argo CD registration → Cluster overlay → Validate.

## Workflow 5 — Request a new technology (e.g., ODF)

1. **Do not** create `gitops/components/storage/odf/` or `gitops/components/operators/storage/odf/` directly.
2. Raise an architecture change request explaining purpose, supportability, storage topology, impact and alternatives to the approved Hitachi design.
3. Platform/Storage Architecture reviews and approves or rejects the change.
4. If approved, governance owners update the trusted repository allowlist and approved structure baseline **before** the component PR is permitted.
5. Introduce the new component via a separate reviewed PR, validate manifests and intended cluster scope, then deploy in controlled stages.

## Repository guardrails — what is enforced

| Rule | Expected enforcement |
|---|---|
| Approved directory/component allowlist | Trusted CI validation rejects unapproved paths |
| No arbitrary operator/storage technologies | Resource-content checks plus approved component list |
| No missing required `kustomization.yaml` | Structural and Kustomize render checks |
| No unexpected cluster impact | Rendered-manifest diff for all eight cluster overlays |
| No plaintext credentials/private keys | Secret scanning |
| No direct pushes to protected branch | Bitbucket branch restrictions |
| No merge with failed/missing validation | Required CI build-status merge check |
| Required platform/security/storage approval | Bitbucket reviewer/merge rules, as supported by the installed edition and integrations |
| No contributor edits to enforcement policy | Trusted policy and CI logic controlled separately from contributor PRs |

**Critical:** A README, allowlist file or script in this repository is not the enforcement mechanism. The repository administrator must require a CI check that executes **trusted** validation rules. Ordinary contributors must not be able to edit those rules in the same PR and bypass them.

## Before submitting a PR

- [ ] The change is in an approved directory and uses approved components.
- [ ] The target cluster(s) and site are explicitly identified.
- [ ] The affected Kustomize overlays render successfully.
- [ ] The rendered change has been checked for unintended cluster impact.
- [ ] No plaintext secrets or private keys are committed.
- [ ] Required platform, security and storage approvals are identified.
- [ ] Deployment validation and rollback are documented.

## Ownership and operational boundaries

- **Platform Engineering:** shared components, overlays, GitOps deployment and operational validation.
- **Platform Architecture / Technical Authority:** approved hierarchy, new technologies and architecture exceptions.
- **Storage Team:** storage designs, CSI configuration and storage change review.
- **Security Team:** security controls and compliance policy approval.
- **CI Administrators:** branch restrictions, merge checks and trusted validation pipeline.

The repository's protected branch is the deployment source of truth; changes to it are reconciled according to each Argo CD Application's sync policy. The exact CI product and Bitbucket Data Center/Stash version must be confirmed before implementing merge checks or path-specific approval controls.
