# OpenShift Management Cluster Repository Governance

**Document status:** Platform engineering governance instructions
**Applies to:** `aib_ocp_cluster_configuration-prod` and any related management-cluster GitOps repository adopting these rules
**Audience:** Platform engineers, reviewers, automation developers, and AI coding assistants (including GitHub Copilot)
**Default integration branch (customer Bitbucket/Stash):** `master`
**Policy authority:** The separate governance repository, `aib_ocp_cluster_configuration_governance`, especially `policies/repository-structure/approved-components.yaml`

> **AI AGENT — MANDATORY:** Treat this document as instructions for proposing changes, not as permission to bypass access controls. Before generating or editing YAML, inspect the current repository, the authoritative governance policy, and the target environment. If a requested change conflicts with these rules, stop, identify the conflict, and propose an authorised change request. Do not silently create new directories, move resources, alter protected content, or change the governance controls.

## 1. Purpose and operating principles

This repository is the declarative source for OpenShift management-cluster configuration. Changes must be **predictable, reviewable, reproducible, environment-specific, secure, and owned by exactly one reconciliation mechanism**.

1. **Git is the reviewed source of desired state.** No direct, undocumented changes to managed cluster resources. Emergency changes require an incident record and a follow-up reconciliation PR.
2. **Separate bootstrap from ongoing reconciliation.** Ansible Automation Platform (AAP) handles Day 0 installation and the GitOps handover. Argo CD / OpenShift GitOps owns explicitly delegated Day 1 and Day 2 configurations. Do not have AAP and Argo CD continuously write to the same resources.
3. **One active controller per resource.** Argo CD applications, ACM policies and other controllers must not independently reconcile the same object. Define ownership explicitly and respect the management/hosted-cluster boundary.
4. **Deny structural changes by default.** Directory structure, protected content and governance policy changes require an explicit, separately reviewed process.
5. **Prefer small, auditable PRs.** Do not combine unrelated operator, security, networking, storage and environment changes.
6. **Do not assume repository approval equals deployment approval.** A successful governance check validates the configured rules; it does not prove the YAML is operationally safe or supported by OpenShift.

## 2. Repository boundaries and structure

**Repository roles**

| Repository | Responsibility |
| --- | --- |
| Management-cluster configuration | Cluster/bootstrap GitOps definitions and environment configuration |
| Separate governance repository | Approved path policy, Python validator and regression tests; independently controlled revisions |

Maintain the existing management repository layout. Established top-level areas include:

```text
ansible/
  inventories/
  playbooks/
  roles/
gitops/
  applicationsets/
  bootstrap/
  clusters/
  components/
  governance/
  hosted-clusters/
README.md
```

**Do not infer that an example layout authorises creating a new directory.** Read the repository tree and policy before choosing a destination. Reuse the established `site-a` / `site-b` (or customer-specific site names), environment and hosted-cluster paths that actually exist. Do not introduce new naming schemes, aliases or overlays unless reviewed.

### 2.1 Reusable components versus instance configuration

- **`gitops/components/`**: shared and reusable platform definitions. Treat it as **recursively protected** under the expanded governance policy.
- **`gitops/clusters/`**: cluster-specific desired state, only where the existing tree and application architecture place it.
- **`gitops/hosted-clusters/`**: hosted control plane definitions and approved site/environment instances; distinguish production, nonproduction and virtualisation infrastructure clusters.
- **`gitops/bootstrap/`**: root/bootstrap application entry points and handover configuration; do not invent additional roots or application names.
- **`gitops/applicationsets/`**: approved ApplicationSets for selected deployment patterns; do not introduce automatic discovery of arbitrary repository paths.
- **`ansible/`**: Day 0 provisioning and handover; do not duplicate Argo CD's steady-state ownership.
- **`gitops/governance/`**: configuration that is explicitly owned by this repository; **not** a replacement for the independent governance repository.

### 2.2 Structure policy and change policy are different

The authoritative governance YAML uses **two independent controls**:

```yaml
protected_paths:
  - gitops/components

approved_directories:
  gitops/components:
    - common
    - management
    - operators
    - security
    - storage
    - virtualization
    - workload
  gitops/components/operators:
    - gitops
    - acm
    - acs
    - logging
    - virtualization
    - storage
  gitops/components/operators/storage:
    - hitachi-csi
  gitops/components/storage:
    - hitachi
    - kubevirt-csi
```

This block describes the **expanded proof-of-concept policy**. Confirm the actual policy in the customer's governance repository before treating the values as deployed. The following semantics are mandatory:

- `protected_paths`: recursively detects additions, modifications and deletions of files anywhere below those paths. **An approved component directory is still protected against changes.**
- `approved_directories`: specifies the allowed **immediate** subdirectories at each listed location and detects missing or extra directories there. It is not, by itself, a general waiver to create or edit files.
- Do not change this document, a YAML manifest, or a PR description to claim a policy exception. The current validator does not implement a per-PR exception mechanism.
- Never alter `.gitkeep` files solely to make an otherwise prohibited structural change pass.

