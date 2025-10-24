# Service Accounts - Complete Reference Guide
## CKA Certification Study Notes

---

## What is a Service Account?

**Definition**: A Service Account is a Kubernetes identity for pods (and other non-human entities) that allows them to authenticate to the Kubernetes API server.

**Simple Version**: A Service Account is a username for pods.

### Key Characteristics
- **Identity, Not Permissions**: Answers "WHO are you?" not "WHAT can you do?"
- **For Pods, Not People**: Service Accounts are for applications; User Accounts are for humans
- **Namespaced Resource**: Belongs to a specific namespace
- **Authentication Mechanism**: Provides JWT tokens for API server authentication

---

## When to Create a Service Account

### Create a Separate Service Account When:

1. **CI/CD Pipelines / Deployment Tools**
   - Example: Jenkins, ArgoCD, Flux running in-cluster
   - Need to create/update/delete deployments, services, configmaps

2. **Monitoring & Observability Tools**
   - Example: Prometheus, Grafana, custom monitoring apps
   - Need to read pod metrics, node stats, service endpoints across namespaces

3. **Custom Controllers / Operators**
   - Watch for specific resources (databases, certificates)
   - Create/modify other resources in response

4. **Secrets Management**
   - App needs to read specific secrets but nothing else
   - Principle of least privilege

5. **Multi-Tenant Environments**
   - Different teams sharing a cluster
   - Each team gets dedicated service accounts with specific permissions

### Don't Create One When:
- Your app doesn't interact with K8s API at all (most web apps)
- The default SA's permissions are sufficient (rare)

---

## Service Account Tokens

### What is a Service Account Token?
A **JWT (JSON Web Token)** that proves a pod's identity to the Kubernetes API server.

### Token Evolution

**Old Way (Before K8s 1.24)** - Legacy Tokens:
- Long-lived tokens stored in Secrets
- Never expired (security risk!)
- Valid forever if compromised

