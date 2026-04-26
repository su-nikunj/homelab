# Homelab
A homelab setup using a k3s cluster. Two servers are running proxmox with multiple VMs to simulate more than 2 nodes.

## Nodes
| Server | Name | Services |
| ------ | ---- | -------- |
| Home Server 1 | sage | k3s control plane and several services to manage my home |
| Home Server 2 | bard | Media server with services like jellyfin |

## VMs
| Server name | VM Name | Services |
| ----------- | ------- | -------- |
| sage        | hearth  | Home assistant OS (not controlled by the cluster) |
| sage        | atrium  | k3s control plane |
| sage        | alcove-N| k3s worker nodes (alcove-1, alcove-2, etc) |
| sage        | archive | Misc containers not managed by the cluster |
| bard        | tavern  | jellyfin and related services |
| bard        | parlor-N| k3s worker nodes (parlor-1, parlor-2, etc) |

I use FluxCD to use this git repository as the single source of truth to manage both servers. Any change here is reflected in appropriate server.

## Installing k3s
1. Install k3s server on `atrium` using this command:
```bash
curl -sfL https://get.k3s.io | sh -
```
2. On `atrium` copy the k3s node token from `/var/lib/rancher/k3s/server/node-token`.
3. Now on `alcove-N` and `parlor-N`, run the following command to install the k3s agent:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<ip address of atrium>:6443 K3S_TOKEN=<sage node token> sh -
```
4. On `atrium`, running `sudo kubectl get nodes` should show all nodes connected together. Optionally, kubectl can be made to run without using sudo with the following commands:
```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml .kube/config
sudo chown $(id -u):$(id -g) .kube/config
export KUBECONFIG=$HOME/.kube/config # Or add it to .bashrc
```

## Installing and bootstrapping FluxCD
1. On `atrium`, install FluxCD using the command
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```
2. Check the prerequisites using
```bash
flux check --pre
```
3. Generate a ssh key on `atrium` for bootstraping FluxCD with SSH directly instead of using a Personal Access Token.
```bash
ssh-keygen -t ed25519 -C "flux-deploy-key" -f ~/.ssh/flux-deploy-key -N ""
```
4. Copy the `~/.ssh/flux-deploy-key.pub` file contents and add a SSH key on github or related git hosting service.
5. Finally bootstrap FluxCD using the following command on `atrium`:
```bash
flux bootstrap git \
  --url=ssh://git@github.com/su-nikunj/homelab \
  --branch=main \
  --private-key-file="$HOME/.ssh/flux-deploy-key" \
  --password="" \
  --path=clusters/homelab \
  --components-extra=image-reflector-controller,image-automation-controller
```
6. Now all the changes pushed to the git repo should be picked automatically by FluxCD and deployed in the cluster.
