# Contributing

Kubernetes manifests for a production platform. A mistake here takes a live
service down, so the bar is a little higher than for application code.

## The rule that matters most

**Never commit a credential.**

This repository previously held a production database connection string, an AWS
access key pair, SMTP and broker passwords, all in plaintext ConfigMaps in a
public repository. It is the most serious incident in this codebase's history.

Concretely:

- **Secrets go in a `Secret`, never a `ConfigMap`.** A ConfigMap is plaintext
  configuration and is treated as non-sensitive by every tool that touches it.
- **`secrets.yaml` is git-ignored. Keep it that way.** Document new keys in
  `secrets.example.yaml` with a placeholder value and a comment saying what the
  key is for.
- **Base64 is not encryption.** `data:` instead of `stringData:` does not make a
  committed secret safe. If you want an encrypted manifest in git, use Sealed
  Secrets or SOPS.
- **Run `git diff --cached` before every commit.** Look at it.
- If you commit a credential, **say so immediately**. It needs rotating, and
  deleting it in a later commit does not remove it from history. Reporting fast
  is the thing that limits the damage.

## Before you push

Validate, then dry-run against the cluster:

```bash
# parses and checks against the API schema, without contacting the cluster
kubectl apply --dry-run=client -f <file>

# validates against the live API, including admission webhooks
kubectl apply --dry-run=server -f <file>

# optional, catches more
kubeconform -strict <file>
```

Then check your diff for secrets, and confirm you have not changed a
`replicas` or resource limit by accident.

## Style

- One component per file, named after the component.
- Keep the existing two-space YAML indentation and `---` document separators.
- Pin container images to a tag or digest, never `:latest`. A mutable tag makes
  a rollback meaningless.
- Set resource requests and limits on new workloads. Without them a pod can
  starve its neighbours.
- Add a readiness probe to anything serving traffic, and a liveness probe where
  a hang is plausible.
- Reference configuration with `envFrom` and a `configMapRef` plus a
  `secretRef`, as the existing deployments do, rather than listing variables
  individually.

## Changing configuration

Non-secret values go in the relevant ConfigMap in `main-config.yaml`. Secret
values go in `secrets.example.yaml` as a placeholder, and in your real
`secrets.yaml` with the value.

A ConfigMap or Secret change does **not** restart pods on its own. Either
restart the deployment explicitly:

```bash
kubectl rollout restart deployment/<name>
```

or add a checksum annotation to the pod template so a config change rolls the
pods automatically.

## Autoscaling changes

Worker scaling is KEDA on queue depth, not CPU. When editing a `*-scale.yaml`:

- `minReplicaCount: 0` is deliberate — idle workers should cost nothing. Think
  hard before raising it, and expect a cold-start delay on the first job.
- Get broker credentials from the `rabbitmq-trigger-auth`
  `TriggerAuthentication`. Do not write an `amqp://user:pass@...` URI inline in
  a trigger; that is exactly how the credentials leaked before.
- Raising `maxReplicaCount` has a cost, and for S3-bound workers it can also
  hit API rate limits. Check both.

## Pull requests

1. Open an issue first for anything that changes topology, storage or scaling.
2. One concern per pull request.
3. Say what you ran — the dry-run output, and whether you tried it on a
   non-production cluster.
4. For anything affecting a stateful component (`mongodb.yaml`, `minio.yaml`,
   or either claim), explain explicitly what happens to existing data. A
   mistaken `PersistentVolumeClaim` change can destroy it.
5. Note whether the change needs a coordinated deploy with Aru-Server or
   Aru-Client.

## Worth fixing

- **Static AWS keys.** `backend-secrets` and `worker-secrets` still carry an
  access key pair. Moving to IRSA — a ServiceAccount bound to an IAM role —
  would remove long-lived AWS credentials from the cluster entirely. This is the
  highest-value change available, and needs a small application change too.
- **No NetworkPolicies.** Any pod can reach any other. MongoDB, MinIO and
  RabbitMQ should only be reachable from the workloads that need them.
- **`MODE: production` in a repository with no environment separation.** There
  is one set of manifests and it is production. Staging has nowhere to live.
  Kustomize overlays, or a directory per environment, would fix this.
- **MongoDB runs in-cluster with a PVC**, while `main-config.yaml` points the
  backend at Atlas. One of those is dead weight; it is unclear which, and it
  should be resolved rather than left ambiguous.
- **No resource limits** on several workloads.
- **KEDA is a hard dependency** that is not declared anywhere in this
  repository.

## License

Contributions are accepted under the [MIT License](LICENSE) covering this
repository.
