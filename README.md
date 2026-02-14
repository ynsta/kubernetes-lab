# Kubernetes Lab - KVM/Libvirt

A local Kubernetes development environment provisioned using Vagrant and Libvirt (KVM), built on **Debian 13 (Trixie)**.

## 🏗 Architecture

The environment uses a private network configured on the **10.4.21.0/24** subnet.

| Machine    | Role                 | IP           | vCPU | RAM  | HDD0 | HDD1 | OS        |
|------------|----------------------|--------------|------|------|------|------|-----------|
| **jumpbox**| Administrative Host  | `10.4.21.80` | 1    | 1GB  | 20GB |  NA  | Debian 13 |
| **server** | K8s Control Plane    | `10.4.21.19` | 2    | 2GB  | 30GB |  NA  | Debian 13 |
| **node-0** | K8s Worker           | `10.4.21.20` | 4    | 8GB  | 30GB | 20GB | Debian 13 |
| **node-1** | K8s Worker           | `10.4.21.21` | 4    | 8GB  | 30GB | 20GB | Debian 13 |
| **node-2** | K8s Worker           | `10.4.21.22` | 4    | 8GB  | 30GB | 20GB | Debian 13 |

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

The Kubernetes installation follows my forked version of the [Kubernetes The Hard Way](https://github.com/ynsta/kubernetes-the-hard-way/tree/1.34.3) guide.

This guide is a **fork** of the original by [kelseyhightower](https://github.com/kelseyhightower/kubernetes-the-hard-way). 

### Key Improvements in this Fork:

* **Updated Versions:** Kubernetes **1.34.3** and other components have been updated to recent versions.
* **Helm:** Integrated support for the Helm package manager.
* **API Aggregation Layer:** Enabled to allow for custom API extensions.
* **DNS:** Pre-configured CoreDNS for service discovery.
* **Metrics API:** Deployment of the Metrics Server for resource monitoring.
* **Portmap:** Support for `hostPort` and port mapping functionality.

> [!WARNING]
> This setup is **not production-ready**. Everything must be performed manually by design. This is intentional to provide a deep understanding and a platform for experimentation with Kubernetes internals.

A [`machines.txt`](machines.txt) file is provided for use with this guide.


## Set up the Krew plugin manager

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
```

You need to reload your shell and check that krew bin directory is in your PATH (done in vagrant zsh config)

## Install useful plugins

```bash
k krew update
k krew install klock
k krew install resource-capacity
k krew install view-allocations
k resource-capacity -u
k view-allocations -u
```

## Install and configure kubecolor

```bash
# Kubecolor
wget -O /tmp/kubecolor.deb "https://kubecolor.github.io/packages/deb/pool/main/k/kubecolor/kubecolor_$(wget -q -O- https://kubecolor.github.io/packages/deb/version)_$(dpkg --print-architecture).deb"
sudo dpkg -i /tmp/kubecolor.deb
```

## Install kns

```bash
curl -s https://raw.githubusercontent.com/blendle/kns/master/bin/kns -o /tmp/kns && sudo install -v -m 755 /tmp/kns /usr/local/bin/kns && rm -v /tmp/kns
```

## Deploy Traefik Ingress as a DaemonSet

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
curl -I http://node-2
```

## Deploy Kyverno

```bash
helm upgrade --install kyverno kyverno \
  --repo https://kyverno.github.io/kyverno/ \
  --namespace kyverno \
  --create-namespace \
  --set admissionController.replicas=2 \
  --set features.policyExceptions.enabled=true
```

## Configure dnsmasq and systemd-resolved to resolve `*.klab.lan` to nodes

```bash
sudo apt install -y dnsmasq dnsutils
cat <<EOF >/tmp/klab.conf
bind-interfaces
listen-address=127.0.0.1

# Map any request to *.klab.lan to your VM IPs
address=/.klab.lan/10.4.21.20
address=/.klab.lan/10.4.21.21
address=/.klab.lan/10.4.21.22

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

## Deploy ceph with rook

### Repository Configuration

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
```

### Deploying the Operator

The operator is the control plane. It must be installed and running before any cluster configuration is applied.

```bash
helm upgrade --install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph \
  -f configs/rook-ceph/operator-values.yaml 
```

Verification: Wait until the operator pod is in the Running state:

```bash
kubectl klock pods -n rook-ceph -l app=rook-ceph-operator
```


### Deploying the Cluster

Apply the custom values.yaml to the cluster chart. This instructs the operator to build the Ceph cluster according to our 3-node, 20GB specification.

```bash
helm upgrade --install --create-namespace --namespace rook-ceph rook-ceph-cluster \
  --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f configs/rook-ceph/cluster-values.yaml
```

### Verification and Health Checks

Post-deployment, verification is essential to confirm that the deviceFilter worked and the databaseSizeMB override prevented the "device too small" error.

#### OSD Pod Status:

Check for the presence of three OSD pods.

```Bash
kubectl klock pods -n rook-ceph -l app=rook-ceph-osd
```

You should see rook-ceph-osd-0, rook-ceph-osd-1, and rook-ceph-osd-2.

#### OSD Preparation Logs:

If OSD pods are missing, check the preparation jobs. These jobs run the ceph-volume logic.

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-osd-prepare
Success Indicator: Logs showing Provisioning device /dev/vdb followed by successful LVM batch creation.
Failure Indicator: Logs showing skipping device "vdb": ["Insufficient space"] or skipping device "vdb": ["Locked"].
```

#### Ceph Cluster Status:
Access the Ceph status via the operator (or a toolbox pod).

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

```text
  cluster:
    id:     6be95a6b-62af-425b-b51b-522cdde7a7d6
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 7m)
    mgr: a(active, since 8m), standbys: b
    mds: 1/1 daemons up, 1 hot standby
    osd: 3 osds: 3 up (since 8m), 3 in (since 32m)
    rgw: 1 daemon active (1 hosts, 1 zones)
 
  data:
    volumes: 1/1 healthy
    pools:   12 pools, 169 pgs
    objects: 266 objects, 501 KiB
    usage:   127 MiB used, 60 GiB / 60 GiB avail
    pgs:     169 active+clean
 
  io:
    client:   1.3 KiB/s rd, 170 B/s wr, 2 op/s rd, 0 op/s wr
```


### Test all kind of storage

```bash
kubectl apply -f examples/cepth-tests/test.yaml
```

```bash
kubectl get pvc
```

```text
kubectl get pvc                                                                                                                    
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
cephfs-pvc-test   Bound    pvc-d9a9b436-d20a-4525-891e-220ecbfa108e   1Gi        RWX            ceph-filesystem   <unset>                 4m22s
rbd-pvc-test      Bound    pvc-6d883466-43aa-4fe5-80c9-f4ec01515de3   1Gi        RWO            ceph-block        <unset>                 4m22s
```

#### Test Block Storage (RBD)


Run this command to read the file from the Block volume:

```bash
kubectl exec rbd-test-pod -- cat /mnt/rbd/test.txt

```

**Expected output:** `Hello from Block Storage!`

#### Test Shared Filesystem (CephFS)

Run this command to read the file from the Shared Filesystem volume:

```bash
kubectl exec cephfs-test-pod -- cat /mnt/cephfs/test.txt

```

**Expected output:** `Hello from Shared Filesystem!`

#### Test Object Storage (S3 / RGW)

For the S3 bucket, we will actually ask the AWS CLI inside the pod to talk to the Ceph Object Gateway, list the contents of the bucket, and then download and print the file.

First, list the files in the bucket:

```bash
kubectl exec s3-test-pod -- sh -c 'aws s3 --endpoint-url http://$BUCKET_HOST:$BUCKET_PORT ls s3://$BUCKET_NAME/'

```

**Expected output:** You should see a timestamp, a file size, and the filename `s3-test.txt`.

Now, read the file directly out of the S3 bucket:

```bash
kubectl exec s3-test-pod -- sh -c 'aws s3 --endpoint-url http://$BUCKET_HOST:$BUCKET_PORT cp s3://$BUCKET_NAME/s3-test.txt -'

```

**Expected output:** `Hello from S3 Object Storage!`

---

#### Clean up

If all of those returned exactly what we expected, your K3s/K8s cluster is now fully integrated with Ceph!

To clean up the test resources so they don't consume your precious capacity, just run:

```bash
kubectl delete -f ceph-test.yaml
```

### Expose ceph dashboard with an ingress

```bash
kubectl apply -f examples/cepth-tests/dashboard-ingress.yaml
```

Once you open `http://dashboard.ceph.klab.lan` in your browser, you will be greeted by the Ceph login screen.

* **Username:** `admin`
* **Password:** The Operator auto-generated a secure password for you. You can extract and decode it by running this one-liner:

Get your password with:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

### rados benchmark

Open a shell in rook-ceph-tool

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

To restrict the benchmark to a maximum of 10GB, we will create a dedicated test pool, apply a byte limit (quota) to it, and then run the benchmark. The tool will automatically halt/error out once the pool hits the 15GB threshold.

```bash
# 1. Create a dedicated pool for benchmarking
ceph osd pool create testbench 128 128

# 2. Enforce a strict 10GB maximum quota on the pool
ceph osd pool set-quota testbench max_bytes 10G

# 3. Run the write benchmark 
rados bench -p testbench 15 write -b 4M -t 16 --no-cleanup

# Run the read benchmark 
rados bench -p testbench 15 seq -t 16

# Cleanup
rados -p testbench cleanup
```

> **Note 1:** We use `--no-cleanup` so the 10GB of data remains in the pool, allowing you to run subsequent read tests against that specific data block.
> **Note 2:** If the clean up get stuck it might be that the OSD are full see bellow. 

---

#### Emergency Cleanup: When the Disk is Completely Full

If a benchmark runs unbounded and fills your underlying OSD disks to their `full_ratio` capacity (default is 95%), Ceph will automatically trigger a hard safety lock.


When Ceph hits its full ratio, it **blocks all write operations** to prevent database corruption. However, deleting data in Ceph actually requires *writing* deletion metadata (tombstones) to the OSDs. Because writes are globally blocked, your `rados cleanup` or pool deletion commands will simply hang indefinitely.

To clear the full disks, you must temporarily raise the full-ratio limit. This gives Ceph enough artificial "breathing room" to process the deletion metadata so you can nuke the data and re-engage the safety lock.

**Step 1: Verify the cluster is locked due to full OSDs**

```bash
ceph -s
```

*(You will see a `HEALTH_ERR` state with a warning like `OSD_FULL: 1 full osd(s)`).*

**Step 2: Temporarily raise the full ratio to 99%**
This tricks the cluster into thinking it has free space, allowing writes (and deletes) to process.

```bash
ceph osd set-full-ratio 0.99
```

*Wait a few seconds. Run `ceph -s` again until the cluster drops to `HEALTH_WARN` (nearfull).*

**Step 3: Delete the benchmark data**
You can now successfully run your cleanup commands without them hanging:

```bash
# Option A: Clean up the benchmark objects safely
rados -p testbench cleanup

# Option B: Delete the entire pool (Much faster for large amounts of data)
ceph tell mon.* injectargs '--mon-allow-pool-delete=true'
ceph osd pool delete testbench testbench --yes-i-really-really-mean-it
ceph tell mon.* injectargs '--mon-allow-pool-delete=false'
```

**Step 4: CRITICAL - Revert the safety limits**
Once `ceph osd df` confirms your storage space has been reclaimed, you **must** reset the cluster's safety lock to prevent catastrophic crashes in the future.

```bash
ceph osd set-full-ratio 0.95
```


## Install kubestr

[Kubestr](https://github.com/kastenhq/kubestr) is a collection of tools to discover, validate and evaluate your kubernetes storage options.

```bash
{
  wget https://github.com/kastenhq/kubestr/releases/download/v0.4.49/kubestr_0.4.49_Linux_amd64.tar.gz
  tar -xzf kubestr_0.4.49_Linux_amd64.tar.gz kubestr
  sudo install -v -m 755 kubestr /usr/local/bin/
  rm kubestr_0.4.49_Linux_amd64.tar.gz kubestr
}
```

Run a fio test:

```bash
kubestr fio -s ceph-block --size 10Gi
```


```text
PVC created kubestr-fio-pvc-qxct6
Pod created kubestr-fio-pod-z4pxh
Running FIO test (default-fio) on StorageClass (ceph-block) with a PVC of Size (10Gi)
Elapsed time- 31.005216056s
FIO test results:
  
FIO version - fio-3.36
Global options - ioengine=libaio verify=0 direct=1 gtod_reduce=1

JobName: read_iops
  blocksize=4K filesize=2G iodepth=64 rw=randread
read:
  IOPS=4516.090332 BW(KiB/s)=18081
  iops: min=3948 max=5351 avg=4545.344727
  bw(KiB/s): min=15792 max=21405 avg=18182.035156

JobName: write_iops
  blocksize=4K filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=1384.712402 BW(KiB/s)=5555
  iops: min=1151 max=2632 avg=1391.586182
  bw(KiB/s): min=4606 max=10530 avg=5567.275879

JobName: read_bw
  blocksize=128K filesize=2G iodepth=64 rw=randread
read:
  IOPS=4216.318848 BW(KiB/s)=540225
  iops: min=3394 max=5010 avg=4247.655273
  bw(KiB/s): min=434432 max=641280 avg=543731.687500

JobName: write_bw
  blocksize=128k filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=1337.141357 BW(KiB/s)=171689
  iops: min=1139 max=2278 avg=1344.379272
  bw(KiB/s): min=145884 max=291584 avg=172116.687500

Disk stats (read/write):
  rbd0: ios=146995/45202 merge=0/12 ticks=1862837/2004973 in_queue=3867810, util=99.568275%
  -  OK
```

## Some Examples

Explore practical use cases for your Kubernetes lab:

* **[App with Ingress](examples/app-with-ingress/README.md)**: Deploy a sample web application and expose it using Traefik and the `*.klab.lan` DNS wildcard.
