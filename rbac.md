RHOAI: Managing Users, Groups, RBAC, and Resource Controls (OpenShift)
======================================================================

This document describes practical steps to manage users, groups, RBAC, multi-tenancy, resource quotas, GPU scheduling, service accounts, auditing, and best practices for a Red Hat OpenShift AI (RHOAI) deployment.

1\. Concepts & goals (brief)
----------------------------

*   Authentication: OpenShift uses OAuth; integrate with LDAP/AD/SAML/OIDC for production.
    
*   Authorization: Kubernetes RBAC (Role/ClusterRole + RoleBinding/ClusterRoleBinding).
    
*   Multi-tenancy goals:
    
    *   Isolate RHOAI control plane (namespace rhoai) from tenant projects.
        
    *   Provide limited admin capabilities to a small RHOAI ops team.
        
    *   Give data scientists scoped permissions to deploy models and run training in tenant namespaces.
        
    *   Enforce quotas/limit ranges for CPU, memory, GPUs, PVs.
        
    *   Secure access to model stores and secrets via service accounts.
        

2\. Recommended user/group layout
---------------------------------

*   cluster-admin (SRE / Platform admins) — very limited membership.
    
*   rhoai-ops-admins — manage operator, CRs, storage, routes in rhoai namespace.
    
*   Tenant groups (e.g., data-science-team-1, ml-platform-devs) — deploy models and jobs.
    
*   rhoai-auditors — read-only access to models/logs/metrics.
    

3\. Create users and groups (examples)
--------------------------------------

> Note: In production, configure an IdentityProvider (IDP) to manage users/groups (LDAP/AD/Keycloak). The following is for testing.

Create users:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc create user alice  oc create user bob   `

Create groups and add users:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc adm groups new ml-team data-science-team-1  oc adm groups add-users ml-team alice,bob   `

Verify:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc get groups   `

4\. RBAC: Roles and RoleBindings for RHOAI
------------------------------------------

Keep platform-level privileges limited and prefer namespace-scoped roles for tenant teams.

### ClusterRole for RHOAI operator managers

Save as rhoaiops-clusterrole.yaml:

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: rbac.authorization.k8s.io/v1  kind: ClusterRole  metadata:    name: rhoai-operator-manager  rules:  - apiGroups: ["ai.redhat.com"]          # adjust to actual RHOAI API group    resources: ["rhoais", "models", "inferenceservices"]   # adjust to actual CRD names    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: ["operators.coreos.com"]    resources: ["subscriptions", "clusterserviceversions"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: [""]    resources: ["secrets", "configmaps", "namespaces"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: ["route.openshift.io"]    resources: ["routes"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]   `

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f rhoaiops-clusterrole.yaml   `

Bind the role to the platform admin group (rhoai-ops-admins):Save as rhoaiops-clusterrolebinding.yaml:

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: rbac.authorization.k8s.io/v1  kind: ClusterRoleBinding  metadata:    name: rhoai-operator-manager-binding  subjects:  - kind: Group    name: rhoai-ops-admins    apiGroup: rbac.authorization.k8s.io  roleRef:    kind: ClusterRole    name: rhoai-operator-manager    apiGroup: rbac.authorization.k8s.io   `

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f rhoaiops-clusterrolebinding.yaml   `

5\. Namespace (project) roles for tenants
-----------------------------------------

Create a Role that allows model developers to manage pods, deployments, jobs, PVCs, secrets, and (if used) KServe InferenceServices.

Save as rhoai-tenant-role.yaml (replace ):

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: rbac.authorization.k8s.io/v1  kind: Role  metadata:    name: rhoai-tenant-developer    namespace:   rules:  - apiGroups: [""]    resources: ["pods", "pods/log", "services", "configmaps", "secrets", "persistentvolumeclaims"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: ["apps"]    resources: ["deployments", "statefulsets"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: ["batch"]    resources: ["jobs", "cronjobs"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  - apiGroups: ["serving.kserve.io"]    # if KServe is used    resources: ["inferenceservices"]    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]   `

Apply for each tenant namespace:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f rhoai-tenant-role.yaml   `

Bind the Role to the tenant group:Save as rhoai-tenant-rolebinding.yaml (replace and group name):

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: rbac.authorization.k8s.io/v1  kind: RoleBinding  metadata:    name: rhoai-tenant-developers-binding    namespace:   subjects:  - kind: Group    name: data-science-team-1    apiGroup: rbac.authorization.k8s.io  roleRef:    kind: Role    name: rhoai-tenant-developer    apiGroup: rbac.authorization.k8s.io   `

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f rhoai-tenant-rolebinding.yaml   `

6\. Read-only roles (auditors)
------------------------------

Create a ClusterRole for read-only access across model-related CRDs and basic resources.

Save as rhoai-readonly-clusterrole.yaml:

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: rbac.authorization.k8s.io/v1  kind: ClusterRole  metadata:    name: rhoai-readonly  rules:  - apiGroups: ["ai.redhat.com", "serving.kserve.io"]    resources: ["*"]    verbs: ["get", "list", "watch"]  - apiGroups: [""]    resources: ["pods", "pods/log", "services", "configmaps"]    verbs: ["get", "list", "watch"]   `

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f rhoai-readonly-clusterrole.yaml   `

Create group and bind:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc adm groups new rhoai-auditors auditor1  oc create clusterrolebinding rhoai-readonly-binding \    --clusterrole=rhoai-readonly \    --group=rhoai-auditors   `

