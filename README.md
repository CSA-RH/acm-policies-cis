# ACM Policies Introduction

This repository shows how Red Hat Advanced Cluster Management (ACM) governance can detect and enforce OpenShift platform configuration standards across a managed fleet - driving consistency, regulatory alignment, and risk reduction at scale.

It includes multiple examples using native ACM ConfigurationPolicy in both `inform` and `enforce` modes, plus a ValidatingAdmissionPolicy example that blocks non-compliant ClusterRoleBinding creation at admission time.

Policies are built with the PolicyGenerator kustomize plugin and can be deployed standalone or via ArgoCD for GitOps-driven delivery.

> **Disclaimer:** This repository is provided as a learning resource and starting point, not a production-ready blueprint. The examples are intentionally simplified to illustrate core concepts and may not cover every requirement of your environment and adapt these guides to fit your specific needs.

## GitHub Repository Organization

The Git repository used for this demo has the following structure:

- **argocd:** ArgoCD ApplicationSet definitions.
- **hub:** Hub cluster configuration only.
- **operators:** One subfolder per operator to install on managed clusters and/or hub cluster. Each follows the same structure as `policies/` (kustomization + PolicyGenerator + placement + manifests). Operators are Deployed via the `operators-appset.yaml` ApplicationSet.
- **policies:** One subfolder per audit/compliance policy domain. 
- **auxiliar:** contains manifests, to be used in the tests chapter and contains the policies, that will be used in the exercices of the `testing` chapter.

# Created Policies

Once the tests from the testing chapter are completed the following are the list of  policies created.

## Governance Metadata

All policies carry three ACM metadata fields used for filtering and grouping in the Governance dashboard:


| Domain     | Policy                               | Target                   | Source                           | What it does                                                                                                               |
| ---------- | ------------------------------------ | ------------------------ | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Registries | `allowed-registries`                 | `registry-enforce=true`  | `auxiliar/registries/`           | Patches `image.config.openshift.io/cluster` to add approved registry configuration                                         |
| Kubeadmin  | `kubeadmin-remove-enforce`           | `kubeadmin-enforce=true` | `policies/kubeadmin/`            | Deletes the `kubeadmin` secret on opted-in clusters                                                                        |
| GitOps     | `gitops-operator`                    | `local-cluster` (hub)    | `hub/gitops/`                    | Installs OpenShift GitOps and configures ArgoCD with PolicyGenerator plugin                                                |
| RBAC       | `cm-rbac-exceptions-exists`          | All OpenShift clusters   | `policies/rbac/`                 | Deploys the `rbac-policy-exceptions` ConfigMap to `acm-policies` on all managed clusters                                   |
| RBAC       | `cis-cluster-admin`                  | All OpenShift clusters   | `policies/rbac/`                 | Flags ClusterRoleBindings to `cluster-admin` where any subject is a non-platform ServiceAccount, User, or Group            |
| RBAC       | `sa-token-restriction`               | All OpenShift clusters   | `policies/rbac/`                 | Flags default ServiceAccounts in user namespaces that lack `automountServiceAccountToken: false`                           |
| RBAC       | `detect-anonymous-and-wildcard-rbac` | All OpenShift clusters   | `policies/rbac/`                 | Flags CRBs granting access to unauthenticated/anonymous users and wildcard Roles in user namespaces      |
| Compliance | `compliance-operator`                | All OpenShift clusters   | `operators/compliance-operator/` | Installs the Compliance Operator (namespace, OperatorPolicy, OperatorGroup)                                                |
| Compliance | `compliance-cis-scan`                | All OpenShift clusters   | `operators/compliance-operator/` | Deploys CIS ScanSetting + ScanSettingBinding                                                                               |
| Compliance | `compliance-cis-results`             | All OpenShift clusters   | `operators/compliance-operator/` | Reports failed CIS ComplianceCheckResults - non-compliant when checks fail                                                 |
| VAP        | `vap-cluster-admin-allow-list`           | All OpenShift clusters   | `policies/vap/`                  | Deploys a ValidatingAdmissionPolicy that denies ClusterRoleBindings to `cluster-admin` with subjects not on the allow-list |


