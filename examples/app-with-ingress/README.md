# Example: App with Traefik Ingress

This example demonstrates how to deploy a simple web application and expose it to the network using the **Traefik Ingress Controller**.

## 📋 Overview

The deployment includes:
- **Deployment**: Runs 2 replicas of the `jpetazzo/color` image (displaying a blue color).
- **Service**: A `ClusterIP` service to load balance traffic across the pods.
- **Ingress**: A Traefik-managed ingress rule mapping `blue.rainbow.klab.lan` to the service.

## 🚀 Deployment

Copy the examples folder to the `jumpbox` amd run these commands from it:

1. **Navigate to the example directory:**
```bash
cd examples/app-with-ingress
```

2. **Create a dedicated namespace:**
```bash
k create ns rainbow
```

3. **Apply the manifest:**
```bash
k apply -f . -n rainbow
```

## 🔍 Verification

### 1. Check the resources

Verify that the pods are running and the ingress is created:

```bash
k get all,ing -n rainbow
```

### 2. Test connectivity

Once the pods are `Running`, test the ingress from the `jumpbox` (or any machine where `*.klab.lan` resolves to your nodes):

```bash
curl http://blue.rainbow.klab.lan
```

output:

```text
🔵This is pod rainbow/blue-55cfd6f77d-8bssg on linux/amd64, serving / for 10.4.21.21:46240.
```

> **Note:** This works because the [main setup](../../README.md#configure-dnsmasq-and-systemd-resolved-to-resolve-klablan-to-nodes) configures `*.klab.lan` to resolve to your Kubernetes nodes where Traefik is listening on port 80.

## 🧹 Cleanup

To remove all resources created for this example:

```bash
k delete ns rainbow
```
