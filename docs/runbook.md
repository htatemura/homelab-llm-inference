# Runbook

Operational procedures for the home lab cluster, recorded as they were performed.
Placeholders in angle brackets (`<harbor-host>`, `<node>`) replace real addresses.

**Environment**

| Item | Value |
|---|---|
| Hypervisor | KVM / libvirt on a single host |
| Cluster | kubeadm, Kubernetes v1.37, containerd 2.2 |
| Nodes | `k8s-control`, `k8s-worker1`, `k8s-worker2-llm` (label `workload=llm`) |
| CNI | Cilium 1.20 |
| Registry | Harbor (Docker Compose, outside the cluster, HTTP) |
| Workstation | Laptop with `kubectl` access to the cluster |

---

## 1. Rename a worker node

A node's name is baked into its kubelet client certificate, so changing the hostname alone breaks authentication. The node must leave and rejoin the cluster.

**Pre-check** (workstation): make sure nothing stateful runs on the node.

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=<old-name>
kubectl get pv
```

Only DaemonSet Pods (cilium, cilium-envoy, kube-proxy) were present, and no PVs, so the node was safe to remove.

**Remove the node** (control plane):

```bash
kubectl drain <old-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <old-name>
kubeadm token create --print-join-command   # keep the output for rejoining
```

**Reset and rename** (on the node):

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo hostnamectl set-hostname <new-name>          # lowercase only: Kubernetes node names must be lowercase
sudo sed -i 's/<old-name>/<new-name>/g' /etc/hosts
sudo reboot                                       # clears leftover Cilium interfaces and iptables rules
```

Check `/etc/hosts` on the other nodes for the old name as well.

**Rejoin** (on the node):

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify and restore labels** (workstation):

```bash
kubectl get nodes -o wide
kubectl label node <new-name> node-role.kubernetes.io/worker=
kubectl label node <new-name> workload=llm
cilium status
```

**Clean up** (control plane): delete the join token once it is no longer needed.

```bash
kubeadm token delete <token-id>
```

> Lesson: labels live on the Node object, so deleting and rejoining a node removes them. They must be re-applied.

---

## 2. Resize a worker VM (CPU and memory)

Target for the LLM worker: **8 vCPU / 24 GB RAM**, sized for a 12 GB GPU (system RAM of roughly 1.5 to 2 times VRAM for model loading).

**Drain** (workstation):

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

**Resize and rename the libvirt domain** (hypervisor host). The libvirt domain name is separate from the guest hostname; renaming it keeps the two consistent.

```bash
virsh shutdown <domain>
virsh list --all                      # wait for "shut off"
virsh setvcpus <domain> 8 --config --maximum
virsh setvcpus <domain> 8 --config
virsh setmaxmem <domain> 24G --config
virsh setmem <domain> 24G --config
virsh domrename <domain> <new-domain>
virsh start <new-domain>
```

**Verify** (on the node), then **uncordon** (workstation):

```bash
nproc        # 8
free -h      # ~22 GiB usable
kubectl uncordon <node>
```

> Note: a VM with GPU passthrough keeps its full memory allocation pinned while running. Plan the host's total memory budget accordingly.

---

## 3. Extend the root filesystem using free LVM space

The Ubuntu installer allocated only half of the 60 GB disk to the root logical volume, leaving the rest unallocated in the volume group.

**Check** (on the node):

```bash
df -h /
lsblk
sudo vgs            # VFree showed 29 GB unallocated
```

**Extend** the logical volume and the ext4 filesystem in one step, online, with no reboot:

```bash
sudo lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /             # 29 GB -> 57 GB
```

If the volume group has no free space, grow the VM disk first:

```bash
# hypervisor host
virsh domblklist <domain>
virsh blockresize <domain> vda <size>G
# node
sudo growpart /dev/vda 3
sudo pvresize /dev/vda3
sudo lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```

> Lesson: other nodes built from the same installer likely have the same unallocated space.

---

## 4. Install crictl on a node

`crictl` inspects containers and images through the same CRI interface the kubelet uses. It was missing on the node.

```bash
sudo apt update
sudo apt install -y cri-tools
sudo tee /etc/crictl.yaml > /dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF
```

Use `crictl`, not `ctr`, to test whether Kubernetes can pull an image: `ctr` bypasses the CRI registry configuration.

```bash
sudo crictl pull <harbor-host>/k8s/ollama:0.34.4
```

---

## 5. Push a large image to Harbor over HTTP

**Allow the HTTP registry** (workstation Docker):

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "insecure-registries": ["<harbor-host>"]
}
EOF
sudo systemctl restart docker
docker info | grep -A2 "Insecure Registries"
```

**Pin a version, tag and push**. A fixed tag avoids silent changes from `latest`. Release candidates (`-rc`) and AMD builds (`-rocm`) were skipped; the standard tag supports both CPU and NVIDIA GPUs.

```bash
VER=0.34.4
docker login <harbor-host>
docker pull ollama/ollama:$VER
docker tag ollama/ollama:$VER <harbor-host>/k8s/ollama:$VER
docker push <harbor-host>/k8s/ollama:$VER
```

**Troubleshooting**: the 3.6 GB layer uploaded fully, then failed with `timeout awaiting response headers` while Harbor finalized the blob. Re-running `docker push` succeeded: completed layers were skipped and the large layer was committed.

---

## 6. Install local-path-provisioner

Dynamic storage on the node's local disk, used for model files. Data lives under `/opt/local-path-provisioner` on the node running the Pod.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl get pods -n local-path-storage
kubectl get storageclass     # local-path, WaitForFirstConsumer, reclaim policy Delete
```

> Note: with reclaim policy `Delete`, deleting the PVC deletes the downloaded models. local-path does not enforce the requested size, so free disk space on the node must be checked manually.

---

## Troubleshooting notes

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl` on a worker: `connection to the server localhost:8080 was refused` | Worker nodes have no admin kubeconfig | Run `kubectl` from the control plane or the workstation |
| `sudo: crictl: command not found` | `cri-tools` not installed | Section 4 |
| `docker push` fails after a large layer reaches 100% | Registry timed out while finalizing the blob | Re-run `docker push` |
| Node labels missing after rejoin | Labels belong to the deleted Node object | Re-apply labels (Section 1) |