---

# Deployment Guide

## Overview

The deployment has three phases:

1. **Hub cluster:**
    - Apply ACM namespace, RBAC, ManagedClusterSets, and ManagedClusterSetBindings to the hub cluster.
    - Deploy Policy to install **GitOps operator** and configure ArgoCD with the PolicyGenerator plugin.
2. **Policy and operator deployment:** Deploy ACM policies and install Compliance Operator to all clusters using a GitOps aproach.
3. In the exercises chapter, create new policies.

## Prerequisites

Before starting, ensure the following:

- **Red Hat ACM** is installed on the hub cluster and the `MultiCluster Engine` is installed and running.
- **At least one managed cluster is deployed** with the name `cluster1`. Verify with: `oc get managedclusters`.
- **oc CLI** is available and authenticated to the hub cluster with `cluster-admin` privileges.
- **Git repository** is accessible from the hub cluster (the ArgoCD repo-server must be able to clone it).

Do the following configurations:

- Clone the repository
You should clone the repository for a git account that you own, so that you are allowed to push to the repository changes.
  ```bash
  git clone https://github.com/CSA-RH/acm-policies-cis.git
  cd acm-policies-cis
  ```
- Login to the ACM HUB
  ```bash
  oc login -u admin -p <password>  https://<api>:6443
  ```
- Apply manifests
Create the `acm-policies` namespace, RBAC for the ArgoCD controller, ManagedClusterSets, and ManagedClusterSetBindings. The `acm-policies` namespace must exist before the OperatorPolicy can be applied in the next step.
  ```bash
  oc apply -f hub/clustergroups/
  ```
- Install Policy that will enforce the creation of the GitOps Operator to the ACM HUB
Apply the GitOps policy to the hub. ACM will install the OpenShift GitOps operator, and configure the ArgoCD instance with the PolicyGenerator plugin.
**Note: You must configure the policy generator plugin in your laptop before running this command.**
  ```bash
  kustomize build hub/gitops --enable-alpha-plugins | oc apply -f -
  ```
  Monitor the policy compliance:
  ```bash
  oc get policy gitops-operator -n acm-policies -w
  ```
  > **Note:** The ConfigurationPolicy for the ArgoCD patch will initially be non-compliant while the operator is installing (the ArgoCD instance does not exist yet). ACM retries automatically - once the operator creates the ArgoCD instance, the patch is applied. This is expected behavior.
- Deploy ACM Policies and Operators
Apply both ApplicationSets. Each auto-discovers folders under its respective directory and creates an ArgoCD Application per domain.
  ```bash
  oc apply -f argocd/operators-appset.yaml
  oc apply -f argocd/policies-appset.yaml
  ```
- Verify the generated ApplicationSets and Applications:
  ```bash
  oc get applicationset.argoproj.io -n openshift-gitops
  oc get app.argoproj.io -n openshift-gitops
  ```
- List Policies created
  ```bash
  # List all policies in the acm-policies namespace
  oc get policies -n acm-policies
  ```

---

# Testing

## Pre-Preparation

- Login to the ACM HUB
  ```bash
  oc login -u admin -p <password> <api_fqdn:6443>
  ```
- Extract secret of one managed cluster
  ```bash
  CLUSTER_NAME=cluster1
  SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
  oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig
  ```
- Review the Compliance Operator scan results and the controls that require manual verification.
  -Navigate to **ACM → Governance** and filter by the `CIS OpenShift Container Platform 4 Benchmark` standard to see all policies annotated against it.
  - The policy named `compliance-cis-results` contains the automated scan results from the Compliance Operator against the `ocp4-cis` profile. Select it to review which controls passed, failed, or returned inconsistent results.
  - The remaining policies correspond to controls that the Compliance Operator cannot evaluate automatically — these require manual review and evidence collection by the platform team.

---

## Demo 1: Remediate Violations reported by the Compliance Operator CIS profile scan

### Step A: Fix Allowed Registries

**Objectives:**

- Verify that the violations `ocp4-cis-ocp-allowed-registries-for-import` and `ocp4-cis-ocp-allowed-registries` are flagged by the `compliance-cis-results` policy.
- Use the ACM ConfigurationPolicy to fix this violation flagged by the Compliance Operator scan.

