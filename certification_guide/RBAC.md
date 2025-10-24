# RBAC (Role-Based Access Control) - Complete Reference Guide
## CKA Certification Study Notes

---

## What is RBAC?

**RBAC (Role-Based Access Control)** is Kubernetes' authorization system that determines what actions authenticated users and service accounts can perform in the cluster.

**The Core Concept**: RBAC answers "What can you do?" while Service Accounts answer "Who are you?"

### The Authorization Flow
```
Authentication (Service Account/User)
    ↓
"I know WHO you are"
    ↓
RBAC Authorization
    ↓
Check Roles & RoleBindings
    ↓
"Here's WHAT you can do"
    ↓
Allow or Deny Action
```

---

## The Four Core RBAC Objects

### 1. Role (Namespace-Scoped)

A Role defines permissions within a single namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: production
rules:
- apiGroups: [""]           # Core API group
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```

**Use Role when**: Permissions are limited to resources in one namespace.

### 2. ClusterRole (Cluster-Wide)

A ClusterRole defines permissions across the entire cluster or for cluster-scoped resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]      # Cluster-scoped resource
  verbs: ["get", "list"]
```

**Use ClusterRole when**:
- Need access across multiple namespaces
- Working with cluster-scoped resources (nodes, persistentvolumes, namespaces)
- Creating reusable permission templates

### 3. RoleBinding (Namespace-Scoped)

A RoleBinding grants permissions defined in a Role (or ClusterRole) to subjects within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

### 4. ClusterRoleBinding (Cluster-Wide)

A ClusterRoleBinding grants permissions defined in a ClusterRole to subjects across the entire cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## The Four Combinations (Critical!)

Understanding how Roles and Bindings combine is essential:

### Combination 1: Role + RoleBinding
**Scope**: Single namespace  
**Pattern**: Standard namespace-scoped permissions

```yaml
# Role in namespace A
kind: Role
namespace: team-a

# RoleBinding in namespace A
kind: RoleBinding
namespace: team-a
```

**Result**: Permissions apply only to resources in namespace A.

### Combination 2: ClusterRole + RoleBinding
**Scope**: Single namespace (but reusable definition)  
**Pattern**: Define once, reuse across namespaces

```yaml
# ClusterRole (no namespace, defined once)
kind: ClusterRole

# RoleBinding in namespace A
kind: RoleBinding
namespace: team-a

# RoleBinding in namespace B (same ClusterRole)
kind: RoleBinding
namespace: team-b
```

**Result**: Each RoleBinding limits the ClusterRole's permissions to its own namespace. This is the DRY (Don't Repeat Yourself) pattern for multi-tenant clusters.

### Combination 3: ClusterRole + ClusterRoleBinding
**Scope**: Entire cluster  
**Pattern**: Cluster-wide permissions

```yaml
# ClusterRole (no namespace)
kind: ClusterRole

# ClusterRoleBinding (no namespace)
kind: ClusterRoleBinding
```

**Result**: Permissions apply across all namespaces and to cluster-scoped resources. Use for monitoring tools, cluster administrators, etc.

### Combination 4: Role + ClusterRoleBinding
**Scope**: Invalid / Not supported  
**Pattern**: Cannot bind a namespace-scoped Role cluster-wide

This combination doesn't make logical sense and is not supported by Kubernetes.

---

## Understanding Rules

Every Role or ClusterRole contains rules that specify permissions:

```yaml
rules:
- apiGroups: ["apps", ""]
  resources: ["deployments", "pods"]
  verbs: ["get", "list", "create"]
  resourceNames: ["my-deployment"]  # Optional
```

### Component 1: API Groups

API Groups organize Kubernetes resources into logical collections.

**Core API Group** (empty string):
```yaml
apiGroups: [""]
resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims", "namespaces"]
```

**Apps API Group**:
```yaml
apiGroups: ["apps"]
resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
```

**Batch API Group**:
```yaml
apiGroups: ["batch"]
resources: ["jobs", "cronjobs"]
```

**Networking API Group**:
```yaml
apiGroups: ["networking.k8s.io"]
resources: ["networkpolicies", "ingresses"]
```

**RBAC API Group**:
```yaml
apiGroups: ["rbac.authorization.k8s.io"]
resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
```

**Storage API Group**:
```yaml
apiGroups: ["storage.k8s.io"]
resources: ["storageclasses", "volumeattachments"]
```

**Wildcard** (use with extreme caution):
```yaml
apiGroups: ["*"]  # All API groups
```

### Component 2: Resources

Resources are the Kubernetes objects you want to grant access to.

**Main resources**: pods, deployments, services, configmaps, secrets

**Subresources**: Some resources have subresources for specific operations
- `pods/log` - Access pod logs
- `pods/exec` - Execute commands in pods
- `pods/portforward` - Port forwarding
- `deployments/scale` - Scale deployments