**New Way (K8s 1.24+)** - Bound Service Account Tokens:
- Dynamically generated when pod starts
- Time-bound (default: 1 hour, auto-refreshed)
- Pod-bound (can't be used elsewhere)
- More secure!

### Where Tokens Live

Automatically mounted at:
```
/var/run/secrets/kubernetes.io/serviceaccount/
├── token          # The JWT token
├── ca.crt         # Cluster CA certificate
└── namespace      # The pod's namespace
```

### Token Contents (JWT Payload)
```json
{
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1728123456,
  "iat": 1728119856,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "test-pod",
      "uid": "abc-123-def"
    },
    "serviceaccount": {
      "name": "my-sa",
      "uid": "xyz-789"
    }
  },
  "sub": "system:serviceaccount:default:my-sa"
}
```

Key fields:
- `sub`: Identity format `system:serviceaccount:<namespace>:<sa-name>`
- `exp`: Expiration timestamp
- `kubernetes.io.pod`: Binds token to specific pod

---

## Authentication vs Authorization

### Critical Distinction

**Service Account WITHOUT RBAC**:
- ✅ Authentication works: Can prove identity to API server
- ❌ Authorization fails: Cannot do anything useful
- Has ZERO permissions (except basic cluster discovery)

### The Flow
```
Service Account (SA)
    ↓
Provides Token (Authentication)
    ↓
API Server validates: "I know who you are"
    ↓
RBAC checks: "What are you allowed to do?"
    ↓
Allow/Deny based on Roles & RoleBindings
```

### Testing Permissions
```bash
# Check what a SA can do
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa

# List all permissions
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

---

## Working with Service Accounts

### Creating Service Accounts
```bash
# Imperative
kubectl create sa my-app-sa -n production

# Declarative
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
```

### Using in Pods
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-app-sa  # Specify SA here
  containers:
  - name: nginx
    image: nginx
```

### Disabling Token Mounting
```yaml
# At SA level
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-sa
automountServiceAccountToken: false

# At Pod level
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: no-api-sa
  automountServiceAccountToken: false
  containers:
  - name: nginx
    image: nginx
```

**When to disable**: If app doesn't need K8s API access (security best practice)

---

## Default Service Account

Every namespace has a `default` service account:
- Automatically created
- Assigned to pods if no SA specified
- Has NO meaningful permissions
- Common misconception: "default" doesn't mean it has default permissions!

```bash
# Even default SA has nothing
kubectl auth can-i list pods --as=system:serviceaccount:default:default
# Output: no
```

---

## RBAC Integration

Service Accounts need RBAC to be useful:

### Complete Example
```yaml
# 1. Service Account (Identity)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: default
---
# 2. Role (Permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# 3. RoleBinding (Links SA to Permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Common Patterns

### 1. Read-Only Monitoring
```bash
kubectl create sa monitor-sa
kubectl create clusterrole monitor-role --verb=get,list --resource=pods,nodes
kubectl create clusterrolebinding monitor-binding \
  --clusterrole=monitor-role \
  --serviceaccount=default:monitor-sa
```

### 2. Deployment Manager
```bash
kubectl create sa deployer-sa -n prod
kubectl create role deployer-role \
  --verb=create,update,delete,get,list \
  --resource=deployments,replicasets \
  -n prod
kubectl create rolebinding deployer-binding \
  --role=deployer-role \
  --serviceaccount=prod:deployer-sa \
  -n prod
```

### 3. Cross-Namespace Access
```bash
# SA in namespace A accessing resources in namespace B
kubectl create sa monitor-sa -n monitoring
kubectl create role pod-reader --verb=get,list --resource=pods -n production
kubectl create rolebinding cross-ns \
  --role=pod-reader \
  --serviceaccount=monitoring:monitor-sa \
  -n production
```

---

## Key Commands Reference

```bash
# Create SA
kubectl create sa <name> [-n <namespace>]

# Get SA details
kubectl get sa <name> -n <namespace> -o yaml

# Check which SA a pod uses
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'

# Test SA permissions
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<namespace>:<sa>

# Manually create a token (testing)
kubectl create token <sa-name> --duration=1h

# View token from inside pod
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

---

## CKA Exam Tips

### High Priority Topics
1. Creating service accounts
2. Assigning SAs to pods
3. Understanding SA has no permissions without RBAC
4. Testing permissions with `kubectl auth can-i`
5. Troubleshooting: "Why can't my pod access the API?"

### Common Exam Scenarios
- Create SA with specific RBAC permissions
- Fix permission issues for existing pods
- Configure cross-namespace access
- Disable token mounting for security

### Quick Checks
```bash
# Is pod using correct SA?
kubectl get pod <name> -o jsonpath='{.spec.serviceAccountName}'

# Does SA have permissions?
kubectl auth can-i <action> --as=system:serviceaccount:<ns>:<sa>

# Is token mounted?
kubectl exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

---

## Analogies for Understanding

### Service Account = Employee Badge
- **SA**: Your employee ID badge (proves who you are)
- **RBAC Role**: Your job description (what you're allowed to do)
- **RoleBinding**: Your employment contract (links your ID to your job)

Without a job (RBAC), your badge only gets you in the lobby!

### Service Account = Bank Account
- **SA**: Bank account number (identity)
- **RBAC**: Money in the account (permissions)
- **Token**: ATM card (authentication mechanism)

Account exists but with $0 balance = SA without RBAC!

---

## Security Best Practices

1. **Principle of Least Privilege**: Grant only necessary permissions
2. **Disable Token Mounting**: If app doesn't need K8s API access
3. **Use Separate SAs**: Different apps should have different SAs
4. **Avoid Wildcards**: Don't use `verb: ["*"]` unless necessary
5. **Regular Audits**: Review SA permissions periodically
6. **Namespace Isolation**: Use namespaces + RBAC for multi-tenancy

---

## Summary

**Service Account**:
- Kubernetes identity for pods
- Provides JWT tokens for authentication
- Namespaced resource
- Has ZERO permissions without RBAC

**Authentication Flow**:
1. Pod uses SA token to authenticate
2. API server validates token (knows WHO)
3. RBAC checks permissions (knows WHAT)
4. Allow or deny based on Roles/ClusterRoles

**Key Principle**: 
Authentication (SA) ≠ Authorization (RBAC)  
You need BOTH for a pod to interact with the K8s API.

---

**Remember**: A Service Account is just an identity. It's RBAC that gives it power!