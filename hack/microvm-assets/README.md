# Micro-VM runtime assets (fetch-not-bake)

The `microvm` runtime (`cmd/ateworker-microvm`, kata + cloud-hypervisor) fetches its
toolchain at runtime. Nothing kata-specific is baked into the worker image. ateworker drives
the kata-agent directly (no kata shim, no containerd). Each actor container's rootfs is an
overlay of a read-only lower (the OCI image, served into the guest over virtio-fs by
`virtiofsd`) and a writable upper, so `virtiofsd` is part of the asset set. The set is four
files, named here by their `SandboxConfig` asset key:

- `cloud-hypervisor`: the VMM binary (from its release)
- `virtiofsd`: the virtio-fs daemon serving the RO lower (from kata-static)
- `kata-kernel` (`vmlinux`): the guest kernel (from kata-static)
- `kata-image` (`rootfs.img`): the guest rootfs image (from kata-static)

These helpers assemble the asset set for your node arch and stage it in the cluster's
bucket. [`manifests/microvm/sandboxconfig-microvm.yaml.tmpl`](../../manifests/microvm/sandboxconfig-microvm.yaml.tmpl)
is the `SandboxConfig` that points at it. When `/dev/kvm` is available,
`hack/create-kind-cluster.sh` mounts it into the node. atelet then advertises it as a
device, which is what places micro-VM workers there.

> [!TIP]
> `hack/install-microvm-deps.sh --install` runs steps 1, 3 and 4 below, on kind or GKE.
> The steps here are the manual equivalent on kind.

## Steps (on a KVM-capable Linux host matching the node arch)

1. **Assemble assets for your arch** into `bin/microvm-assets/$ARCH/`:
   ```sh
   ARCH=amd64 hack/microvm-assets/assemble.sh   # or ARCH=arm64
   ```

2. **Bring up the cluster and control plane:**
   ```sh
   hack/create-kind-cluster.sh   # mounts /dev/kvm into the nodes
   hack/install-ate-kind.sh deploy ate-system --credential-provider='{"name":"k8s.io"}'
   ```
   The kind install includes rustfs, with the bucket `ate-snapshots`.

3. **Stage assets into rustfs:**
   ```sh
   OUT="$PWD/bin/microvm-assets/amd64" hack/microvm-assets/stage-to-rustfs.sh
   ```

4. **Apply the `microvm` SandboxConfig:**
   ```sh
   sed -e 's|${BUCKET_SCHEME}|s3|g' -e 's|${BUCKET_NAME}|ate-snapshots|g' \
     manifests/microvm/sandboxconfig-microvm.yaml.tmpl | kubectl apply -f -
   ```

5. **Deploy the counter demo and drive it:**
   ```sh
   hack/install-ate-kind.sh deploy demo counter-microvm
   ```
   Create an actor from `counter-microvm`, hit the in-RAM counter to increment it, suspend
   (checkpoint), resume on a different worker pod, and confirm the count continues. That
   proves the guest-memory snapshot round-tripped across pods. See the
   [counter demo](../../demos/counter/README.md).

## Notes

- Assets may share the snapshot bucket, but must not be stored under a snapshot location
  (an ActorTemplate's `snapshotConfig.location` followed by `/atespaces/`): the node's
  object-store plugin refuses to serve anything there as an asset. The staging scripts use
  `kata-assets/`, which is outside it.
- Both architectures in the `SandboxConfig` point at the same `kata-assets/` objects, so a
  bucket holds one architecture's set. Stage the set that matches your nodes.
