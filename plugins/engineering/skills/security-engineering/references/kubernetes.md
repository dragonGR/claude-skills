# Kubernetes security

Read this when reviewing workload manifests, RBAC, network policy, admission policy, secret delivery or workload identity. Review the rendered manifests (after Helm or Kustomize) against the target cluster's CNI, admission stack and identity provider; YAML that looks restrictive can do nothing at runtime.

## Failures that ship

**Privileged or host-coupled pods.** `privileged: true`, `hostNetwork`, `hostPID`, `hostIPC`, `hostPath` mounts (especially `/`, `/var/run/docker.sock`, `/var/lib/kubelet`, `/etc`), added capabilities such as `SYS_ADMIN` or `NET_ADMIN`. Any of these usually turns a compromised container into a compromised node. Each needs a written reason and an admission exception scoped to that workload.

**Default security context.** No `runAsNonRoot`, `allowPrivilegeEscalation` left true, capabilities not dropped, writable root filesystem, no seccomp profile. For application pods set:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

Then check the image actually runs as a non-root user; `runAsNonRoot` makes a root image fail to start, which is the point, but it surprises people during rollout. Enforce with Pod Security Admission `restricted` on application namespaces, using the `warn` and `audit` modes first to find what would break.

**Service account tokens nobody needs.** Every pod gets the namespace's `default` service account token mounted unless told otherwise. If the workload does not call the Kubernetes API, set `automountServiceAccountToken: false`. If it does, give it a dedicated service account.

**RBAC wider than it looks.** Wildcards in `verbs` or `resources`; `get`/`list` on `secrets` in a namespace (reads every secret there, including other workloads' credentials); `create` on `pods` or `pods/exec` (run anything as any service account in the namespace, which inherits its secrets); `escalate`, `bind` or `impersonate`; a `ClusterRole` meant for one namespace granted in every namespace through a `ClusterRoleBinding` instead of a namespaced `RoleBinding`; aggregated ClusterRoles picking up new rules. Verify effective permissions, not the YAML you wrote:

```bash
kubectl auth can-i --list --as=system:serviceaccount:payments:api -n payments
```

**Network policy that does not enforce.** NetworkPolicy objects are accepted by the API even when the CNI does not implement them. Confirm the CNI enforces policy, then start each namespace from default deny for both directions:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Add explicit allows for DNS, ingress to the web tier, app to database, monitoring scrape and named external destinations. Test both an allowed and a denied connection from a real pod after applying. Egress to the cloud metadata address (`169.254.169.254`, and on AWS the IPv6 IMDS address) should be denied for workloads that use workload identity instead of node credentials.

**Node credentials reachable from pods.** On EKS, a pod that can reach IMDS can obtain the node role's credentials. Require IMDSv2 with a hop limit of 1 on nodes where pods should not reach it, and use workload identity (IRSA or EKS Pod Identity, GKE Workload Identity, Azure Workload Identity) for pods that need cloud access.

**Secrets delivered too broadly.** Kubernetes Secrets are base64, not encrypted, unless encryption at rest is configured on the API server. Anyone who can read Secrets in a namespace or create pods there can read them. If an external-secrets controller syncs from a cloud secret manager, review the controller's own permissions (often cluster-wide), which namespaces it writes to, and which workloads can read the result.

**Mutable images and unverified sources.** `:latest` or other mutable tags, images from public registries without review, no signature verification. Pin by digest in production manifests and enforce allowed registries at admission.

## Admission policy

Use the policy engine already in the cluster (Pod Security Admission, Gatekeeper, Kyverno, or ValidatingAdmissionPolicy) rather than adding a second. Rules worth enforcing: no privileged or host-namespace pods outside named system namespaces, no `hostPath` except allowlisted paths, images from approved registries by digest, resource limits present, `automountServiceAccountToken` explicit. Test that a violating manifest is rejected and that the documented exception path works.

## Service mesh

Nominal mTLS can be undone by a permissive peer authentication mode, namespaces excluded from injection, pods annotated to skip the sidecar, or egress that bypasses the mesh. Authorization policies must reference workload identities, not IPs. Test a call that should be denied and confirm it is.

## Release evidence

- Rendered manifests pass the cluster's admission policy, and a deliberately bad manifest is rejected.
- `kubectl auth can-i --list` output for each workload identity matches what the workload needs.
- Allowed and denied network paths were tested from a pod, including metadata endpoint egress.
- Images are pinned by digest from approved registries.
- Secret delivery, encryption at rest and who can read each Secret are documented.
