This document provides a comprehensive guide for setting up and evaluating a self-hosted Supabase instance on a Linux Virtual Machine using Docker Compose.

---

# Evaluation & Setup Guide: Self-Hosted Supabase

## 1. Deployment Environment Details
To ensure a stable production or evaluation environment, the following specifications are recommended for the Linux VM.

*   **Operating System:** Ubuntu 22.04 LTS or higher (Recommended).
*   **CPU:** 2 Cores (Minimum), 4 Cores (Recommended).
*   **RAM:** 4 GB (Minimum), 8 GB+ (Recommended for production).
*   **Disk:** 50 GB SSD (Minimum).
*   **Software Dependencies:**
    *   Docker Engine (v20.10+)
    *   Docker Compose (v2.0+)
    *   Git

---

## 2. Setup Guide
Follow these steps to deploy Supabase using the official Docker Compose configuration.

### Step 1: Clone the Repository
```bash
# Get the code
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
```

### Step 2: Configure Environment Variables
Copy the template and modify the required secrets.
```bash
cp .env.example .env
```
**Important:** Open `.env` and update the following variables immediately for security:
*   `POSTGRES_PASSWORD`: Your database root password.
*   `JWT_SECRET`: A long, random string for signing tokens.
*   `ANON_KEY` & `SERVICE_ROLE_KEY`: Generate these using the [Supabase JWT tool](https://jwt.io/) or a script.

### Step 3: Launch Containers
```bash
docker compose pull
docker compose up -d
```

### Step 4: Verify Deployment
Check that all containers (Postgres, GoTrue, PostgREST, Realtime, Storage, Kong, etc.) are running:
```bash
docker compose ps
```

---

## 3. Access Details
Once running, Supabase services are exposed through the **Kong API Gateway** on port `8000`.

| Service | Internal Port | External Access (via Kong) |
| :--- | :--- | :--- |
| **Supabase Studio** | 3000 | `http://<VM_IP>:8000` |
| **REST API** | 3000 | `http://<VM_IP>:8000/rest/v1/` |
| **Auth (GoTrue)** | 9999 | `http://<VM_IP>:8000/auth/v1/` |
| **PostgreSQL** | 5432 | `postgresql://postgres:<PASSWORD>@<VM_IP>:54322` |
| **Storage** | 7500 | `http://<VM_IP>:8000/storage/v1/` |

**Default Credentials (if not changed in .env):**
*   **Studio Login:** Default is often disabled or requires `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` in `.env`.
*   **Postgres User:** `postgres`

---

## 4. Usage Sample
Connecting a JavaScript frontend to your self-hosted instance:

```javascript
import { createClient } from '@supabase/supabase-js'

const SUPABASE_URL = 'http://<VM_IP>:8000'
const SUPABASE_ANON_KEY = 'your-anon-key'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// Example: Fetch data
async function getTodos() {
  const { data, error } = await supabase
    .from('todos')
    .select('*')
  
  if (error) console.error('Error:', error)
  else console.log('Data:', data)
}
```

---

## 5. Benefits of Self-Hosting Supabase
*   **Data Sovereignty:** You have 100% control over where your data resides, satisfying GDPR/HIPAA requirements.
*   **No Usage Limits:** Unlike the cloud version, there are no artificial limits on database size, API requests, or auth users (limited only by VM hardware).
*   **Cost Predictability:** You pay only for the VM cost, regardless of the number of projects or traffic volume.
*   **Network Locality:** If your application is also on-prem or in the same VPC, latency is significantly reduced.
*   **Superuser Access:** You get full `postgres` user access to the database, allowing for custom extensions and low-level tuning.

---

## 6. Limitations & Challenges

### 1. Single Postgres Instance Constraint
A single self-hosted Supabase deployment is architecturally bound to **one** PostgreSQL instance.
*   **The Problem:** Unlike the Cloud version where the dashboard manages multiple independent projects (each with its own DB), a Docker Compose setup creates one project for one DB.
*   **Workaround:** To manage multiple databases, you must deploy multiple independent stacks of Supabase (separate Docker Compose files/folders), which significantly increases resource overhead.

### 2. Global Authentication Challenge (Orchestration)
When scaling horizontally or running multiple Supabase instances behind an orchestrator:
*   **Kong Conflict:** Each Supabase instance comes with its own Kong Gateway. Managing a "Global Auth" service that works across different Supabase instances is difficult because each instance maintains its own `auth.users` table.
*   **The Challenge:** JWTs issued by Instance A will not be valid for Instance B unless you manually synchronize `JWT_SECRET` across all instances and share a centralized Auth database—a configuration not supported "out of the box."

### 3. Missing Cloud-Only Features
*   **Dashboard UI:** The self-hosted Studio lacks some advanced features like the "Log Explorer," "Analytics Dashboards," and "Observability" charts.
*   **Edge Functions:** Deploying Edge Functions self-hosted requires manual setup of the `edge-runtime` and does not provide the global low-latency CDN distribution found in Supabase Cloud.
*   **Automatic Backups:** You are responsible for configuring your own Postgres WAL-G or pg_dump backup strategy.

### 4. Maintenance & Security Overhead
*   **Updates:** Upgrading services (e.g., moving from Postgres 15 to 16) requires manual migration scripts and downtime.
*   **Security:** You must manage your own SSL certificates (e.g., via Nginx/Certbot) and firewall rules, as the default Docker setup is exposed via HTTP.
*   **Log Management:** In the self-hosted version, logs are sent to the console by default; setting up a persistent log drain (like Logflare) requires additional infrastructure.
