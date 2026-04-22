Using **CloudNativePG (CNPG)** is the industry-standard "best practice" for running production-grade PostgreSQL in Kubernetes.

Unlike a simple Helm chart, CNPG is a **Kubernetes Operator**. It acts as an intelligent controller that understands PostgreSQL's internals (replication, failover, backup, and recovery) and manages them declaratively.

### The Best Way to Architect This
You **do not** need to create separate manifests for a "primary" and "read-only replicas." In CloudNativePG, you define a single `Cluster` resource with **3 instances**.

CNPG will automatically:
1.  Elect one instance as the **Primary** (for read/write).
2.  Configure the other two instances as **Hot Standby Replicas** (for read-only).
3.  Manage streaming replication between them.
4.  Handle automatic failover if the primary crashes.

---

### Step 1: Install the CNPG Operator
First, install the operator itself into your cluster. You can do this using `kubectl` by applying the official manifest.

```bash
# Apply the latest CNPG operator manifest
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.29/releases/cnpg-1.29.0.yaml
```

*Verify the installation:*
```bash
kubectl get pods -n cnpg-system
```

---

### Step 2: Define and Apply the PostgreSQL Cluster
Create a file named `postgres-cluster.yaml`. This file defines your requirement: 3 instances (1 primary + 2 read-only).

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-app-db
spec:
  # This creates 1 Primary + 2 Read-Only Replicas
  instances: 3 

  # Storage configuration
  storage:
    size: 20Gi
    # storageClass: "standard" # Uncomment and set if you have a specific storage class

  # Bootstrap: This creates the database and owner user automatically
  bootstrap:
    initdb:
      database: myapp
      owner: myuser
      secret:
        name: my-app-db-creds

  # Optional: Resource requests/limits for stability
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "500m"
```

**Before applying**, create the secret for your credentials:
```bash
kubectl create secret generic my-app-db-creds \
  --from-literal=username=myuser \
  --from-literal=password=supersecretpassword
```

**Now, apply the cluster:**
```bash
kubectl apply -f postgres-cluster.yaml
```

---

### Step 3: How to Connect (The "CNPG Way")

CNPG automatically creates two services for your application to use. You should **never** hardcode IP addresses or pod names.

1.  **For Read/Write traffic (Primary):**
    Use the service named `<cluster-name>-rw`.
    *   **Host:** `my-app-db-rw`
    *   *Example:* `postgres://myuser:password@my-app-db-rw:5432/myapp`

2.  **For Read-Only traffic (Replicas):**
    Use the service named `<cluster-name>-ro`. CNPG automatically load-balances your `SELECT` queries across both of your read-only replicas.
    *   **Host:** `my-app-db-ro`
    *   *Example:* `postgres://myuser:password@my-app-db-ro:5432/myapp`

---

### Why this is the "Best" way:
*   **Self-Healing:** If your Primary pod dies, the CNPG operator will detect it within seconds, promote one of the two replicas to be the new Primary, and update the `-rw` service automatically.
*   **Zero-Downtime:** Scaling is as simple as changing `instances: 3` to `instances: 5` in your YAML and applying it (`kubectl apply`). The operator will add the new replicas without interrupting the primary.
*   **Kubernetes Native:** It treats the database like any other Kubernetes object. You manage it with `kubectl`, monitor it with standard tools, and integrate it into your CI/CD pipelines as GitOps code.
