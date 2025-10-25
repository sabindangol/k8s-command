# Security Contexts - Complete Reference Guide
## CKA Certification Study Notes

---

## What is a Security Context?

**Definition**: A Security Context is a set of security-related settings that define the privileges and access control configurations for a pod or container. It controls what a process inside a container can do at the operating system level - things like which user it runs as, what file permissions it has, and what special Linux capabilities it possesses.

**Simple Version**: Service Accounts tell Kubernetes "who you are" for API access, while Security Contexts tell Linux "who you are" and "what you can do" at the operating system level inside the container.

---

## The Analogy: Hotel Room Access 🏨

Think of Security Contexts like the rules and restrictions for your hotel room:

**RunAsUser** (which user ID you run as) is like your guest profile. Are you a regular guest, a VIP, or hotel staff? Each has different privileges in the building.

**RunAsNonRoot** is like a hotel policy that says "no maintenance workers can use guest rooms" - it prevents privileged system accounts from running your application.

**ReadOnlyRootFilesystem** is like having a room where you can see and use everything, but you can't rearrange furniture or damage anything. You can only write to specific designated areas (like a safe or a notepad).

**Privileged mode** is like giving someone a master key that opens every door in the hotel, including restricted areas. This is dangerous and should almost never be used.

**Capabilities** are like specific permissions - maybe you can access the pool, use the gym, or enter the business center. You grant only the specific capabilities needed, not a master key.

---

## Two Levels of Security Context

Security Contexts can be defined at two levels:

### Pod-Level Security Context

Applies to all containers in the pod. Think of it as building-wide rules.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  securityContext:          # Pod-level
    runAsUser: 1000         # All containers run as user 1000
    runAsGroup: 3000        # All containers run as group 3000
    fsGroup: 2000           # Filesystem group for volumes
  containers:
  - name: demo
    image: nginx
```

### Container-Level Security Context

Applies to a specific container and overrides pod-level settings.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  securityContext:
    runAsUser: 1000         # Pod default
  containers:
  - name: demo
    image: nginx
    securityContext:
      runAsUser: 2000       # This container overrides pod setting
      allowPrivilegeEscalation: false
```

**The hierarchy**: Container-level settings override pod-level settings. If both are specified, the container wins.

---

## Core Security Context Settings

### runAsUser and runAsGroup

Specify which user and group ID the process runs as.

```yaml
securityContext:
  runAsUser: 1000      # Process runs as user ID 1000
  runAsGroup: 3000     # Process runs as group ID 3000
```

**When to use**: Always set these to non-root values (anything above 0) for production applications. User 1000 is commonly used for application users.

**What it does**: Changes the effective user ID of processes inside the container. If someone compromises your application, they only have the privileges of this user ID, not root privileges.

### runAsNonRoot

Boolean safety check that prevents running as root.

```yaml
securityContext:
  runAsNonRoot: true   # Container must NOT run as root (UID 0)
```

**When to use**: Set to true for all production workloads. If you accidentally deploy a container that tries to run as root, Kubernetes will refuse to start it.

**What it does**: Acts as a validation check. Kubernetes inspects the container's USER directive and runAsUser setting and rejects the pod if it would run as UID 0.

### fsGroup

Sets the group ownership for mounted volumes.

```yaml
securityContext:
  fsGroup: 2000        # Volumes are owned by group 2000
```

**When to use**: When your pod uses persistent volumes and containers need to access files. All files in mounted volumes will be owned by this group ID.

**What it does**: When a volume is mounted, Kubernetes changes the ownership of all files to this group ID, allowing your non-root processes to read and write them.

### readOnlyRootFilesystem

Makes the container's root filesystem read-only.

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**When to use**: For stateless applications that don't need to modify their filesystem. This is a strong security practice.

**What it does**: Prevents any writes to the container's filesystem except for mounted volumes. Even if an attacker compromises the container, they cannot modify system binaries or configuration files.

### allowPrivilegeEscalation

Controls whether a process can gain more privileges than its parent.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

**When to use**: Set to false for almost all applications. This prevents processes from using setuid binaries or other mechanisms to escalate privileges.

**What it does**: Sets the no_new_privs flag on the process, preventing it from gaining additional privileges through setuid, setgid, or file capabilities.

### privileged

Runs the container in privileged mode with access to all host devices.

