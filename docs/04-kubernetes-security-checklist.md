# Kubernetes Security Checklist

This checklist aims at providing a basic list of guidance with links to more comprehensive documentation on each topic. It does not claim to be exhaustive and is meant to evolve.

On how to read and use this document:

- The order of topics does not reflect an order of priority.
- Some checklist items are detailed in the paragraph below the list of each section.

> **Caution:** Checklists are **not** sufficient for attaining a good security posture on their own. A good security posture requires constant attention and improvement, but a checklist can be the first step on the never-ending journey towards security preparedness. Some of the recommendations in this checklist may be too restrictive or too lax for your specific security needs. Since Kubernetes security is not "one size fits all", each category of checklist items should be evaluated on its merits.

## Authentication & Authorization

- [ ] `system:masters` group is not used for user or component authentication after bootstrapping.
- [ ] The kube-controller-manager is running with `--use-service-account-credentials` enabled.
- [ ] The root certificate is protected (either an offline CA, or a managed online CA with effective access controls).
- [ ] Intermediate and leaf certificates have an expiry date no more than 3 years in the future.
- [ ] A process exists for periodic access review, and reviews occur no more than 24 months apart.
- [ ] The Role Based Access Control Good Practices are followed for guidance related to authentication and authorization.

After bootstrapping, neither users nor components should authenticate to the Kubernetes API as `system:masters`. Similarly, running all of kube-controller-manager as `system:masters` should be avoided. In fact, `system:masters` should only be used as a break-glass mechanism, as opposed to an admin user.

## Network security

- [ ] CNI plugins in use support network policies.
- [ ] Ingress and egress network policies are applied to all workloads in the cluster.
- [ ] Default network policies within each namespace, selecting all pods, denying everything, are in place.
- [ ] If appropriate, a service mesh is used to encrypt all communications inside of the cluster.
- [ ] The Kubernetes API, kubelet API and etcd are not exposed publicly on Internet.
- [ ] Access from the workloads to the cloud metadata API is filtered.
- [ ] Use of LoadBalancer and ExternalIPs is restricted.
