# Elite Cluster
The upgrade and continuation of the [atomic cluster]([url](https://github.com/todaywasawesome/atomic-cluster)https://github.com/todaywasawesome/atomic-cluster). Overall, the Atomic Cluster was difficult to work with because it was underpowered. The Elite cluster adds 3 Dell Elite Desk G3s.


## Nodes

Everything set up by hand on the nodes. Argo CD manages everything else, from this repo. To rebuild a node, work through this section in order.

| Node | Role | IP | Hardware |
|---|---|---|---|
| elite1 | k3s server (control plane) | 10.0.0.21 | Dell EliteDesk G3, Intel i5-7600 (4 cores), 32 GB |
| elite2 | k3s agent | 10.0.0.22 | same |
| elite3 | k3s agent | 10.0.0.23 | same |

OS: Ubuntu 24.04 LTS.

### 1. BIOS (all nodes)

Turn on **Intel Virtualization Technology (VT-x)**, **VT-d** and the **TPM**. Kata Containers needs VT-x. Without it the kernel logs `VMX (outside TXT) disabled by BIOS` and `/dev/kvm` doesn't exist, and the Kata install fails on that node.

Check: `ls -l /dev/kvm`.

### 2. Kernel settings (all nodes)

The Ubuntu default of 128 inotify instances per user is too low for a Kubernetes node. k3s fails to restart with `Failed to allocate directory watch: Too many open files`.

```bash
sudo tee /etc/sysctl.d/90-inotify.conf >/dev/null <<'CONF'
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
CONF
sudo sysctl --system | grep inotify
```

### 3. k3s

Installed with the install script, using the versions from the `v1.36` channel. After that, the system-upgrade-controller Plans keep k3s current (see section 6).

**elite1 (server):**

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.36 sh -s - server \
  --disable traefik --disable servicelb \
  --prefer-bundled-bin --flannel-backend=vxlan --node-name=elite1
```

The flags end up in the systemd unit (`systemctl cat k3s`). The server's join token is in `/var/lib/rancher/k3s/server/node-token`.

**elite2, elite3 (agents):**

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.36 \
  K3S_URL=https://10.0.0.21:6443 K3S_TOKEN=<node-token from elite1> sh -s - agent
```

The install script writes `K3S_URL` and `K3S_TOKEN` to `/etc/systemd/system/k3s-agent.service.env`. Don't commit that file; it holds the token.

### 4. Embedded registry mirror (Spegel)

Nodes share pulled images with each other over the LAN, so each image is downloaded from the internet once per cluster instead of once per node. `:latest` tags are never shared between nodes. Docs: https://docs.k3s.io/installation/registry-mirror

**elite1:** `/etc/rancher/k3s/config.yaml`. This file is merged with the unit's flags; a flag wins if both set the same option.

```yaml
embedded-registry: true
```

**All nodes:** `/etc/rancher/k3s/registries.yaml`. Agents don't have `/etc/rancher/k3s` by default, so create it with `sudo mkdir -p /etc/rancher/k3s`.

```yaml
mirrors:
  "*":
```

The nodes must reach each other on TCP 5001 (P2P) and 6443. If `ufw` is on: `sudo ufw allow from 10.0.0.0/24 to any port 5001 proto tcp`.

Restart elite1 first (`sudo systemctl restart k3s`), then the agents (`sudo systemctl restart k3s-agent`). Running pods stay up through a k3s restart.

Check: `kubectl get node <name> -o jsonpath='{.metadata.annotations.p2p\.k3s\.cattle\.io/node-address}'` should print a `/ip4/10.0.0.x/tcp/5001/p2p/...` address on every node.

Trade-off: any node can serve any image it holds without the pull secret. Pin images by digest; a digest can't be spoofed by a peer.

### 5. Argo CD

Bootstrapped with argocd-autopilot from `bootstrap/` (`bootstrap/root.yaml`). Argo then installs everything under `user/apps/elite-cluster/`.

### 6. Things Argo installs onto the nodes themselves

You don't do these by hand, but they change the hosts, so they're listed here for recovery:

- **system-upgrade-controller** (`user/manifests/system-upgrade-controller`) upgrades k3s on the `v1.36` channel, Sundays 02:00-05:00 America/Denver. It upgrades elite1 first, then the agents one at a time, cordoning each node but not draining it. To move to a new minor version, change the channel URL in `plans.yaml`.
- **Kata Containers** (`user/manifests/kata-containers`): kata-deploy job mode installs to `/opt/kata` and adds a containerd drop-in that k3s imports. It also records the kernel modules it needs under `/etc/modules-load.d`, and labels each node `katacontainers.io/kata-runtime=true`. Deleting the Argo app runs the uninstall on every node. A node added later gets Kata on the next sync of that app.
  - Check: `kubectl get nodes -L katacontainers.io/kata-runtime`.
