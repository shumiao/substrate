# Sandbox runtimes

Substrate runs every actor in a sandbox. It supports two sandbox classes, and
one cluster can run both:

- **gVisor** (`gvisor`) runs each actor under [gVisor](https://gvisor.dev/), an
  application kernel that implements Linux system calls in user space. It runs
  on any Linux node.
- **micro-VM** (`microvm`) runs each actor in its own lightweight virtual
  machine: a [Kata Containers](https://katacontainers.io/) guest with its own
  Linux kernel, on the [Cloud Hypervisor](https://www.cloudhypervisor.org/)
  VMM. It runs on nodes that expose `/dev/kvm`.

Actors use the same API on both classes: templates, suspend and resume at
every snapshot fidelity, actor identity, networking, egress policy, and
durable volumes. A `WorkerPool` sets the class, and an `ActorTemplate` picks
pools with its `workerSelector`.

## Choosing a class

| | gVisor | micro-VM |
|---|---|---|
| Isolation boundary | gVisor application kernel | Hardware virtualization (KVM), one guest kernel per actor |
| Linux compatibility | The system calls gVisor implements | A full Linux kernel |
| Per-container CPU and memory limits | Applied to the whole sandbox | Enforced per container ([details](api-guide.md#per-container-limits)) |
| Node requirement | Any Linux node | `/dev/kvm`: bare metal, or a VM with nested virtualization |
| Installed by | `ate-setup deploy ate-system` | `hack/install-microvm-deps.sh --install` |

Pick micro-VM for workloads that need a full Linux kernel or a hardware
virtualization boundary. Pick gVisor to run on nodes without `/dev/kvm`.

## Node requirements

gVisor needs nothing beyond a Linux node.

A micro-VM worker pod requests the `ate.dev/kvm` device. atelet advertises the
device only on nodes that have `/dev/kvm`, so Kubernetes places micro-VM
workers on those nodes without any node labels. Bare-metal nodes have
`/dev/kvm` natively. VMs have it when nested virtualization is on.

Once Substrate is installed, this lists the nodes that can run micro-VM
workers. Those nodes print a number:

```sh
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.ate\.dev/kvm}{"\n"}{end}'
```

To reserve a node pool for micro-VM workers, taint its nodes
`ate.dev/sandboxClass=microvm:NoSchedule`. Micro-VM worker pods tolerate that
taint. To use another taint, add its toleration to the WorkerPool's
`spec.template.tolerations`.

### GKE

`go run ./tools/setup-gcp bootstrap` creates its node pool with nested
virtualization on, so a cluster it creates can run micro-VM workers.

On an existing cluster, add a node pool with nested virtualization, or one with
a bare-metal (`-metal`) machine type. Nested virtualization can only be set
when the pool is created. See
[nested virtualization](https://cloud.google.com/kubernetes-engine/docs/how-to/nested-virtualization)
for the supported machine types.

```sh
gcloud container node-pools create microvm-pool \
  --cluster=CLUSTER --location=LOCATION \
  --machine-type=n2-standard-8 \
  --enable-nested-virtualization \
  --enable-autoscaling --min-nodes=0 --max-nodes=10 \
  --node-labels=clusterautoscaler-nodetemplate-resources.ate.dev/kvm=4096
```

The cluster autoscaler adds nodes to a pool by copying the pool's running
nodes, which advertise `ate.dev/kvm`. The
`clusterautoscaler-nodetemplate-resources.ate.dev/kvm` label tells it the same
thing about a pool with no nodes, so the pool can scale up from zero. Node
pools that a ComputeClass creates automatically don't pass this label to the
autoscaler yet
([#2691](https://github.com/agent-substrate/substrate/issues/2691)). Keep at
least one node in those with the ComputeClass `minimumCapacity`.

## Enable micro-VM

Run this after the [Quickstart](../README.md#quickstart-development) has
installed Substrate. The script assembles the guest assets with `curl`, `tar`
and `zstd`, so your workstation needs those.

### On GKE

The script reads `.ate-dev-env.sh` from the repository root, like the other
`hack/` scripts:

```sh
hack/install-microvm-deps.sh --install
```

The script downloads the guest assets, uploads them to
`gs://$BUCKET_NAME/kata-assets/`, and applies the cluster-wide `microvm`
SandboxConfig. It builds the assets for the architecture in
`KO_DEFAULTPLATFORMS`, or for your workstation's architecture when that is
unset. Set `ARCH=amd64` or `ARCH=arm64` to match your nodes.

### On kind

The kind cluster needs a Linux host with `/dev/kvm` and rootful Docker.
`hack/create-kind-cluster.sh` mounts `/dev/kvm` into the nodes when the host
has it. On Apple Silicon, use Lima as described in
[Running the microVM runtime locally](dev/microvm-local.md).

```sh
NO_DEV_ENV=true ATE_INSTALL_KIND=true hack/install-microvm-deps.sh --install
```

The script stages the assets in the cluster's rustfs bucket and applies the
`microvm` SandboxConfig.

`hack/run-microvm-demo-kind.sh` installs the control plane, runs this step and
deploys the counter demo in one command:

```sh
ATE_CREDENTIAL_PROVIDER='{"name":"k8s.io"}' hack/run-microvm-demo-kind.sh
```

### Check the install

```sh
kubectl get sandboxconfigs
```

The list shows `gvisor-default` and `microvm`. To run a micro-VM actor, deploy
the counter demo and follow its [micro-VM steps](../demos/counter/README.md):

```sh
go run ./cmd/ate-setup deploy demo counter-microvm     # GKE
hack/install-ate-kind.sh deploy demo counter-microvm   # kind
```

## Run actors on micro-VM

Set the micro-VM class and worker image on a WorkerPool:

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: agents-microvm
  namespace: my-agents
  labels:
    runtime: microvm
spec:
  replicas: 2
  sandboxClasses:
  - name: microvm
  workerImage: ko://go.ate.dev/substrate/cmd/ateworker-microvm
  template:
    resources:
      limits:
        cpu: "1"
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 1Gi
```

The pool resolves the `microvm` SandboxConfig because it is the default for its
class. Point an `ActorTemplate` at the pool with `workerSelector`:

```yaml
workerSelector:
  matchLabels:
    runtime: microvm
```

[`demos/counter/counter-microvm.yaml.tmpl`](../demos/counter/counter-microvm.yaml.tmpl)
and
[`counter-microvm-template.yaml.tmpl`](../demos/counter/counter-microvm-template.yaml.tmpl)
are complete examples. See the [API Configuration Guide](api-guide.md) for
every field.

## Upgrades

After you upgrade Substrate, run `hack/install-microvm-deps.sh --install` again
from the new release's checkout. It stages that release's guest assets and
updates the `microvm` SandboxConfig to match. See the
[Upgrade Runbook](upgrade.md) for the rest of the upgrade.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Micro-VM workers stay `Pending` | No node advertises `ate.dev/kvm` | Run the node check above. Add a node pool with nested virtualization, or a bare-metal pool. |
| Micro-VM workers stay `Pending` on a tainted pool | The pool's taint is not `ate.dev/sandboxClass=microvm` | Use that taint, or add a toleration to `spec.template.tolerations`. |
| `hack/run-microvm-demo-kind.sh` fails with `a credential provider is required` | `ATE_CREDENTIAL_PROVIDER` is unset | Set it as shown above. |
| The kind KVM probe fails with `permission denied` | Rootless Docker | See [Running the microVM runtime locally](dev/microvm-local.md#troubleshooting). |
