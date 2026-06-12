# Kubernetes Security Checklist

This checklist provides basic guidance with links to fuller documentation on each topic. It is not exhaustive and is meant to evolve.

How to read this document:

- Topic order does not reflect priority.
- Some items are detailed in the paragraph below each section's list.

> **Caution:** Checklists are **not** sufficient for a good security posture on their own. Good security requires constant attention and improvement; a checklist is only the first step. Some recommendations here may be too restrictive or too lax for your specific security needs. Since Kubernetes security is not "one size fits all", each category should be evaluated on its merits.

## Authentication & Authorization

- [ ] `system:masters` group is not used for user or component authentication after bootstrapping.
- [ ] The kube-controller-manager is running with `--use-service-account-credentials` enabled.
- [ ] The root certificate is protected (either an offline CA, or a managed online CA with effective access controls).
- [ ] Intermediate and leaf certificates have an expiry date no more than 3 years in the future.
- [ ] A process exists for periodic access review, and reviews occur no more than 24 months apart.
- [ ] The Role Based Access Control Good Practices are followed for guidance related to authentication and authorization.

After bootstrapping, neither users nor components should authenticate to the Kubernetes API as `system:masters`. Avoid running kube-controller-manager as `system:masters`. It should only be a break-glass mechanism, not an admin user.

## Network security

- [ ] CNI plugins in use support network policies.
- [ ] Ingress and egress network policies are applied to all workloads in the cluster.
- [ ] Default network policies within each namespace, selecting all pods, denying everything, are in place.
- [ ] If appropriate, a service mesh is used to encrypt all communications inside of the cluster.
- [ ] The Kubernetes API, kubelet API and etcd are not exposed publicly on Internet.
- [ ] Access from the workloads to the cloud metadata API is filtered.
- [ ] Use of LoadBalancer and ExternalIPs is restricted.
