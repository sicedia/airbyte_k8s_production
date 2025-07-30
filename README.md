# Airbyte On-Premise Deployment Guide (CEDIA)

This guide explains how to deploy a single-node Kubernetes cluster using MicroK8s and install Airbyte using Helm, step by step. It is designed for internal use at CEDIA.

---

## Prerequisites

- Ubuntu 24.04 server (recommended)
- At least 4 CPUs, 16GB RAM, and 100Gb disk space
- Root or sudo access

---

## 1. Update the Server

Update all packages to ensure your system is ready.

```sh
sudo apt update && sudo apt upgrade -y
```

---

## 2. Install MicroK8s

Install MicroK8s from the stable channel (v1.28).

```sh
sudo snap install microk8s --classic --channel=1.28/stable
```

---

## 3. Configure User Permissions

Add your user to the `microk8s` group to avoid using `sudo` for every command, and fix permissions for the Kubernetes config directory.

```sh
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

Apply the new group membership (open a new terminal or run):

```sh
newgrp microk8s
```

---

## 4. Wait for MicroK8s to be Ready

Check that MicroK8s is running and all services are ready.

```sh
microk8s status --wait-ready
```

---

## 5. Enable Essential MicroK8s Addons

Enable DNS, storage, ingress, and Helm 3. These are required for Airbyte and most Kubernetes workloads.

```sh
microk8s enable dns
microk8s enable hostpath-storage
microk8s enable ingress
microk8s enable helm3
```

---

## 6. (Optional) Create a `kubectl` Alias

To use `kubectl` instead of `microk8s kubectl`, add this alias:

```sh
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
echo "alias kubectl='microk8s heml3'" >> ~/.bashrc
source ~/.bashrc

```

---

## 7. Verify the Cluster

Check that your node is ready:

```sh
kubectl get nodes
```

Check that all system pods are running:

```sh
kubectl get pods --all-namespaces
```

---

## 8. Prepare Namespace and Helm Repo for Airbyte

Before continuing, **make sure to allocate all Airbyte-related files (such as `airbyte-values.yaml`, TLS certificates, and any configuration files) in the directory `/opt/airbyte` on your server**.  
Set the correct permissions so the user `cedia` can access and manage these files:

```sh
sudo mkdir -p /opt/airbyte
sudo chown -R cedia:cedia /opt/airbyte
```

Now, create a namespace for Airbyte and add the Airbyte Helm repository:

```sh
kubectl create namespace airbyte
helm repo add airbyte https://airbytehq.github.io/helm-charts
helm repo update
helm search repo airbyte
```

---

## 9. Create Airbyte Values File

Create a minimal [`airbyte-values.yaml`](./airbyte-values.yaml) file with your configuration. Example:

[`airbyte-values.yaml`](./airbyte-values.yaml)

---

## 10. Install Airbyte with Helm

Install Airbyte using your values file:

```sh
helm install airbyte airbyte/airbyte \
  --namespace airbyte \
  --values ./airbyte-values.yaml
```

Check the Airbyte pods:

```sh
kubectl -n airbyte get pods
```

---

## 11. Configure TLS Certificates

Copy your TLS certificates to `/opt/airbyte/certs/` on the host.

Create the Kubernetes TLS secret (replace secret name and paths as needed):

```sh
kubectl -n airbyte create secret tls dataflow-tls \
  --cert=/opt/airbyte/certs/fullchain.pem \
  --key=/opt/airbyte/certs/privkey.pem
```

---

## 12. Expose Airbyte via Ingress

Create an [`airbyte-ingress.yaml`](./airbyte-ingress.yaml) file:

[`airbyte-ingress.yaml`](./airbyte-ingress.yaml)

Apply the ingress:

```sh
kubectl apply -f airbyte-ingress.yaml
kubectl -n airbyte get ingress
```

---

## 13. Open Firewall Ports

Allow HTTP and HTTPS traffic:

```sh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

---

## 14. Useful Commands

- Watch Airbyte pods:  
  `kubectl -n airbyte get pods -w`
- Upgrade Airbyte:  
  `helm repo update`  
  `helm upgrade airbyte airbyte/airbyte -n airbyte -f airbyte-values.yaml`
- Get Airbyte resources:  
  `kubectl -n airbyte get pods,deploy,sts,job,pvc`
- Troubleshoot pods:  
  `kubectl -n airbyte describe pod <pod-name>`

---

## 15. Maintenance & Scaling

- To update Airbyte, edit `airbyte-values.yaml` and run the upgrade command above.
- For production, consider using external Postgres and object storage. See Airbyte docs for details.
- Monitor pod logs and resource usage to adjust worker counts and limits as needed.

---

## 16. Hardening and Moving Toward High Availability (HA) *(Future Plan if necessary)*

For a single-node setup, you can start immediately as described above. However, for a **production-grade** or highly available deployment, you should move critical components out of the cluster and implement backups. Below are recommendations and example configurations:

---

### a) Use External Postgres

**Why:** The default Airbyte deployment uses an internal Postgres database, which is not suitable for production. Use a managed or standalone Postgres instance outside the cluster for reliability and backup.

**How:**  
- Create a Kubernetes secret named `airbyte-config-secrets` with your DB credentials.
- See [Airbyte docs: External Database](https://docs.airbyte.com/platform/deploying-airbyte/integrations/database) for details.

---

### b) Use External Object Storage

**Why:** By default, Airbyte uses MinIO (S3-compatible) inside the cluster for logs and state. For production, use external object storage (AWS S3, Google Cloud Storage, Azure Blob, etc.) for durability and scalability.

**How:**  
Add the relevant configuration to your `airbyte-values.yaml` to disable MinIO and use your bucket(s).

- See [Airbyte docs: External Object Storage](https://airbyte.com/connectors/azure-blob-storage) for examples.

---

### c) Backups & Retention

- **Postgres:** Back up your external Postgres database regularly.
- **Object Storage:** Set up lifecycle rules for retention and deletion.
- **Temporal History:** Consider setting `TEMPORAL_HISTORY_RETENTION_IN_DAYS` (default is 30 days).

---

### d) Observability

Enable metrics and monitoring for better visibility:

- Set `PUBLISH_METRICS` and `METRIC_CLIENT` environment variables to export metrics to OTEL, Datadog, or your preferred system.
- See [Airbyte docs: Metrics](https://docs.airbyte.com/platform/operator-guides/collecting-metrics).

---

### e) Scale & Concurrency

- Increase `MAX_SYNC_WORKERS`, `MAX_CHECK_WORKERS`, and adjust pod replicas as your workload grows.
- Start with conservative values and monitor pod logs and node resource usage.
- See [Airbyte docs: Scaling](https://docs.airbyte.com/platform/operator-guides/scaling-airbyte).

---

**References:**
- [Airbyte Helm Chart Docs](https://docs.airbyte.com/platform/deploying-airbyte)
- [Airbyte External Database](https://docs.airbyte.com/platform/deploying-airbyte/integrations/database)
- [Airbyte External Blob Storage](https://docs.airbyte.com/platform/deploying-airbyte/integrations/storage)
- [Airbyte Metrics](https://docs.airbyte.com/platform/operator-guides/collecting-metrics)

## References

- [Airbyte Helm Chart Docs](https://docs.airbyte.com/platform/deploying-airbyte)
- [MicroK8s Documentation](https://microk8s.io/docs)