**Test Procedure**

1. Go to ACM -> Governance, select the `CIS OpenShift Container Platform 4 Benchmark` standard and select the policy `compliance-cis-results`. Under this Policy you will find the template `cis-results`, that shows the violations raised against each cluster.
    - Select `cis-results`
    - On the cluster template `cis-results` for the `cluster1` cluster, press `View Details` to show the violations raised by `ocp4-cis`.
    - One of such violations is `ocp4-cis-ocp-allowed-registries` - press on it and a new window will open with the recommendation to fix it.
2. Create a new Policy to fix this control. This policy will be applied to all clusters because the placement is selecting the label `vendor: OpenShift`, which is configured in all clusters.
  - Copy the policy to the operators path, this will result that the applicationSet will create the Policy on the Hub
    ```bash
    cp -r auxiliar/registries policies/
    ```
3. Push changes to GitHub - **To save time, as ArgoCD will take a couple of minutes to sync each time a new directory is added, you can jump to step B and create the kubeadmin remove policy as well. And only then sync the git repository**
    ```bash
    git add policies/registries/*
    git commit -m "added allowed registries"
    git push
    ```
4. Result: A new policy will be added to ACM HUB. 
    This policy will be applied to all clusters and will enforce the adding the trusted registries to import images and imagestreams. 
    ```bash
    oc -n acm-policies get policy allowed-registries
    ```

**NOTE: The `ocp4-cis-ocp-allowed-registries` will keep showing a violation until the Openshift compliance scan runs again.**

### Step B: Fix Kubeadmin

> **WARNING - This deletion is permanent.** The kubeadmin secret cannot be restored without reinstalling the cluster. Before labelling a cluster, verify that an identity provider is configured and at least one user can log in with `cluster-admin` privileges: `oc login -u <user>`, or you have a valid `kubeconfig` file.

**Objectives:**

- Use the ACM configuration manager policy to fix the violation raised by the Compliance operator about the presence of a kubeadmin secret.
- This policy will also demonstrate how labels are used to control the placement of the policies. In this case the policy is enforced on the clusters labeled with `kubeadmin-enforce=true`.

**Test Procedure**

1. Go to ACM -> Governance, select the `CIS OpenShift Container Platform 4 Benchmark` standard and select the policy `compliance-cis-results`. Under this Policy you will find the template `cis-results`, that shows the violations for each cluster.
    - Select `cis-results`
    - Select the template `cis-results` of the cluster named `cluster1`, press on the `View Details` textfield, this will show the Violations raised by `cis-ocp4`.
    - one of such violations is `ocp4-cis-kubeadmin-removed`, press on top of this violation and a new window will open with the recommendation to fix it.
    - Confirm the secret exists on a target cluster before enforcing, for example in cluster1:
      ```bash
      oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get secret kubeadmin -n kube-system
      ```
2. Create a Policy and required files under policies path
  - Copy the policy to the operators path, once changes are pushed to Git, this will result that the applicationSet will create the Policy on the Hub
    ```bash
    cp -r auxiliar/kubeadmin policies/
    ```
3. Push changes to GitHub
    ```bash
    git add policies/kubeadmin/*
    git commit -m "policy kubeadmin"
    git push
    ```
4. A new policy will be added to ACM HUB
    ```bash
    oc -n acm-policies get policy kubeadmin-remove-enforce
    ```
5. Fix this violation on the selected clusters by labelling the clusters where this policy should be placed.
  To decide the clusters where this policy will be placed one must set a label into the clusters where it should be enforced.
    - Label the clusters to enforce the kubeadmin removal:
      ```bash
      oc label managedcluster cluster1 kubeadmin-enforce=true --overwrite
      ```
    - Confirm the policy is Compliant for `cluster1`:
    Wait for ArgoCD to sync and the policy to propagate, then verify the secret was deleted.
      ```bash
      oc get policy kubeadmin-remove-enforce -n acm-policies -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
      ```

**NOTE: The `ocp4-cis-kubeadmin-removed` will keep showing a violation until the Openshift compliance scan runs again**.

