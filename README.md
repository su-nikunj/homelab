# Homelab
A homelab setup using k3s cluster. It has two clusters, one at a VPS hosting critical public facing services, and one at home, managing two nodes. The home servers are running proxmox with multiple VMs, but the VPS is a single debian instance so it will just be a single node cluster and won't have any load balancing or replications.

| Server | Name | Services |
| ------ | ---- | -------- |
| Home Server 1 | sage | k3s server and several services to manage my home |
| Home Server 2 | bard | Media server with services like jellyfin |
| VPS | herald | Public facing services that require 24/7 availability |

I use FluxCD to use this git repository as the single source of truth to manage all 3 servers. Any change here is reflected in appropriate server.

## Getting Started
1. Install k3s server on `sage` and `herald` using this command:
```bash
curl -sfL https://get.k3s.io | sh -
```
2. On `sage` copy the k3s node token from `/var/lib/rancher/k3s/server/node-token`.
3. Now on `bard`, run the following command to install the k3s agent:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<ip address of sage>:6443 K3S_TOKEN=<sage node token> sh -
```
4. On `sage`, running `sudo kubectl get nodes` should show 2 nodes connected together. On `herald`, it should show only 1. Optionally, kubectl can be made to run without using sudo with the following commands:
```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml .kube/config
sudo chown $(id -u):$(id -g) .kube/config
export KUBECONFIG=$HOME/.kube/config # Or add it to .bashrc
```

## Bootstraping FluxCD
1. On `sage` and `herald`, install FluxCD using the command
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```
2. Check the prerequisites using
```bash
flux check --pre
```
3. Generate a ssh key on `sage` for bootstraping FluxCD with SSH directly instead of using a Personal Access Token. Copy the same keys on `herald`.
```bash
ssh-keygen -t ed25519 -C "flux-deploy-key" -f ~/.ssh/flux-deploy-key -N ""
```
4. Copy the `~/.ssh/flux-deploy-key.pub` file contents and add a SSH key on github or related git hosting service.
5. Finally bootstrap FluxCD using the following command
```bash
flux bootstrap git \
  --url=ssh://git@github.com/su-nikunj/homelab \
  --branch=main \
  --private-key-file="$HOME/.ssh/flux-deploy-key" \
  --password="" \
  --path=clusters/cloud \ # clusters/home for sage
  --components-extra=image-reflector-controller,image-automation-controller
```
6. Now all the changes pushed to the git repo should be picked automatically by FluxCD and deployed in the clusters.
