A homelab setup using k3s cluster. It have 3 servers, one VPS and two servers at home. They are connected through tailscale as there is a native integration of tailscale in k3s introduced recently (though still marked as experimental at this point). Their structure is as follows:

|   | Name | Role | Availability | Services |
|---|------|------|--------------|----------|
| VPS | Sage | k3s server | Guaranteed 24/7 operation | Internet facing services like nextcloud, immich, vaultwarden |
| Home Server 1 | Scribe | General Purpose Server | Can be turned off occassionally | Home related services like Home assistant, pihole, some dashboards |
| Home Server 2 | Bard | Media Server | Will be turned off when not in use | Jellyfin and related services

The k3s is configured such that services would run on specified machines only and not try to implement load balancing.

I also use FluxCD to use this git repository as the single source of truth to manage all 3 servers. Any change here is reflected in appropriate server.

# Getting Started
1. Create a [tailscale](https://tailscale.com) account. The free tier is more than enough for self hosting needs. Self hosting headscale is also an option, but recommended to do on a separate machine/VM than the k3s server.
2. Follow the instructions [here](https://docs.k3s.io/networking/distributed-multicloud#integration-with-the-tailscale-vpn-provider-experimental) to generate an auth key on tailscale and install the tailscale client on all the machines.
3. Install k3s server on `sage` using this command:
```bash
curl -sfL https://get.k3s.io | sh -s - --vpn-auth="name=tailscale,joinKey=<tailscale_auth_key>"
```
4. This should create a new node on tailscale dashboard called `sage`. Copy the ipv4 address of this node. Also copy the k3s node token from `/var/lib/rancher/k3s/server/node-token`.
5. Now on `scribe` and `bard`, run the following command to install the k3s agent:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<sage_ipv4>:6443 K3S_TOKEN=<sage_node_token> sh -s - --vpn-auth="name=tailscale,joinKey=<tailscale_auth_key>"
```
6. On `sage`, running `sudo kubectl get nodes` should show all 3 nodes connected together. Optionally, kubectl can be made to run without using sudo with the following commands:
```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml .kube/config
sudo chown $(id -u):$(id -g) .kube/config
export KUBECONFIG=$HOME/.kube/config # Or add it to .bashrc
```

# Bootstraping FluxCD
1. On `sage`, install FluxCD using the command
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```
2. Check the prerequisites using
```bash
flux check --pre
```
3. Generate a ssh key on `sage` for bootstraping FluxCD with SSH directly instead of using a Personal Access Token.
```bash
ssh-keygen -t ed25519 -C "flux-deploy-key" -f ~/.ssh/flux-deploy-key -N ""
```
4. Copy the `~/.ssh/flux-deploy-key.pub` file contents and add a SSH key on github or related git hosting service.
5. Finally bootstrap FluxCD using the following command
```bash
flux bootstrap git \
  --url=git@github.com:su-nikunj/homelab.git \
  --branch=main \
  --private-key-file=~/.ssh/flux-deploy-key \
  --path=clusters/homelab
```
6. Now all the changes pushed to the git repo should be picked automatically by FluxCD and deployed in the cluster.