**Agent behaviour:** If asked to edit, create, delete, rename or move anything under a protected path, **do not perform the edit automatically**. Identify the path, explain that the change is blocked by policy, and request an authorised governance change workflow. If a change is approved, an owner must specify how the protected change will be implemented and validated; the present CI script would still fail on that changed path until a controlled mechanism is introduced.

## 3. Where to put new YAML

Before generating a file, determine and record:

1. **Target:** management cluster or a named hosted cluster; site; production/nonproduction/virtualisation role.
2. **Lifecycle:** Day 0 bootstrap, Day 1 platform configuration, or Day 2 operation.
3. **Ownership:** the single controller responsible for reconciliation (AAP, Argo CD, ACM or another explicitly designated operator).
4. **Scope:** reusable component versus cluster-specific or instance-specific values.
5. **Approved destination:** an existing, permitted directory and existing application/Kustomize inclusion mechanism.
6. **Dependencies:** namespace, CRDs, operator availability, credentials, storage class, ingress, certs, network dependencies and rollout sequence.

**Never** create folders because a prompt asks for a new technology, environment or operator. Propose the necessary directory/policy change if the approved structure does not contain a suitable location. **Never** make a shared base environment-specific just to simplify a generated YAML file.

## 4. YAML authoring requirements

All proposed Kubernetes/OpenShift YAML must:

- Use valid YAML with consistent **2-space indentation**, no tabs, UTF-8 encoding and a trailing newline.
- Declare the correct `apiVersion`, `kind`, `metadata.name`, and `metadata.namespace` for namespaced resources. Verify the API is supported by the *target cluster version* and installed CRDs; do not invent fields.
- Use DNS-compliant, stable names and the repository's established labels/annotations; add standard `app.kubernetes.io/*` metadata where appropriate and consistent with existing conventions. Do not introduce arbitrary labels as implicit selectors.
- Keep environment-specific settings in the approved instance/overlay location. Avoid duplicate resource definitions and hardcoded site-specific values in shared components.
- Use the repository's established Kustomize/Helm/Application patterns. Confirm `kustomization.yaml`, Helm values or Application source references include the new manifest; a YAML file that is never referenced is not deployed.
- Declare resource requests/limits where appropriate; use readiness/liveness/startup probes for applicable workloads; specify supported security contexts and least-privilege service accounts.
- Avoid hardcoded node IPs, VIPs, DNS zones, hostnames, image tags, operator channels, storage-class names, certificates or ingress classes unless supplied from approved design artefacts and verified for the target environment.
- Use immutable image digests where the organisation's image policy requires them; only use approved mirrored/internal registries in disconnected clusters.
- Avoid broad wildcard permissions, `cluster-admin`, privileged containers, `hostNetwork`, `hostPath` and unsafe SCC assignments unless explicitly justified and approved.
- Do not add Argo CD sync waves, automated pruning, self-heal or deletion behaviour without considering dependencies and blast radius. Protect foundational resources from accidental deletion.
- Make idempotent changes that minimise drift and do not depend on unspecified manual console steps.

### 4.1 Secrets and credentials

**Never commit plaintext passwords, tokens, private keys, kubeconfigs, client secrets, access tokens, certificates with private material, or production credentials.** Use the customer's approved secret-management mechanism, even if the eventual target object is a native Kubernetes `Secret`. Do not assume that base64 is encryption. Do not write fabricated secret values that resemble real credentials; use clearly identified references or placeholders in non-deployable examples. Do not print sensitive data in CI logs or generated explanations.

### 4.2 Management and hosted cluster separation

- Explicitly distinguish management cluster resources from hosted control plane resources and hosted worker-cluster configuration.
- Do not assume one site's resources, certificates, storage configuration or identity integration can be copied unchanged to another site.
- Do not deploy ACM policies and Argo CD applications to reconcile the same target object independently.
- If using ACM `PlacementDecision`, confirm whether reconciliation is centralised or performed by per-cluster Argo CD instances. Do not assume ACM placement is the correct deployment mechanism for every application.

## 5. Git workflow in Bitbucket/Stash

**Default branch:** `master` for the current customer repository, subject to the repository's configured default branch and agreed standard. **Do not assume `main`** from the GitHub proof of concept.