```yaml
securityContext:
  privileged: true     # DANGEROUS!
```

**When to use**: Almost never! Only for very specific infrastructure components like CNI plugins or storage drivers that need host-level access.

**What it does**: Disables all isolation and gives the container the same access to the host as a process running directly on the host. This is essentially running with root on the host machine.

### supplementalGroups

Additional group IDs that the process belongs to.

```yaml
securityContext:
  supplementalGroups: [4000, 5000]  # Additional group memberships
```

**When to use**: When you need access to files owned by specific groups beyond your primary group.

**What it does**: Adds the process to additional Unix groups, similar to how Unix users can belong to multiple groups for accessing different resources.

---

## Basic Examples

### Example 1: Running as a Non-Root User

By default, many container images run as root (user ID 0), which is a security risk.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root-demo
spec:
  securityContext:
    runAsUser: 1000         # Run as user ID 1000 (not root)
    runAsNonRoot: true      # Enforce that it's not root
  containers:
  - name: nginx
    image: nginx
```

**What happens**: The process inside the container runs as user 1000 instead of root. The runAsNonRoot setting acts as a safety check - if the image tries to run as root, Kubernetes will refuse to start the container.

### Example 2: Read-Only Root Filesystem

Most applications don't need to write to their root filesystem.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-demo
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true    # Can't write to /
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx     # But CAN write here
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}                       # Temporary writable storage
  - name: run
    emptyDir: {}
```

**What happens**: The container can read all files in its filesystem, but cannot write to them. Applications often need some writable directories (like cache or temp directories), so we mount emptyDir volumes for those specific paths.

### Example 3: Dropping Capabilities

Linux capabilities are fine-grained privileges that traditionally belonged to root.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: drop-caps-demo
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL                # Drop all capabilities
        add:
        - NET_BIND_SERVICE   # Add back only what we need
```

**What happens**: By default, containers get many capabilities. We drop them all and add back only NET_BIND_SERVICE, which allows binding to privileged ports (below 1024). This follows the principle of least privilege.

---

## Advanced Production Examples

### Example 4: Complete Production-Ready Security

A well-secured pod in production looks like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**What makes this secure**: This pod runs as a non-root user, has a read-only filesystem except for specific writable directories, drops all Linux capabilities, and prevents privilege escalation. This is the gold standard for security.

### Example 5: Multi-Container Pod with Different Security Contexts

Different containers in the same pod can have different privileges:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-security
spec:
  securityContext:
    runAsUser: 1000       # Default for all containers
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsUser: 1000     # Uses pod default
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
  - name: sidecar
    image: logging-agent:1.0
    securityContext:
      runAsUser: 2000     # Different user!
      capabilities:
        drop:
        - ALL
        add:
        - DAC_READ_SEARCH  # Needs to read app logs
```

**What's happening**: The app container runs as user 1000 with very restricted privileges, while the logging sidecar runs as user 2000 and has the DAC_READ_SEARCH capability to read files that might belong to other users.

### Example 6: Volume Access with fsGroup

When containers need to access mounted volumes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-access
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000        # Makes mounted volumes accessible
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

**What's happening**: The fsGroup setting ensures that files in the mounted volume are owned by group 2000. Since the process runs as user 1000 (which is implicitly added to group 2000), it can access the files.

---

## Understanding Linux Capabilities

Capabilities break root's powers into smaller, granular privileges. Traditional Unix has two levels: root (everything) and users (limited). Capabilities provide fine-grained control.

### Common Capabilities for CKA

**CAP_NET_ADMIN**: Configure network interfaces, routing tables, firewall rules. Needed by network tools and CNI plugins.

**CAP_NET_BIND_SERVICE**: Bind to privileged ports (below 1024). Needed if your app serves on port 80 or 443.

**CAP_SYS_TIME**: Set the system clock. Needed by NTP daemons.

**CAP_SYS_ADMIN**: A catch-all for many system administration tasks. Very powerful - try to avoid it.

**CAP_CHOWN**: Change file ownership. Needed if your application creates files and needs to set their ownership.

**CAP_DAC_OVERRIDE**: Bypass file permission checks. Dangerous - allows reading any file.

**CAP_DAC_READ_SEARCH**: Bypass read and directory search permission checks. Useful for logging agents.

**CAP_SETUID and CAP_SETGID**: Change user and group IDs. Needed by login programs.

### Best Practice Pattern

