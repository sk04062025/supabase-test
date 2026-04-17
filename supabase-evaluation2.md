This updated document provides a comprehensive guide for setting up and evaluating a self-hosted Supabase instance on a Linux Virtual Machine using Docker Compose. It now features **Python** for the evaluation sample and a detailed comparison between **Self-Hosted** and **Cloud** versions.

---

# Evaluation & Setup Guide: Self-Hosted Supabase

## 1. Deployment Environment Details
To ensure a stable evaluation environment, the following specifications are recommended for the Linux VM.

*   **Operating System:** Ubuntu 22.04 LTS or higher.
*   **CPU:** 2 Cores (Minimum), 4 Cores (Recommended).
*   **RAM:** 4 GB (Minimum), 8 GB+ (Recommended).
*   **Disk:** 50 GB SSD (Minimum).
*   **Software Dependencies:**
    *   Docker Engine (v20.10+)
    *   Docker Compose (v2.0+)
    *   Python 3.9+ (for evaluation)
    *   Git

---

## 2. Setup Guide
Follow these steps to deploy Supabase using the official Docker Compose configuration.

### Step 1: Clone the Repository
```bash
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
```

### Step 2: Configure Environment Variables
```bash
cp .env.example .env
```
**Required Changes:** Open `.env` and update `POSTGRES_PASSWORD`, `JWT_SECRET`, `ANON_KEY`, and `SERVICE_ROLE_KEY`. *Do not use the default keys for any public-facing evaluation.*

### Step 3: Launch Containers
```bash
docker compose pull
docker compose up -d
```

### Step 4: Verify Deployment
```bash
docker compose ps
```
Ensure all services (Kong, GoTrue, PostgREST, Realtime, etc.) show a status of `Up`.

---

## 3. Access Details
The **Kong API Gateway** acts as the entry point for all services on port `8000`.

| Service | Protocol/Port | Access URL (Local/VM IP) |
| :--- | :--- | :--- |
| **Supabase Studio (UI)** | HTTP / 8000 | `http://<VM_IP>:8000` |
| **REST API** | HTTP / 8000 | `http://<VM_IP>:8000/rest/v1/` |
| **Auth Service** | HTTP / 8000 | `http://<VM_IP>:8000/auth/v1/` |
| **PostgreSQL** | TCP / 54322 | `postgresql://postgres:<PW>@<VM_IP>:54322` |

---

## 4. Usage Sample (Python)
To evaluate the connection, use the official Supabase Python client.

### Installation
```bash
pip install supabase
```

### Evaluation Script
Create a file named `eval_supabase.py`:
```python
import os
from supabase import create_client, Client

url: str = "http://<VM_IP>:8000"
key: str = "your-anon-key"

supabase: Client = create_client(url, key)

def run_evaluation():
    try:
        # 1. Insert Data
        data, count = supabase.table("todos").insert({"task": "Evaluate Self-hosted Supabase"}).execute()
        print(f"Insert Success: {data}")

        # 2. Query Data
        response = supabase.table("todos").select("*").execute()
        print(f"Query Results: {response.data}")

    except Exception as e:
        print(f"Evaluation Failed: {e}")

if __name__ == "__main__":
    run_evaluation()
```

---

## 5. Benefits of Self-Hosting Supabase
*   **Data Locality:** Keeps sensitive data within your own infrastructure or specific geographic region.
*   **No Tier Limits:** You are not restricted by the "Free Tier" limits (e.g., 500MB DB size) or "Pro Tier" costs of the Cloud version.
*   **Direct DB Access:** Full `superuser` access to the underlying PostgreSQL instance for complex migrations and configuration.
*   **Cost Control:** Fixed infrastructure costs regardless of the number of API calls or Auth users.

---

## 6. Limitations & Architectural Challenges

### 1. Single Postgres Instance Mapping
*   **The Limitation:** A self-hosted Supabase deployment is architecturally hard-coded to connect to **exactly one** PostgreSQL database instance.
*   **The Challenge:** In the Cloud version, one dashboard manages multiple projects. In self-hosted, if you need a second isolated project/database, you must deploy an entirely new stack (Kong, Auth, PostgREST, etc.), which doubles the resource consumption.

### 2. Multi-Instance Orchestration & Global Auth
*   **Kong Conflicts:** Each instance includes its own Kong gateway. If deploying via Kubernetes or multiple VMs, routing traffic becomes complex as each Kong expects to be the primary entry point.
*   **Global Auth Challenge:** There is no "Global Identity" layer. If you deploy Instance A and Instance B, a user registered on A cannot log into B because the `auth.users` table and `JWT_SECRET` are local to each deployment. Synchronizing these across an orchestrated cluster requires custom engineering.

### 3. Comparison: Self-Hosted vs. Supabase Cloud

| Feature | Self-Hosted (Docker) | Supabase Cloud |
| :--- | :--- | :--- |
| **Setup & Maintenance** | Manual (You handle updates/OS) | Zero-config (Serverless/Managed) |
| **Edge Functions** | Requires manual `edge-runtime` | Integrated (Global CDN) |
| **Log Management** | Basic Docker logs (Manual ELK/Grafana) | Integrated Log Explorer & Charts |
| **Auth Providers** | Manual config in `.env` | One-click Social Auth (Google, etc.) |
| **Backups** | Manual (pg_dump / WAL-G) | Automated Daily Backups |
| **Point-in-Time Recovery** | Not included | Available on Paid Tiers |
| **Scaling** | Vertical (Resize VM) | Horizontal & Seamless |
| **Support** | Community / Self-debug | Enterprise Support Available |

### 4. Additional Self-Hosted Risks
*   **Security Responsibility:** You must manually manage SSL/TLS (via Nginx/Certbot). The Docker setup defaults to HTTP.
*   **Observability:** The Self-hosted Studio lacks the "Observability" and "Report" tabs found in the Cloud version, making it harder to track slow queries or performance bottlenecks without external tools.
*   **Realtime Scaling:** The Realtime (Elixir) cluster configuration in Docker Compose is simplified; scaling it to handle 100k+ concurrent connections requires significant manual tuning compared to the Cloud's auto-scaling.