1. Synchronise the target repository and branch.
2. Create a descriptive feature branch (`feature/<change-id>-<short-description>` or the customer's established convention).
3. Make only the authorised, scoped change; inspect the diff and avoid committing unrelated files or ZIP copies.
4. Run applicable local checks and record their output.
5. Commit with a meaningful description and push the feature branch.
6. Open a Bitbucket Pull Request targeting `master`; explain purpose, target site/cluster, resource ownership, impact, rollback and evidence.
7. Obtain required approvals and successful CI/build status before merging. **Never bypass a failing or missing required governance check.**
8. After merge, verify the target application or deployment state and record any incident/rollback.

**No direct commits to protected branches; no force pushes; no rewriting shared history; no silent conflict resolution.** Do not use privileged credentials or change Bitbucket permissions/merge checks to bypass a failure.

## 6. Governance repository change control

The separate governance repository contains:

```text
policies/repository-structure/approved-components.yaml
scripts/validate-structure.py
tests/test_validate_structure.py
README.md
```

Changes to the governance policy, validator or tests require a separate reviewed PR, with particular scrutiny because the author can otherwise weaken enforcement. Independent code-owner approval is recommended and must be configured by the customer; it is **not supplied by this Markdown file**.

**Process:** propose change and rationale → update policy/script/tests together → run regression suite → PR/independent approval → merge to protected governance branch → select/pin an approved governance commit in the platform CI configuration → test a compliant and a deliberately non-compliant platform PR.

The tested Python regression suite is:

```powershell
python -m unittest discover -s tests -p "test_*.py" -v
```

The expanded proof of concept includes ten tests covering approved/missing/unauthorised directories, committed modifications/additions/deletions, untracked files, recursive security/common protection and changes outside the protected tree.

**Do not change the management repository's `protected_paths` indirectly by editing a copy of the policy in that repository.** The authoritative policy resides in the governance repository; CI must fetch a trusted, explicitly approved revision.

## 7. CI validation contract for Stash/Bitbucket

The proof of concept used GitHub Actions. **GitHub Actions files do not run in Stash/Bitbucket.** The customer must connect an approved CI platform (for example Jenkins, if available), publish build status to Bitbucket and configure a required successful-build merge check.

At minimum, the platform PR pipeline must:

1. Trigger for PRs targeting the protected default branch (`master`).
2. Check out the **PR head** and fetch the correct **target base commit**.
3. Retrieve the Python validator and YAML policy from a **trusted, pinned governance revision**, not from unreviewed PR content.
4. Install a supported Python runtime and PyYAML.
5. Run the validator with `--repo` and the exact PR target SHA via `--base-ref`; avoid assuming `origin/main` or a stale local branch.
6. Publish PASS/FAIL to Bitbucket and require that status as a merge condition.
7. Fail closed on errors fetching the policy, resolving the baseline or executing the validator; errors must not be treated as PASS.

Illustrative CLI (substitute real checkout paths and base SHA):

```text
python governance/scripts/validate-structure.py --repo platform --base-ref <PR_BASE_COMMIT_SHA>
```

Validator exit codes: `0` = PASS; `1` = governance violation; `2` = execution error. Only `0` permits the required build status to pass.

**Until the Jenkins/Bamboo/other CI integration and Bitbucket merge checks are live, this file acts as a human/AI authoring rulebook, not automatic enforcement.**

## 8. Required verification evidence

For each proposed YAML change, provide:

- Proposed file path and why it is authorised by policy.
- Site, environment, cluster and ownership/controller.
- Scope, dependencies, supported OpenShift/Kubernetes API and expected reconciliation behaviour.
- Relevant `git diff` summary and whether a protected path is affected.
- YAML/Kustomize/Helm/schema checks appropriate to the change and evidence of results.
- Security, RBAC, network, certificate and secret implications.
- Deployment/rollback and verification method; identify anything not tested.

**Do not claim that tests passed unless they were actually executed.** A good-looking manifest is not proof of correctness.

## 9. Required agent response format

Before making a change, output a concise **change assessment**:

```text
Target repository:
Target branch:
Target site / cluster / environment:
Requested resource and purpose:
Proposed file path:
Protected path affected: Yes / No / Unknown
Approved directory rule satisfied: Yes / No / Unknown
Reconciliation owner:
Dependencies and risk:
Validation to run:
```

If a protected path is affected, or the directory rule/owner/target version is unknown, **stop before writing** and state what requires approval or verification. Otherwise, generate a minimal diff (not a repository-wide rewrite), explain why it fits the established pattern, and provide the checks for the developer to execute. Never state that a PR was approved, merged or deployed unless independently confirmed.

## 10. Known limitations and follow-up controls

The governance validator protects **repository structure and selected Git changes**, not Kubernetes/OpenShift correctness. It does **not** itself perform full manifest schema validation, RBAC risk analysis, secret scanning, supported-version checks, admission-policy compliance, Argo CD dry runs or cluster deployment validation. These require additional CI/security checks.

The proof-of-concept local untracked-file detection honours `.gitignore`; ignored, uncommitted files are not reported. The current validator has no controlled exception mechanism for an otherwise protected change. Directory checks inspect the checked-out filesystem, so Git-tree edge cases such as symlinks and deleted empty directories need further hardening for production use.

**Ownership and enforcement must be explicitly handed over:** the platform team owns repository content and change review; the governance owner approves policy changes; the CI/Bitbucket owner ensures the pipeline and merge checks enforce the trusted governance release.