Always use this pattern for capabilities:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL              # Start with nothing
    add:
    - NET_BIND_SERVICE # Add only what you need
```

Start by dropping everything, then add back only the specific capabilities your application requires. This is the recommended security approach.

---

## Common Exam Scenarios

### Scenario 1: Secure an Insecure Pod

Given an insecure pod running as root:

```yaml
# Before (insecure)
apiVersion: v1
kind: Pod
metadata:
  name: insecure-app
spec:
  containers:
  - name: nginx
    image: nginx
```

Make it secure:

```yaml
# After (secure)
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: cache
    emptyDir: {}
```

### Scenario 2: Fix Volume Permission Issues

A pod can't write to its mounted volume because it's running as user 1000, but the volume is owned by user 2000. The fix is using fsGroup:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-fix
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000        # This makes mounted volumes accessible!
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### Scenario 3: Enforce Non-Root Across Namespace

Create a pod that absolutely cannot run as root and will fail if it tries:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: enforced-nonroot
spec:
  securityContext:
    runAsNonRoot: true    # Kubernetes validates this
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
```

If the image's USER directive is set to root or runAsUser is set to 0, Kubernetes will reject the pod with an error.

---

## Verification Commands for Exam

Quick commands to verify security settings during the exam:

```bash
# Check what user a container is running as
kubectl exec pod-name -- id
# Output shows: uid=1000 gid=3000 groups=2000,3000

# Check if filesystem is read-only
kubectl exec pod-name -- touch /test-file
# If it fails with "Read-only file system", it's working correctly

# View security context of a running pod
kubectl get pod pod-name -o yaml | grep -A 10 securityContext

# Describe pod to see security settings
kubectl describe pod pod-name

# Check capabilities (requires nsenter or similar)
kubectl exec pod-name -- cat /proc/1/status | grep Cap
```

---

## Security Best Practices

### Always Run as Non-Root

Set runAsUser to something other than 0 and set runAsNonRoot true as a safety check.

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
```

### Use Read-Only Root Filesystems

Set readOnlyRootFilesystem true and mount emptyDir volumes for writable directories.

```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
- name: tmp
  mountPath: /tmp
volumes:
- name: tmp
  emptyDir: {}
```

### Drop All Capabilities

Use the pattern of dropping ALL and adding back only what's needed.

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
```

### Prevent Privilege Escalation

Always set allowPrivilegeEscalation to false unless you have a specific reason not to.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

### Use fsGroup for Volume Access

When containers need to access mounted volumes, use fsGroup to set group ownership.

```yaml
securityContext:
  fsGroup: 2000
```

### Avoid Privileged Mode

Never use privileged true unless absolutely required for infrastructure components.

---

## Key Concepts Summary

**Two Levels**: Pod-level security context applies to all containers, but container-level can override it.

**User and Group IDs**: runAsUser and runAsGroup control which user the process runs as. Always use non-root values (above 0).

**Filesystem Security**: readOnlyRootFilesystem prevents modifications to the container's filesystem. Use emptyDir volumes for directories that need writes.

**Capabilities**: Fine-grained privileges. Drop ALL and add back only what's needed.

**Volume Access**: fsGroup sets group ownership on mounted volumes, allowing non-root processes to access them.

**Validation**: runAsNonRoot acts as a safety check, rejecting pods that would run as root.

**Privilege Escalation**: allowPrivilegeEscalation false prevents processes from gaining additional privileges.

**Privileged Mode**: privileged true gives full host access. Avoid unless absolutely necessary.

---

## CKA Exam Tips

**High Priority Topics**: Running as non-root, read-only filesystems, dropping capabilities, using fsGroup for volume access.

**Common Tasks**: Securing an insecure pod, fixing volume permission issues, understanding capability requirements.

**Quick Wins**: Know the imperative commands, understand the two-level hierarchy, practice verifying security settings.

**Time Savers**: Use kubectl run with --dry-run=client -o yaml to generate pod YAML, then add security contexts. Know where to find examples in the Kubernetes documentation.

**Common Mistakes**: Forgetting to mount writable volumes when using readOnlyRootFilesystem, not understanding that fsGroup only applies to volumes, confusing pod-level and container-level settings.

---

**Remember**: Security Contexts control operating system-level security, while RBAC controls Kubernetes API access. Together, they provide defense in depth for your applications.