# Kubernetes Lab - KVM/Libvirt

A local Kubernetes development environment provisioned using Vagrant and Libvirt (KVM), built on **Debian 13 (Trixie)**.

## 🏗 Architecture

The environment uses a private network configured on the **10.4.21.0/24** subnet.

| Machine    | Role                 | IP           | vCPU | RAM  | HDD  | OS        |
|------------|----------------------|--------------|------|------|------|-----------|
| **jumpbox**| Administrative Host  | `10.4.21.80` | 1    | 1GB  | 20GB | Debian 13 |
| **server** | K8s Control Plane    | `10.4.21.19` | 2    | 4GB  | 40GB | Debian 13 |
| **node-0** | K8s Worker           | `10.4.21.20` | 2    | 4GB  | 40GB | Debian 13 |
| **node-1** | K8s Worker           | `10.4.21.21` | 2    | 4GB  | 40GB | Debian 13 |

> **Performance Note:** The CPU mode is set to `host-passthrough` to use native host processor instructions, ensuring near-native performance.

## 🚀 Quick Start

Ensure that `vagrant`, `libvirt`, and the `vagrant-libvirt` plugin are installed.

```bash
# 1. Launch the full environment (provisions all 4 VMs):
vagrant up

# 2. Verify that the machines are running:
vagrant status

```

## 🎮 Vagrant Commands (Cheat Sheet)

### Lifecycle Management

| Action | Command | Description |
| --- | --- | --- |
| **Start** | `vagrant up` | Creates and configures the VMs according to the Vagrantfile. |
| **Stop** | `vagrant halt` | Gracefully shuts down the VMs. |
| **Restart** | `vagrant reload` | Reboots the VMs (useful after editing the Vagrantfile). |
| **Suspend** | `vagrant suspend` | Pauses the VMs (saves the RAM state to disk). |
| **Destroy** | `vagrant destroy` | Permanently deletes the VMs and their associated disks. |

### Access & Debug

| Action | Command | Description |
| --- | --- | --- |
| **Connect** | `vagrant ssh <name>` | Connects to a VM via SSH (e.g., `vagrant ssh server`). |
| **SSH Config** | `vagrant ssh-config` | Outputs the SSH configuration (keys/ports) for use with external tools. |
| **Logs** | `vagrant global-status` | Lists all active Vagrant environments on the host machine. |

## 🛠 Specific Configuration

* **OS:** Uses Debian Testing (Trixie) to provide a recent kernel version.
* **Network:** VMs communicate via a private libvirt network bridge.
* **Forwarding:** IP forwarding is enabled by default to allow pod traffic routing.

## 📝 Usage Examples

**1. Accessing the Control Plane:**

```bash
vagrant ssh server
# Check OS version
cat /etc/os-release
```

**2. Testing Connectivity from the Jumpbox:**

```bash
vagrant ssh jumpbox
ping 10.4.21.20
```


## Kubernetes Setup

Follow the [Kubernetes The Hard Way](https://github.com/ynsta/kubernetes-the-hard-way/tree/1.34.3) guide.

A [`machines.txt`](machines.txt) file is provided for use with this guide.

## Post-installation

Perform the following steps on the `jumpbox`:

### Set up the Krew plugin manager

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

echo 'export PATH="$PATH:${KREW_ROOT:-$HOME/.krew}/bin"' >> ~/.zshrc

```

Restart your shell (zsh) to apply changes.

### Install useful plugins

```bash
k krew update
k krew install klock
k krew install resource-capacity
k krew install view-allocations
k resource-capacity -u
k view-allocations -u
```

### Install and configure kubecolor

```bash
# Kubecolor
wget -O /tmp/kubecolor.deb "https://kubecolor.github.io/packages/deb/pool/main/k/kubecolor/kubecolor_$(wget -q -O- https://kubecolor.github.io/packages/deb/version)_$(dpkg --print-architecture).deb"
sudo dpkg -i /tmp/kubecolor.deb

# Update the 'k' alias
sed -i -e s/k='kubectl'/k='kubecolor'/ .zshrc

```

### Install kns

```bash
curl -s https://raw.githubusercontent.com/blendle/kns/master/bin/kns -o /tmp/kns && sudo install -v -m 755 /tmp/kns /usr/local/bin/kns && rm -v /tmp/kns
```
### Deploy Traefik Ingress as a DaemonSet

```bash
helm repo add traefik https://traefik.github.io/charts
helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set ports.web.hostPort=80 \
  --set ports.websecure.hostPort=443 \
  --set service.type=ClusterIP \
  --set deployment.kind=DaemonSet \
  --set resources.requests.cpu="100m" \
  --set resources.requests.memory="64Mi" \
  --set resources.limits.cpu="300m" \
  --set resources.limits.memory="128Mi"
```

Verify that it can be accessed via all nodes:

```bash
curl -I http://node-0
curl -I http://node-1
```

### Deploy Kyverno

```bash
helm upgrade --install kyverno kyverno \
  --repo https://kyverno.github.io/kyverno/ \
  --namespace kyverno \
  --set admissionController.replicas=2 \
  --set features.policyExceptions.enabled=true
```

### Configure dnsmasq and systemd-resolved to resolve `*.klab.lan` to nodes

```bash
sudo apt install -y dnsmasq dnsutils
cat <<EOF >/tmp/klab.conf
bind-interfaces
listen-address=127.0.0.1

# Map any request to *.klab.lan to your VM IPs
address=/.klab.lan/10.4.21.20
address=/.klab.lan/10.4.21.21

# Optional: Ensure it doesn't try to forward these local queries to the internet
local=/klab.lan/
EOF
sudo cp /tmp/klab.conf /etc/dnsmasq.d/klab.conf
sudo systemctl restart dnsmasq.service

sudo mkdir -p /etc/systemd/resolved.conf.d/
cat <<EOF >/tmp/klab.conf
[Resolve]
DNS=127.0.0.1
Domains=~klab.lan
EOF
sudo cp /tmp/klab.conf /etc/systemd/resolved.conf.d/
sudo systemctl restart systemd-resolved
```

Verify the configuration:

```bash
curl -I http://test.test.klab.lan
```

```text
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Tue, 10 Feb 2026 23:58:24 GMT
Content-Length: 19
```

This configuration can also be applied to the host machine, allowing you to access the nodes via the `*.klab.lan` DNS wildcard.


## 📚 Examples

Explore practical use cases for your Kubernetes lab:

* **[App with Ingress](examples/app-with-ingress/README.md)**: Deploy a sample web application and expose it using Traefik and the `*.klab.lan` DNS wildcard.