### Step C: Run the Compliance Operator OCP CIS Benchmark again and check that the fixed violations are cleared

**Objectives:**

- Run the Compliance operator scan, and verify that the previous checks were fixed. The scan may take several minutes.

1. Scan all clusters
    ```bash
    oc annotate compliancescans/ocp4-cis compliance.openshift.io/rescan= -n openshift-compliance
    oc --kubeconfig=/tmp/cluster1-kubeconfig annotate compliancescans/ocp4-cis compliance.openshift.io/rescan= -n openshift-compliance
    ```
2. check the state of the new scan
    ```bash
    oc get ComplianceSuite -n openshift-compliance
    oc --kubeconfig=/tmp/cluster1-kubeconfig get ComplianceSuite -n openshift-compliance
    ```
3. Once the scan is completed check the state of the controls
  - The `cluster1` is not violating the `ocp4-cis-kubeadmin-removed` nor the allowed `ocp4-cis-ocp-allowed-registries`. 
  - And the local-cluster will not show the `ocp4-cis-ocp-allowed-registries` violation
    ```bash
    oc get compliancecheckresult ocp4-cis-ocp-allowed-registries -n openshift-compliance -o jsonpath='{.status}'

    oc --kubeconfig=/tmp/cluster1-kubeconfig get compliancecheckresult ocp4-cis-ocp-allowed-registries -n openshift-compliance -o jsonpath='{.status}'
    ```

---

## Demo 2: Bring Your Own policy for CIS benchmark manual checks

### Step A: Trigger `rbac-no-wildcard-roles` - Wildcard Role Detection

**Objectives:**

- Verify that a Role with wildcard verbs, resources, and apiGroups in a user namespace is flagged by the `detect-anonymous-and-wildcard-rbac` policy, `rbac-no-wildcard-roles` ConfigurationPolicy template.
- Verify that adding the Role to the trusted list (`$allowedRoles`) clears the violation.

**Test Procedure**

1. Open the **ACM Console → Governance** dashboard. Select the CIS OpenShift Container Platform 4 Benchmark standard and Locate the policy `detect-anonymous-and-wildcard-rbac` and review the current violations for the `rbac-no-wildcard-roles` template. Note the existing violations (if any).
2. Create a test namespace and a Role with full wildcard permissions:
    ```bash
    oc new-project test-wildcard-role
    oc create role test-wildcard --verb='*' --resource='*.*' -n test-wildcard-role
    ```
3. Wait for the next policy evaluation cycle (1–2 minutes) or check the policy status with:
    ```bash
    oc get policy detect-anonymous-and-wildcard-rbac -n acm-policies \
      -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
    ```
   The policy should show **NonCompliant** and the violation detail in the ACM Governance UI should list `test-wildcard` in namespace `test-wildcard-role`.
4. Add the Role to the trusted list. Edit `policies/rbac/manifests/cis-rbac-controls.yaml` and add `"test-wildcard-role/test-wildcard"` to the `$allowedRoles` list:
    ```yaml
    {{- $allowedRoles := list
          "test-wildcard-role/test-wildcard"
    }}
    ```
5. Commit and push:
    ```bash
    git add policies/rbac/manifests/cis-rbac-controls.yaml
    git commit -m "test: add test-wildcard-role/test-wildcard to trusted roles"
    git push
    ```
6. Wait for ArgoCD to sync and the next policy evaluation cycle (1–2 minutes). Verify the violation for `test-wildcard` is cleared in the ACM Governance UI.
7. **Clean up** - remove the test namespace and revert the trusted list:
    ```bash
    oc delete project test-wildcard-role
    ```
    Then remove `"test-wildcard-role/test-wildcard"` from `$allowedRoles` in `cis-rbac-controls.yaml`, commit and push.

### Step B: Fix `rbac-no-unauth-access` - Unauthenticated CRB Detection

**Objectives:**

- Verify that a `ClusterRoleBinding` granting access to `system:unauthenticated` is flagged by the `detect-anonymous-and-wildcard-rbac` policy, (template `rbac-no-unauth-access`).
- Verify how to whitelist CRB and CR, adding inline in the policy the trusted ones. This whitelist contains a list of trusted CBR and CR - `$allowedCRBs` and `$allowedRoles`