**Wildcard**:
```yaml
resources: ["*"]  # All resources (dangerous!)
```

### Component 3: Verbs (Actions)

Verbs define what actions can be performed on resources.

**Read Operations**:
- `get` - Retrieve a specific resource by name
- `list` - List all resources of a type
- `watch` - Watch for changes to resources (streaming API)

**Write Operations**:
- `create` - Create new resources
- `update` - Replace an entire resource
- `patch` - Partially modify a resource
- `delete` - Delete a specific resource
- `deletecollection` - Delete multiple resources at once

**Special Verbs**:
- `use` - For PodSecurityPolicies and other policy resources
- `bind` - Bind roles to subjects
- `escalate` - Create roles with more permissions than you have
- `impersonate` - Act as another user or service account

**Wildcard**:
```yaml
verbs: ["*"]  # All verbs (very dangerous!)
```

### Component 4: ResourceNames (Optional)

Limit permissions to specific named resources for fine-grained control:

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "update"]
  resourceNames: ["app-config", "database-config"]  # Only these two
```

---

## Subjects - Who Receives Permissions?

Subjects are the entities that receive permissions through RoleBindings or ClusterRoleBindings.

### ServiceAccount Subject
```yaml
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
```

Used for applications running in pods.

### User Subject
```yaml
subjects:
- kind: User
  name: jane@example.com
  apiGroup: rbac.authorization.k8s.io
```

Used for human users. Users are managed externally (certificates, OIDC providers, etc.).

### Group Subject
```yaml
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
```

Used for groups of users. Groups are also managed externally.

### Multiple Subjects

A single binding can grant permissions to multiple subjects:

```yaml
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: platform-team
  apiGroup: rbac.authorization.k8s.io
```

---

## Real-World Examples

### Example 1: Developer (Read-Only)

A developer who needs to view resources but not modify them:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-read
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
```

### Example 2: CI/CD Deployer

A Jenkins or GitLab runner that deploys applications:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

### Example 3: Monitoring (Cluster-Wide Read-Only)

A monitoring tool like Prometheus that reads metrics across all namespaces:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
```

### Example 4: Namespace Admin

Someone who can manage everything in a namespace except RBAC:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: team-a
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: []  # Explicitly no RBAC permissions
```

---

## Default ClusterRoles

Kubernetes provides several built-in ClusterRoles that cover common use cases.

### cluster-admin
Full access to everything in the cluster. This is the superuser role.

```bash
kubectl create clusterrolebinding admin-binding \
  --clusterrole=cluster-admin \
  --user=admin@example.com
```

**When to use**: Platform administrators only. Use very sparingly.

### admin
Full access within a namespace, including the ability to create Roles and RoleBindings. Cannot modify ResourceQuotas or the namespace itself.

```bash
kubectl create rolebinding team-admin \
  --clusterrole=admin \
  --user=team-lead@example.com \
  -n team-namespace
```

**When to use**: Team leads who manage their namespace.

### edit
Read and write access to most resources in a namespace. Cannot view or modify Roles, RoleBindings, or Secrets.

```bash
kubectl create rolebinding developers \
  --clusterrole=edit \
  --group=developers \
  -n development
```

**When to use**: Developers who need to deploy and manage applications.

### view
Read-only access to most resources. Cannot view Secrets, Roles, or RoleBindings.

```bash
kubectl create rolebinding viewers \
  --clusterrole=view \
  --group=stakeholders \
  -n production
```

**When to use**: Stakeholders, junior developers, or monitoring tools that only need read access.

---

## Testing and Debugging Permissions

### Check Your Own Permissions

```bash
# Can I create deployments?
kubectl auth can-i create deployments

# Can I delete pods in production?
kubectl auth can-i delete pods -n production

# Can I do everything?
kubectl auth can-i "*" "*"

# What can I do in this namespace?
kubectl auth can-i --list -n development
```

### Check Someone Else's Permissions

```bash
# Check a service account
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:my-sa

# Check a user
kubectl auth can-i create deployments \
  --as=jane@example.com \
  -n production

# Check what a service account can do
kubectl auth can-i --list \
  --as=system:serviceaccount:monitoring:prometheus-sa \
  -n production
```

### Inspect Roles and Bindings

```bash
# View a Role's permissions
kubectl describe role developer-read -n development

# View a ClusterRole's permissions
kubectl describe clusterrole view

# See all RoleBindings in a namespace
kubectl get rolebindings -n production

# See all ClusterRoleBindings
kubectl get clusterrolebindings

# Find who has access to a namespace
kubectl get rolebindings -n production -o wide
```

---

## Common Patterns and Best Practices

### Pattern 1: Principle of Least Privilege

Always grant the minimum permissions needed to perform a task.

**Bad Example** (too permissive):
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

**Good Example** (specific permissions):
```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  resourceNames: ["app-config"]
```

### Pattern 2: Separate Read and Write Roles

