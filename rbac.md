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

## 2. Recommended user/group layout

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
Keep platform-level privileges limited and prefer namespace-scoped roles for tenant teams.

Following rules entries are used for defining roles
- apiGroups,
- resources 
- verb
    Verb are CRUDP operations as follow
      1. C- Create
      2. R- Read - get, List, watch
      3. U- Update
      4. D- Delete
      5. P - Patch
        

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