**Test Procedure**

1. Open the **ACM Console → Governance** dashboard. Select the `CIS OpenShift Container Platform 4 Benchmark` standard. Locate the policy `detect-anonymous-and-wildcard-rbac` and review the current violations, for the `rbac-no-unauth-access` template. Note the existing violations (if any).
2. Create on the ACM Hub (local-cluster), a test ClusterRoleBinding that grants the `view` ClusterRole to the group `system:unauthenticated`:
    ```bash
    oc create clusterrolebinding test-unauth-access --clusterrole=view --group=system:unauthenticated
    ```
3. Wait for the next policy evaluation cycle (1–2 minutes) or check the policy status with:
    ```bash
    oc get policy detect-anonymous-and-wildcard-rbac -n acm-policies \
      -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
    ```
   The policy `detect-anonymous-and-wildcard-rbac` should show **NonCompliant**, for the local-cluster, and the violation detail in the ACM Governance UI should list the violating CRB `test-unauth-access`.
4. Fix this violation by adding a new the CRB name to the trusted list. Edit `policies/rbac/manifests/cis-rbac-controls.yaml` and add `"test-unauth-access"` to the `$allowedCRBs` list:
    ```yaml
    {{- $allowedCRBs := list
          "test-unauth-access"
    }}
    ```
5. Commit and push:
    ```bash
    git add policies/rbac/manifests/cis-rbac-controls.yaml
    git commit -m "test - add test-unauth-access to trusted CRBs"
    git push
    ```
6. Wait for ArgoCD to sync. Verify the violation for `test-unauth-access` CRB is cleared in the ACM Governance UI.
7. **Clean up** - remove the test CRB and revert the trusted list:
    ```bash
    oc delete clusterrolebinding test-unauth-access
    ```
    Then remove `"test-unauth-access"` from `$allowedCRBs` in `cis-rbac-controls.yaml`, commit and push.

### Step C: Fix Unauthorized CRB to cluster-admin CR

**Objectives:**

- Clear violation of a `ClusterRoleBinding` not whitlisted that is binded to a cluster-admin `ClusterRole`.
- Adding the CRB to the list of trusted CRBs will clear the violation. A ConfigMap contains the list of the whitelisted entities.

**Test Procedure**

1. Open the **ACM Console → Governance** dashboard. Select the CIS OpenShift Container Platform 4 Benchmark standard and locate the policy `cis-cluster-admin`. A violation is raising in the `local-cluster` for the CRB `cluster-admin-keycloak-admin`.
2. Configure the CRB to be trusted by adding the CRB name to the configmap with the trusted CRB `policies/rbac/manifests/cm-rbac-exceptions.yaml`:
    ```yaml
    data:
      cis-cluster-admin: |
        cluster-admin-keycloak-admin
    ```
3. Push changes to GitHub:
    ```bash
    git add policies/rbac/manifests/cm-rbac-exceptions.yaml
    git commit -m "rbac cluster-admin exception"
    git push
    ```
4. Wait for ArgoCD to sync. The violation is cleared.
5. **Clean up** - remove the test resources and revert the exception:
    ```bash
    oc delete project test-cis
    ```
    Then remove the CRB name from `cm-rbac-exceptions.yaml`, commit and push.
---

## Demo 3: Prohibit the creation of an object - using ValidatingAdmissionPolicy

**Objectives:**

- Verify that the VAP denies creation of `ClusterRoleBinding` binded with `cluster-admin` ClusterRole, for subjects not listed on the approved allow-list at admission time, while allowing bindings to other ClusterRoles (e.g. `view`, `edit`) for any user.

> **Prerequisite:** ValidatingAdmissionPolicy (`admissionregistration.k8s.io/v1`) requires **Kubernetes 1.30+ / OpenShift 4.17+**. Confirm your managed clusters meet this requirement before proceeding.

**Test Procedure**

1. Verify the ACM policy is created and distributed:
    ```bash
    oc get policy vap-cluster-admin-allow-list -n acm-policies
    oc get policy vap-cluster-admin-allow-list -n acm-policies \
      -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
    ```