Create distinct roles for read and write operations to allow flexible permission assignment.

```yaml
# Read role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
# Write role (separate)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-writer
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create", "update", "patch", "delete"]
```

Then bind them independently based on needs.

### Pattern 3: Reusable ClusterRoles

Define permissions once as ClusterRoles and reuse them across namespaces with RoleBindings.

```yaml
# Define once
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# Use in team-a
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-readers
  namespace: team-a
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: team-a
---
# Use in team-b (same ClusterRole)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-b-readers
  namespace: team-b
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: team-b
```

### Pattern 4: Deny by Default

Kubernetes RBAC is deny-by-default. Entities have no permissions unless explicitly granted. This is a security feature.

### Pattern 5: Avoid Wildcards

Minimize use of wildcards (`*`) for API groups, resources, and verbs. Be explicit about what's needed.

---

## Imperative Commands (Fast for CKA)

```bash
# Create a Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n development

# Create a ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list \
  --resource=nodes

# Create a RoleBinding
kubectl create rolebinding dev-pod-reader \
  --role=pod-reader \
  --serviceaccount=development:app-sa \
  -n development

# Create a ClusterRoleBinding
kubectl create clusterrolebinding monitoring-nodes \
  --clusterrole=node-reader \
  --serviceaccount=monitoring:prometheus-sa

# Bind a ClusterRole with a RoleBinding (namespace-limited)
kubectl create rolebinding team-edit \
  --clusterrole=edit \
  --user=developer@example.com \
  -n team-namespace
```

---

## Troubleshooting Common Issues

### Issue 1: "Forbidden" Errors

**Symptom**: Application or user gets "forbidden" when trying to perform an action.

**Debug Steps**:
```bash
# Check if the entity has permissions
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa> \
  -n <namespace>

# Verify Role/ClusterRole exists
kubectl get role <role-name> -n <namespace>
kubectl describe role <role-name> -n <namespace>

# Verify RoleBinding/ClusterRoleBinding exists
kubectl get rolebinding -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>

# Check if subject is correctly specified
kubectl get rolebinding <binding-name> -n <namespace> -o yaml
```

### Issue 2: Permissions Not Working After Creating RBAC

**Common Causes**:
- Wrong namespace in serviceaccount reference
- Typo in role name in roleRef
- Wrong API group specified
- RoleBinding in wrong namespace

**Solution**: Carefully review YAML for typos and verify with describe commands.

### Issue 3: Cross-Namespace Access Not Working

**Common Mistake**: Using Role instead of ClusterRole, or forgetting to specify the correct namespace in the RoleBinding.

**Solution**: 
- Create Role in target namespace
- Create RoleBinding in target namespace
- Reference ServiceAccount from source namespace in subjects

### Issue 4: Can't Create Resources Even With Permissions

**Check**: Does the SA need access to related resources? For example, creating Deployments requires access to ReplicaSets.

---

## CKA Exam Tips

### High Priority for Exam
- Creating Roles and ClusterRoles quickly with imperative commands
- Understanding the four combinations
- Using kubectl auth can-i for testing
- Troubleshooting permission issues
- Knowing when to use Role vs ClusterRole

### Common Exam Scenarios
- Create RBAC for a service account to manage specific resources
- Grant cross-namespace access
- Fix broken RBAC configurations
- Use default ClusterRoles (view, edit, admin)
- Test permissions for verification

### Time-Saving Tips
- Use imperative commands when possible
- Use --dry-run=client -o yaml to generate templates
- Bookmark the RBAC documentation page
- Practice the four combinations until they're second nature
- Always verify with kubectl auth can-i

### Common Mistakes to Avoid
- Forgetting namespace in serviceaccount reference: `namespace:sa-name`
- Using empty string `""` for core API group
- Confusing Role with ClusterRole scope
- Not testing both positive and negative cases

---

## Key Concepts Summary

**RBAC Components**:
- Role/ClusterRole define permissions
- RoleBinding/ClusterRoleBinding grant permissions to subjects
- Subjects are ServiceAccounts, Users, or Groups

**The Four Combinations**:
- Role + RoleBinding = namespace-scoped
- ClusterRole + RoleBinding = reusable, namespace-limited
- ClusterRole + ClusterRoleBinding = cluster-wide
- Role + ClusterRoleBinding = invalid

**Rules Structure**:
- API Groups organize resources
- Resources are what you access
- Verbs are actions you can perform
- ResourceNames provide fine-grained control

**Testing**:
- kubectl auth can-i checks permissions
- Always verify your RBAC configuration
- Test both positive and negative cases

**Best Practices**:
- Principle of least privilege
- Avoid wildcards when possible
- Use default ClusterRoles when appropriate
- Separate read and write permissions
- Document your RBAC design

---

**Remember**: RBAC is deny-by-default. Everything is forbidden unless you explicitly allow it through Roles and RoleBindings.