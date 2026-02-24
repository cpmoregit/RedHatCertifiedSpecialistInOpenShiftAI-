# RHOAI: Managing Users, Groups, RBAC, and Resource Controls (OpenShift)

This document provides a practical, copy-paste ready guide for managing users, groups, RBAC, multi-tenancy, resource quotas, GPU scheduling, service accounts, auditing, and best practices for a Red Hat OpenShift AI (RHOAI) deployment.

> Assumptions
> - OpenShift (OCP) 4.13/4.14 or similar
> - RHOAI operator installed or planned installation
> - GPU nodes optional (NVIDIA); instructions include GPU-related controls but are safe to ignore in CPU-only clusters
> - Use your organization's IDP (LDAP/AD/Keycloak) for production; the examples include local/test user creation for convenience

---

## Table of Contents

- [1. Concepts & goals](#1-concepts--goals)
- [2. Recommended user/group layout](#2-recommended-usergroup-layout)
- [3. Create users and groups (examples)](#3-create-users-and-groups-examples)
- [4. RBAC: Roles and RoleBindings for RHOAI](#4-rbac-roles-and-rolebindings-for-rhoai)
- [5. Namespace (project) roles for tenants](#5-namespace-project-roles-for-tenants)
- [6. Read-only roles (auditors)](#6-read-only-roles-auditors)
- [7. ResourceQuotas and LimitRanges (per-namespace)](#7-resourcequotas-and-limitranges-per-namespace)
- [8. GPU scheduling controls and fairness](#8-gpu-scheduling-controls-and-fairness)
- [9. ServiceAccounts and secret access controls](#9-serviceaccounts-and-secret-access-controls)
- [10. Auditing and logging](#10-auditing-and-logging)
- [11. Example: bootstrap a tenant namespace](#11-example-bootstrap-a-tenant-namespace)
- [12. Enforce policies with OPA/Gatekeeper (recommended)](#12-enforce-policies-with-opagatekeeper-recommended)
- [13. Best practices & tips](#13-best-practices--tips)
- [14. Next-step examples I can provide](#14-next-step-examples-i-can-provide)

---

## 1. Concepts & goals

- Authentication: OpenShift uses OAuth; integrate with LDAP/AD/SAML/OIDC for production.
- Authorization: Kubernetes RBAC (Role/ClusterRole + RoleBinding/ClusterRoleBinding).
- Multi-tenancy goals:
  - Isolate RHOAI control plane (namespace `rhoai`) from tenant projects.
  - Provide limited admin capabilities to a small RHOAI ops team.
  - Give data scientists scoped permissions to deploy models and run training in tenant namespaces.
  - Enforce quotas/limit ranges for CPU, memory, GPUs, PVs.
  - Secure access to model stores and secrets via service accounts.

---

## 2. Recommended user/group 

- `cluster-admin` (SRE / Platform admins) — minimal membership.
- `rhoai-ops-admins` — manage operator, CRs, storage, routes in `rhoai` namespace.
- Tenant groups (e.g., `data-science-team-1`, `ml-platform-devs`) — deploy models and jobs.
- `rhoai-auditors` — read-only access to models/logs/metrics.

---

## 3. Create users and groups (examples)

> In production: configure an IdentityProvider (IDP) to manage users/groups (LDAP/AD/Keycloak). The following is for testing.

Create users:
```bash
oc create user alice
oc create user bob```
```
Create groups and add users:
```
oc adm groups new ml-team data-science-team-1
oc adm groups add-users ml-team alice,bob
```
Verify:
 ```
oc get groups
```
## 4. RBAC: Roles and RoleBindings for RHOAI
Role is namespaced. ClusterRole is cluster-scoped and can be bound in a namespace via a RoleBinding or cluster-wide via a ClusterRoleBinding.

Keep platform-level privileges limited and prefer namespace-scoped roles for tenant teams.

Following rules entries are used for defining roles
- apiGroups
  Get all apiGroups
```
oc api-versions
```

To search the apiGroup use grep command
```
oc api-versions | grep coreos.com
```
- resources
<pre>
Resources are Namespace-scoped resources and Cluster-scoped resources.

Namespace-scoped resources: Pods, ConfigMaps, Secrets, Deployments (in apps), etc. These resources exist inside a namespace.
Cluster-scoped resources: Nodes, PersistentVolumes, ClusterRoles, Namespaces, CRDs, etc. These exist at the cluster level.
</pre>
    Get all resource
```
oc api-resources
```

To search a resource use grep command
```
oc api-resources | grep routes
```
  Common OpenShift AI (RHOAI) resource names (CRs) — use these in resources: []
|Resource Names|    Comments                                   |
|--------------|-----------------------------------------------|
|rhoais| (sometimes the cluster-scoped manager custom resource)|
|rhoaiconfigs| operator configuration CR, name may vary|
|models |model objects, model resources|
|modelbackups| if operator supports model backup/restore|
|inferenceservices| inference service objects / deployments
|inferencejobs| batch inference jobs, if present|
|modeldeployments| alternative name for deployed models, may appear|
|modelregistrations| model registry entries|
|modelversions| versioned model CR|
|modelrepositories (registry/registry config CR)|
|modelservings|alternate name used by some operators|
|deploymentrequests| requests to deploy models, operator-specific|
|datasetconfigs| dataset/feature-store related CRs|
|trainingjobs| training job CRs, if operator supports training|
|notebooks| if included by the AI stack; could overlap with other operators|
|vaultconfigs| secrets/vault integration CR, operator-specific|
|runtimes| runtime or runtimeclasses for model serving|
|apiservices| if the operator registers aggregated APIService CRs|
- verb
<pre>
    Verbs define which API operations are allowed on resources
    Create --> create
    Read -->  get, List or watch
    Update --> update
    Delete --> delete
    Patch --> patch
</pre>

### ClusterRole for RHOAI operator managers
Save as rhoaiops-clusterrole.yaml:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rhoai-operator-manager
rules:
- apiGroups: ["ai.redhat.com"]          # adjust to actual RHOAI API group
  resources: ["rhoais", "models", "inferenceservices"]   # adjust to actual CRD names
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["operators.coreos.com"]
  resources: ["subscriptions", "clusterserviceversions"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets", "configmaps", "namespaces"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["route.openshift.io"]
  resources: ["routes"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
Apply:
```
oc apply -f rhoaiops-clusterrole.yaml
```
Bind the role to the platform admin group (rhoai-ops-admins):

Roles are bind to following subjects
* User
* Group
* ServiceAccount

Save as rhoaiops-clusterrolebinding.yaml:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rhoai-operator-manager-binding
subjects:
- kind: Group
  name: rhoai-ops-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: rhoai-operator-manager
  apiGroup: rbac.authorization.k8s.io
```
## 5. Namespace (project) roles for tenants
Create a Role that allows model developers to manage pods, deployments, jobs, PVCs, secrets, and (if used) KServe InferenceServices.

Save as rhoai-tenant-role.yaml (replace <tenant-namespace>):
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rhoai-tenant-developer
  namespace: <tenant-namespace>
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["serving.kserve.io"]    # if KServe is used
  resources: ["inferenceservices"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply for each tenant namespace:
```
oc apply -f rhoai-tenant-role.yaml
```
Bind the Role to the tenant group:

Save as rhoai-tenant-rolebinding.yaml (replace <tenant-namespace> and group name):
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rhoai-tenant-developers-binding
  namespace: <tenant-namespace>
subjects:
- kind: Group
  name: data-science-team-1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: rhoai-tenant-developer
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```
oc apply -f rhoai-tenant-rolebinding.yaml
```
## 6. Read-only roles (auditors)
Create a ClusterRole for read-only access across model-related CRDs and basic resources.

Save as rhoai-readonly-clusterrole.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rhoai-readonly
rules:
- apiGroups: ["ai.redhat.com", "serving.kserve.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "configmaps"]
  verbs: ["get", "list", "watch"]

```
Apply:
```
oc apply -f rhoai-readonly-clusterrole.yaml
```

Create group and bind:
```
oc adm groups new rhoai-auditors auditor1
oc create clusterrolebinding rhoai-readonly-binding \
  --clusterrole=rhoai-readonly \
  --group=rhoai-auditors
```
## 7. ResourceQuotas and LimitRanges (per-namespace)
Use ResourceQuota to set hard limits and LimitRange to provide default resource requests/limits.

ResourceQuota (including GPU count)
Save as tenant-quota.yaml (replace <tenant-namespace>):
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-resource-quota
  namespace: <tenant-namespace>
spec:
  hard:
    requests.cpu: "200"
    requests.memory: 512Gi
    limits.cpu: "400"
    limits.memory: 1Ti
    pods: "100"
    persistentvolumeclaims: "10"
    requests.storage: 50Ti
    limits.storage: 100Ti
    count/nvidia.com/gpu: "8"    # verify extended resource name on your cluster
```

Note: Extended resource names (GPU) must be checked on your cluster (commonly nvidia.com/gpu).

Apply:
```
oc apply -f tenant-quota.yaml
```
