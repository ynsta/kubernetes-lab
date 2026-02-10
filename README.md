# Kubernetes Lab - KVM/Libvirt

Local Kubernetes development environment provisioned via Vagrant and Libvirt (KVM).
Built on **Debian 13 (Trixie)**.

## 🏗 Architecture

The private network is configured on the **10.4.21.0/24** subnet.

| Machine    | Role                   | IP           | vCPU | RAM  | HDD  | OS        |
|------------|------------------------|--------------|------|------|------|-----------|
| **jumpbox**| Admin Host / Gateway   | `10.4.21.80` | 1    | 1GB  | 20GB | Debian 13 |
| **server** | K8s Control Plane      | `10.4.21.19` | 2    | 4GB  | 40GB | Debian 13 |
| **node-0** | K8s Worker             | `10.4.21.20` | 2    | 4GB  | 40GB | Debian 13 |
| **node-1** | K8s Worker             | `10.4.21.21` | 2    | 4GB  | 40GB | Debian 13 |

> **Performance Note:** The CPU mode is set to `host-passthrough` to utilize native host processor instructions, ensuring near-native performance.

## 🚀 Quick Start

Ensure you have `vagrant`, `libvirt` and the `vagrant-libvirt` plugin installed.

```bash
# 1. Launch the full environment (provisions all 4 VMs)
vagrant up

# 2. Verify machines are running
vagrant status

```

## 🎮 Vagrant Commands (Cheat Sheet)

### Lifecycle Management

| Action | Command | Description |
| --- | --- | --- |
| **Start** | `vagrant up` | Creates and configures VMs according to the Vagrantfile. |
| **Stop** | `vagrant halt` | Gracefully shuts down the VMs (ACPI shutdown). |
| **Restart** | `vagrant reload` | Reboots the VMs (useful after editing Vagrantfile). |
| **Suspend** | `vagrant suspend` | Pauses the VMs (saves RAM state to disk). |
| **Destroy** | `vagrant destroy` | Permanently deletes the VMs and their disks. |

### Access & Debug

| Action | Command | Description |
| --- | --- | --- |
| **Connect** | `vagrant ssh <name>` | SSH into a VM (e.g., `vagrant ssh server`). |
| **SSH Config** | `vagrant ssh-config` | Outputs SSH configuration (keys/ports) for external tools. |
| **Logs** | `vagrant global-status` | Lists all active Vagrant environments on the host. |

## 🛠 Specific Configuration

* **OS:** Uses Debian Testing (Trixie) to provide a recent kernel version.

* **Network:** VMs communicate via the private libvirt network bridge.
* **Forwarding:** IP forwarding is enabled by default to allow pod traffic routing.

## 📝 Usage Examples

**1. Access the Control Plane:**

```bash
vagrant ssh server
# Check OS version
cat /etc/os-release
```

**2. Test Connectivity from Jumpbox:**

```bash
vagrant ssh jumpbox
ping 10.4.21.20
```


## Kubernetes Setup

Follow [Kubernetes The Hard Way](https://github.com/ynsta/kubernetes-the-hard-way/tree/1.34.3) Guide
 
A [`machines.txt`](machines.txt) file is provided to be used within this guide.