2. Confirm the ValidatingAdmissionPolicy and its binding exist on a managed cluster:
    ```bash
    oc --kubeconfig=/tmp/cluster1-kubeconfig get validatingadmissionpolicy vap-cluster-admin-allow-list
    oc --kubeconfig=/tmp/cluster1-kubeconfig get validatingadmissionpolicybinding vap-cluster-admin-allow-list-binding
    ```
3. **Test: bind an unauthorized user to the ClusterRole `cluster-admin`.** This should be **denied** - the user `toni` is not on the allow-list. Users/SA/Groups trust-list are under parameter `object.subjects.all(`, under the file `policies/vap/manifests/vap-cluster-admin-allow-list.yaml`.
    ```bash
    oc --kubeconfig=/tmp/cluster1-kubeconfig create clusterrolebinding test-vap-deny \
      --clusterrole=cluster-admin --user=toni
    ```
    Expected output:
    ```
    error: failed to create clusterrolebinding: clusterrolebindings.rbac.authorization.k8s.io "test-vap-deny" is forbidden: ValidatingAdmissionPolicy 'vap-cluster-admin-allow-list' with binding 'vap-cluster-admin-allow-list-binding' denied request: ClusterRoleBinding to cluster-admin contains subjects not on the allow-list. Only approved users, groups, and service accounts can be bound to cluster-admin.
    ```
4. **Test: bind the same unauthorized user to a non-`cluster-admin` role.** This should be **accepted** - the VAP only restricts bindings to `cluster-admin`, other ClusterRoles (e.g. `view`, `edit`) are -not affected:
    ```bash
    oc --kubeconfig=/tmp/cluster1-kubeconfig create clusterrolebinding test-vap-view \
      --clusterrole=view --user=toni
    ```
    Expected output: `clusterrolebinding.rbac.authorization.k8s.io/test-vap-view created`
5. **Test: bind an allowed user to `cluster-admin`.** This should be **accepted** - `system:admin` is on the allow-list:
    ```bash
    oc --kubeconfig=/tmp/cluster1-kubeconfig create clusterrolebinding test-vap-allow \
      --clusterrole=cluster-admin --user=system:admin
    ```
    Expected output: `clusterrolebinding.rbac.authorization.k8s.io/test-vap-allow created`
6. **Clean up** - remove the test CRBs:
    ```bash
    oc --kubeconfig=/tmp/cluster1-kubeconfig delete clusterrolebinding test-vap-view test-vap-allow
    ```

A ValidatingAdmissionPolicy is an **admission controller** - it only validates API requests at `CREATE` or `UPDATE` time. It does **not** retroactively scan resources that already exist on the cluster.

---

# Auxiliary Commands

## ArgoCD

```bash
#To force, via `argocd` CLI, to sync a application
oc -n openshift-gitops patch applications.argoproj.io <app_name> --type merge   -p '{"operation":{"initiatedBy":{"username"
:"admin","automated":false},"sync":{"revision":"HEAD","prune":true}}}'
```

## Compliance Operator

```bash
#Overall suite status (covers both scans)
oc get ComplianceSuite cis-compliance -n openshift-compliance

# Individual scan status
oc get ComplianceScan -n openshift-compliance

# Summary of all check results
oc get ComplianceCheckResult -n openshift-compliance --no-headers

# Failed checks only
oc get ComplianceCheckResult -n openshift-compliance -l compliance.openshift.io/check-status=FAIL

# Rerun scans:
#1. Rescan ocp4-cis (platform checks)
oc annotate compliancescans/ocp4-cis compliance.openshift.io/rescan= -n openshift-compliance

#2. Rescan ocp4-cis-node (node-level checks)
oc annotate compliancescans/ocp4-cis-node-master compliance.openshift.io/rescan= -n openshift-compliance
oc annotate compliancescans/ocp4-cis-node-worker compliance.openshift.io/rescan= -n openshift-compliance
```

---

# References

- [ACM Policy Samples](https://github.com/stolostron/????????????)
- [ACM Policy Samples](https://github.com/bry-tam/acm-policy-samples/tree/main/policies/operators/)