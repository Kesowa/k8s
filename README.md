# ARU — Kubernetes manifests

Kubernetes infrastructure-as-code for the **ARU** drone data platform, running
on AWS EKS. Plain manifests applied with `kubectl` — no Helm chart, no
Kustomize overlays.

The companion [terraform](https://github.com/Kesowa/terraform) repository
provisions the cluster itself; this repository is what runs on it.

> ### ⚠️ Secrets no longer live in this repository
>
> Credentials were previously committed to `main-config.yaml` in plaintext
> ConfigMaps, including a production database connection string and AWS access
> keys. **Those credentials have been invalidated and must not be reused.**
>
> Configuration is now split: non-secret values stay in ConfigMaps in
> `main-config.yaml`, and everything sensitive comes from Kubernetes Secrets
> described by `secrets.example.yaml`. See
> [Managing secrets](#managing-secrets) below, and read
> [SECURITY.md](SECURITY.md) before contributing.

## What is deployed

| Manifest | Component |
| --- | --- |
| `frontend.yaml`, `frontend-ingress.yaml` | [Aru-Client](https://github.com/Kesowa/Aru-Client) web UI |
| `backend.yaml`, `backend-ingress.yaml`, `backend-scale.yaml` | [Aru-Server](https://github.com/Kesowa/Aru-Server) API |
| `mongodb.yaml`, `mongo-claim.yaml` | in-cluster MongoDB and its volume claim |
| `minio.yaml`, `minio-claim.yaml`, `minio-ingress.yaml` | MinIO object storage |
| `rabbitmq.yaml` | message broker the workers consume from |
| `titiler.yaml`, `titiler-ingress.yaml`, `titiler-scale.yaml` | COG raster tile server |
| `decompress.yaml`, `decompress-scale.yaml` | zip decompression worker |
| `transcode.yaml`, `transcode-scale.yaml` | video transcoding worker |
| `report.yaml`, `report-scale.yaml` | report generation worker |
| `traefik*.yaml` | Traefik ingress controller, RBAC and dashboard |
| `certificate.yaml`, `cluster-issuer.yaml` | cert-manager TLS via Let's Encrypt |
| `main-config.yaml` | non-secret ConfigMaps |
| `secrets.example.yaml` | **template** for the required Secrets |
| `*.json` | AWS IAM policies, S3 bucket policy and CORS rules |

## Autoscaling

The four workers scale on queue depth with [KEDA](https://keda.sh), not on CPU.
Each `*-scale.yaml` is a `ScaledObject` watching a RabbitMQ queue with
`minReplicaCount: 0`, so a worker costs nothing when idle and spins up when
work arrives.

Broker credentials reach KEDA through a `TriggerAuthentication` that reads from
the `rabbitmq-secrets` Secret, rather than being written inline in each
`ScaledObject`.

## Managing secrets

`secrets.example.yaml` is a template of placeholders. The simplest path:

```bash
cp secrets.example.yaml secrets.yaml
$EDITOR secrets.yaml          # fill in real values
kubectl apply -f secrets.yaml
```

`secrets.yaml` is git-ignored. **Keeping a plaintext copy on a laptop is still
a weak spot, so prefer one of these instead:**

- **[External Secrets Operator](https://external-secrets.io/)** reading from AWS
  Secrets Manager. The natural fit here — the cluster is already on EKS with IAM
  available, and no secret material ever lands in git or on a developer machine.
- **[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)**, which
  encrypts a manifest so it *can* safely be committed. Only the cluster can
  decrypt it.
- **[SOPS](https://github.com/getsops/sops)** with AWS KMS — same idea.

### Two things to understand

**A Kubernetes Secret is not encrypted.** It is base64-encoded, which is
encoding, not encryption. Anyone with `get secrets` on the namespace can read
it, and so can anyone who reads the manifest. Secrets are an improvement over
ConfigMaps because they are access-controlled separately and excluded from
casual output — not because the value is protected. Enable
[encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
on etcd, and keep RBAC tight.

**Prefer IRSA over static AWS keys.** The `worker-secrets` and
`backend-secrets` Secrets still carry an access key and secret key, because
that is how the applications are currently written. On EKS you can bind a
ServiceAccount to an IAM role and have the SDK obtain short-lived credentials
automatically, which removes the long-lived key entirely. That is the single
biggest hardening win available here, and it needs a small change in each
application rather than in these manifests.

## Applying

Order matters, because later objects reference earlier ones:

```bash
# 1. Secrets and config first — pods will not start without them
kubectl apply -f secrets.yaml
kubectl apply -f main-config.yaml

# 2. Ingress controller and TLS
kubectl apply -f traefik-rbac.yaml -f traefik-perms.yaml
kubectl apply -f traefik-ds.yaml -f traefik-services.yaml
kubectl apply -f cluster-issuer.yaml -f certificate.yaml

# 3. Stateful dependencies, and their claims before them
kubectl apply -f mongo-claim.yaml -f mongodb.yaml
kubectl apply -f minio-claim.yaml -f minio.yaml
kubectl apply -f rabbitmq.yaml

# 4. Applications
kubectl apply -f backend.yaml -f frontend.yaml -f titiler.yaml
kubectl apply -f decompress.yaml -f transcode.yaml -f report.yaml

# 5. Ingresses and autoscalers
kubectl apply -f backend-ingress.yaml -f frontend-ingress.yaml \
              -f titiler-ingress.yaml -f minio-ingress.yaml
kubectl apply -f backend-scale.yaml -f titiler-scale.yaml \
              -f decompress-scale.yaml -f transcode-scale.yaml -f report-scale.yaml
```

The KEDA `ScaledObject`s require KEDA to be installed in the cluster
beforehand; it is not defined here.

## AWS policies

The JSON files are applied with the AWS CLI, not `kubectl`:

| File | Purpose |
| --- | --- |
| `iam_policy.json` | permissions for the AWS Load Balancer Controller |
| `eks-trust-policy.json` | trust policy for the EKS cluster role |
| `trust-policy.json` | trust policy for Fargate pod execution |
| `titiler-bucket-policy.json` | read access to the raster prefix for TiTiler |
| `cors-config.json` | S3 CORS rules for the browser client |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Security reports go through
[SECURITY.md](SECURITY.md) and private disclosure, never a public issue.

## License

Released under the [MIT License](LICENSE). © Kesowa Infinite Ventures.