7\. ResourceQuotas and LimitRanges (per-namespace)
--------------------------------------------------

Use ResourceQuota to set hard limits and LimitRange to provide default resource requests/limits.

### ResourceQuota (including GPU count)

Save as tenant-quota.yaml (replace ):

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: v1  kind: ResourceQuota  metadata:    name: tenant-resource-quota    namespace:   spec:    hard:      requests.cpu: "200"      requests.memory: 512Gi      limits.cpu: "400"      limits.memory: 1Ti      pods: "100"      persistentvolumeclaims: "10"      requests.storage: 50Ti      limits.storage: 100Ti      count/nvidia.com/gpu: "8"    # verify extended resource name on your cluster   `

> Note: extended resource names (GPU) must be checked on your cluster (commonly nvidia.com/gpu).

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f tenant-quota.yaml   `

### LimitRange (default requests/limits)

Save as tenant-limitrange.yaml (replace ):

yaml

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   apiVersion: v1  kind: LimitRange  metadata:    name: tenant-limitrange    namespace:   spec:    limits:    - type: Container      default:        cpu: "4"        memory: 8Gi      defaultRequest:        cpu: "1"        memory: 2Gi   `

Apply:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc apply -f tenant-limitrange.yaml   `

8\. GPU scheduling controls and fairness
----------------------------------------

*   Use node labels, taints/tolerations, and nodeSelectors so only authorized namespaces can schedule GPU workloads.
    
*   Enforce with an admission policy (Gatekeeper/OPA) to deny pods requesting GPUs unless namespace has a label (recommended).
    

### Simple namespace label for GPU access

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc label namespace tenant-ml1 gpu-access=true   `

For strict enforcement, use Gatekeeper with a Constraint that checks pod resource requests/limits for nvidia.com/gpu and denies creation unless namespace label gpu-access=true exists. (See Section 12 for Gatekeeper option.)

9\. ServiceAccounts and secret access controls
----------------------------------------------

Avoid exposing secrets to users. Create a service account per tenant workload that has access to only the secrets required (S3 creds, etc).

Example: create service account and link secret:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc create serviceaccount model-runner -n   oc create secret generic tenant-s3-secret \    --from-literal=access_key= \    --from-literal=secret_key= \    -n   oc secrets link serviceaccount model-runner tenant-s3-secret -n  --for=pull   `

Use serviceAccountName: model-runner in Pod/Job specs.

For cloud environments prefer IAM-backed access (IRSA for AWS) or short-lived credentials.

10\. Auditing and logging
-------------------------

*   Enable OpenShift audit logs to track modifications to CRDs and secrets.
    
*   Use OpenShift Cluster Logging (EFK/Elasticsearch + Fluentd + Kibana) or Loki/Prometheus+Grafana for metrics and logs.
    
*   Limit log access with RBAC (auditors should have read-only role).
    

11\. Example: bootstrap a tenant namespace
------------------------------------------

Fill in and :

Commands:

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   oc new-project   oc label namespace  purpose=ml  oc apply -f tenant-limitrange.yaml    # use file with namespace replaced  oc apply -f tenant-quota.yaml         # use file with namespace replaced  oc apply -f rhoai-tenant-role.yaml    # file has namespace set  oc apply -f rhoai-tenant-rolebinding.yaml  # file has namespace and group set  oc adm groups new data-science-team-1 alice,bob  oc adm policy add-role-to-group view data-science-team-1 -n   oc create rolebinding tenant-ml1-edit --clusterrole=edit --group=data-science-team-1 -n   oc label namespace  gpu-access=true  oc create serviceaccount model-runner -n   oc create secret generic tenant-s3-secret \    --from-literal=access_key= --from-literal=secret_key= -n   oc secrets link model-runner tenant-s3-secret -n  --for=pull   `

12\. Enforce policies with OPA/Gatekeeper (recommended)
-------------------------------------------------------

Gatekeeper lets you enforce policies such as "only namespaces with label gpu-access=true may create pods requesting nvidia.com/gpu".

If you want, I can produce:

*   A Gatekeeper ConstraintTemplate and Constraint with Rego policy to deny pod creation if they request GPU without namespace label.
    
*   A sample Rego that checks pod.spec.containers\[\].resources.requests and limits for the GPU key.
    

(Ask for Gatekeeper example if you want the exact YAML.)

13\. Best practices & tips
--------------------------

*   Principle of least privilege: prefer namespace-scoped roles.
    
*   Manage group membership in your IDP, not manually in OpenShift.
    
*   Protect secrets; use a vault (HashiCorp Vault or cloud KMS) for long-term secret storage.
    
*   Use Gatekeeper to enforce security and compliance (e.g., disallow privileged containers, enforce GPU-label policy).
    
*   Use ResourceQuota and LimitRange per environment (dev/test/prod).
    
*   Implement chargeback using namespace labels + Prometheus/Grafana metrics.
    
*   Use NetworkPolicies to restrict network access between namespaces.
    

14\. Next-step examples I can provide
-------------------------------------

Tell me which of the following you want next (A, B, or C) and I’ll provide the exact YAML and commands:

*   A) Gatekeeper policy to restrict GPU usage to labeled namespaces (ConstraintTemplate + Constraint + Rego)
    
*   B) ServiceAccount + IRSA (AWS) example for S3 access (IRSA IAMRole creation, IAM policy, Kubernetes service account annotation)
    
*   C) Full tenant bootstrap script (bash) that creates namespace, quotas, roles, rolebindings, service accounts, and links secrets (parameterized)
