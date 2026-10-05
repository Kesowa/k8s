# Security Policy

## Reporting a vulnerability

**Please report privately. Do not open a public issue.**

- GitHub [private vulnerability reporting](https://github.com/Kesowa/k8s/security/advisories/new)
  on this repository, or
- email **security@kesowa.com**

We will acknowledge your report and tell you whether we intend to act on it.

**Do not use any credential you find in this repository or its history to
access any system, and do not test against Kesowa infrastructure.** Reading a
public repository is fine. Using what you find in it is not, and takes you
outside this policy. If you believe a credential is still live, tell us and we
will verify it ourselves.

## Credentials previously committed here

Production credentials were committed to `main-config.yaml` in plaintext
ConfigMaps and were publicly readable. This included a database connection
string, an AWS access key pair, mail and message-broker passwords, and an
application session signing secret.

**We know.** Those credentials are being rotated and have been removed from the
current manifests. Git history still contains them, and they are worthless once
rotated. You do not need to enumerate them for us.

What is worth reporting:

- **A credential you have reason to believe is still valid.** Say where you
  found it and why you think so — do not test it.
- **A credential somewhere we would not think to look**: a container image
  layer, a Helm value, a committed `kubeconfig`, an unmerged branch, a git
  object that is unreachable but still fetchable.

## In scope

- **Credentials still present** in the current manifests, in any form.
- **Over-permissive IAM**, in `iam_policy.json`, either trust policy, or the
  bucket policy. A wildcard that grants more than the component needs is a
  finding — the load balancer controller policy is broad by upstream design, so
  tell us if you think we have widened it further.
- **S3 bucket policy or CORS misconfiguration** in `titiler-bucket-policy.json`
  or `cors-config.json` that would expose object data or allow an unintended
  origin to read it.
- **Over-permissive Kubernetes RBAC**, particularly in the Traefik manifests.
  A ClusterRole granting secret access it does not need is a finding.
- **Container security context issues** — running as root, a privileged
  container, an unnecessary hostPath mount, a missing `readOnlyRootFilesystem`
  where it would apply.
- **Network exposure.** An Ingress, Service or dashboard reachable from the
  internet that should not be. The Traefik dashboard is a specific concern.
- **Missing NetworkPolicy** consequences: a demonstrated path from a
  lower-trust pod to MongoDB, MinIO or RabbitMQ.
- **Pinning and supply chain** — a mutable image tag that would let an upstream
  change alter what runs.
- **Secrets reaching logs** through a container's startup output or an
  environment dump.

## Out of scope

- The historical credentials described above.
- Internal cluster DNS names (`rabbitmq.default.svc.cluster.local`), container
  names and in-cluster ports. These are not reachable from outside.
- Public hostnames such as `aru.kesowa.com`. They are meant to be public;
  report a vulnerability *at* one of them against the relevant application
  repository.
- Placeholder values in `secrets.example.yaml`. `REPLACE_ME` is not a
  credential.
- The fact that a Kubernetes Secret is base64-encoded rather than encrypted.
  Known, documented in the README, and inherent to Kubernetes.
- Findings in Kubernetes, Traefik, KEDA, cert-manager, MinIO or RabbitMQ
  themselves. Report those upstream, though we appreciate a heads-up.
- Automated scanner output with no demonstrated impact on this configuration.

## Hardening we know about and want

Stated plainly so you do not spend time telling us what we already know:

- **Static AWS access keys should be replaced by IRSA.** Highest-value change
  available; needs an application change as well as a manifest change.
- **NetworkPolicies are absent entirely.** Every pod can reach every other.
- **Encryption at rest for etcd** should be confirmed enabled on the cluster.
- **There is no environment separation.** One set of manifests, and it is
  production.

A concrete exploit path through any of these is still worth reporting — knowing
a weakness exists is not the same as knowing what it lets someone do.
