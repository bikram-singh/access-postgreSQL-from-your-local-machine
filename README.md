<div align="center">


# 🐘 Access-PostgreSQL-from-Your-Local-Machine

### Cloud SQL · PostgreSQL 17 · IAP Tunnel · Bastion VM · pgAdmin 4
### DHG Rate Automation Platform - `dhg-vaccine-rateauto-nonpord`

[![Cloud SQL](https://img.shields.io/badge/Cloud_SQL-PostgreSQL_17-4285F4?logo=google-cloud&logoColor=white)](https://cloud.google.com/sql)
[![IAP](https://img.shields.io/badge/Google-Identity_Aware_Proxy-FF6D00?logo=google-cloud&logoColor=white)](https://cloud.google.com/iap)
[![pgAdmin](https://img.shields.io/badge/pgAdmin-4-336791?logo=postgresql&logoColor=white)](https://www.pgadmin.org/)
[![Windows](https://img.shields.io/badge/Platform-Windows-0078D6?logo=windows&logoColor=white)](https://www.microsoft.com/windows)
[![PSC](https://img.shields.io/badge/Connectivity-Private_Service_Connect-34A853?logo=google&logoColor=white)](https://cloud.google.com/vpc/docs/private-service-connect)
[![No VPN](https://img.shields.io/badge/VPN-Not_Required-EA4335?logo=googlechrome&logoColor=white)]()

---

*Connect to a private Cloud SQL PostgreSQL 17 instance (no public IP) from your local Windows machine using a Bastion VM, Google IAP TCP tunneling, and pgAdmin 4 - no VPN required.*

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Instance Details](#-instance-details)
- [How This Works](#-how-this-works)
- [Prerequisites](#-prerequisites)
- [Phase 1 - Find the PSC Endpoint IP](#-phase-1---find-the-psc-endpoint-ip)
- [Phase 2 - Grant IAM Permissions](#-phase-2---grant-iam-permissions)
- [Phase 3 - Create the IAP Firewall Rule](#-phase-3---create-the-iap-firewall-rule)
- [Phase 4 - Verify SSH Key on Bastion](#-phase-4---verify-ssh-key-on-bastion)
- [Phase 5 - Open the IAP Tunnel](#-phase-5---open-the-iap-tunnel)
- [Phase 6 - Verify Tunnel is Active](#-phase-6---verify-tunnel-is-active)
- [Phase 7 - Connect via pgAdmin 4](#-phase-7---connect-via-pgadmin-4)
- [Daily Workflow](#-daily-workflow)
- [Windows-Specific Notes](#-windows-specific-notes)
- [IAM Reference](#-iam-reference)
- [Related Repositories](#-related-repositories)

---

## 🌐 Overview

To access the DHG Rate Automation Cloud SQL PostgreSQL 17 instance (`dhg-rateauto-postgres-dev`) from a local Windows machine using Google IAP (Identity-Aware Proxy) tunneling via a Bastion VM.

> ✅ **Status: VERIFIED WORKING**

### 🔑 Key Facts

| Property | Value |
|---|---|
| ☁️ **GCP Project** | `dhg-vaccine-rateauto-nonpord` |
| 🐘 **Instance Name** | `dhg-rateauto-postgres-dev` |
| 📦 **PostgreSQL Version** | `17` |
| 🔌 **Connectivity Type** | Private Service Connect (PSC) - No Public IP |
| 🌍 **Region** | `us-central1` |
| 🖥️ **Bastion VM** | `postgres-bastion` (`us-central1-a`) |
| 🔒 **Bastion IP** | `10.10.0.22` (private only) |
| 🗄️ **PSC Endpoint IP** | `10.10.0.3` |
| 🔌 **DB Port** | `5432` |
| 🔁 **Local Tunnel Port** | `5433` |
| 🚫 **VPN Required** | No |
| 🔐 **Auth Method** | Google IAP + SSH key |

---

## 🏛️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Your Local Machine                              │
│                     pgAdmin 4 → localhost:5433                          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ HTTPS / TCP 443
                               │ (no VPN required)
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   Google IAP (Identity-Aware Proxy)                      │
│              Authenticates your GCP identity before forwarding           │
│                    Source range: 35.235.240.0/20                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ Authenticated SSH tunnel (port 22)
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Bastion VM: postgres-bastion                         │
│              Zone: us-central1-a  │  Private IP: 10.10.0.22             │
│              Network: dhg-rateauto-dev-vpc                               │
│              Subnet: dhg-rateauto-dev-us-subnet                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ Internal VPC traffic (port 5432)
                               │ via PSC endpoint
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Cloud SQL PostgreSQL 17: dhg-rateauto-postgres-dev          │
│              PSC Endpoint IP: 10.10.0.3  │  Port: 5432                  │
│              Private Service Connect - No Public IP                      │
│              Databases: cloudsqladmin │ dhg-db │ postgres                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key advantage:** IAP tunnels exclusively over HTTPS port 443. No VPN connection, no public IP on the bastion, no firewall hole for port 22 from the internet.

---

## 📋 Instance Details

### Cloud SQL Instance

| Field | Value |
|---|---|
| **Instance Name** | `dhg-rateauto-postgres-dev` |
| **Connection Name** | `dhg-vaccine-rateauto-nonpord:us-central1:dhg-rateauto-postgres-dev` |
| **PostgreSQL Version** | `17` |
| **Region** | `us-central1` |
| **Public IP** | Disabled |
| **Private IP** | None (PSC only) |
| **PSC Endpoint IP** | `10.10.0.3` |
| **PSC DNS Name** | `2a4bbffc1e59.1jb6fma7xqy7l.us-central1.sql.goog.` |
| **Default Port** | `5432` |
| **SSL Mode** | Allow SSL (not enforced) |
| **Databases** | `cloudsqladmin`, `dhg-db`, `postgres` |

### Bastion VM

| Field | Value |
|---|---|
| **VM Name** | `postgres-bastion` |
| **Zone** | `us-central1-a` |
| **Private IP** | `10.10.0.22` |
| **Network** | `dhg-rateauto-dev-vpc` |
| **Subnet** | `dhg-rateauto-dev-us-subnet` |
| **OS Login Username** | `bikram` |

---

## 💡 How This Works

### Why PSC Has No Private IP

This instance uses **Private Service Connect (PSC)** - not standard VPC peering. With PSC, Cloud SQL does not get a directly assigned private IP address. Instead, a **PSC endpoint** is created inside your VPC that forwards traffic to the managed Cloud SQL service.

```
Standard Private IP (PSA):   VPC ──peering──► Google Managed VPC ──► Cloud SQL
Private Service Connect:     VPC ──endpoint (10.10.0.3)──► Service Attachment ──► Cloud SQL
```

The PSC endpoint IP (`10.10.0.3`) lives inside your VPC and is reachable by any VM in the same VPC - including the bastion.

### Why IAP Instead of Direct SSH

The bastion VM has no public IP. IAP acts as a **zero-trust proxy** - it verifies your GCP identity and forwards the connection through Google's infrastructure without exposing port 22 to the internet.

```
Traditional SSH:   Your machine ──(internet:22)──► Bastion (needs public IP)
IAP Tunnel:        Your machine ──(HTTPS:443)──► Google IAP ──► Bastion (private IP only)
```

---

## ✅ Prerequisites

| # | Requirement | Details |
|---|---|---|
| 1 | **gcloud CLI** installed locally | [Install guide](https://cloud.google.com/sdk/docs/install) |
| 2 | **pgAdmin 4** installed locally | [Download](https://www.pgadmin.org/download/) |
| 3 | **GCP account** with project access | `bikram@gcpcloudhub.shop` |
| 4 | **IAM roles** assigned | See [Phase 2](#-phase-2---grant-iam-permissions) |
| 5 | **IAP firewall rule** in place | See [Phase 3](#-phase-3---create-the-iap-firewall-rule) |
| 6 | **VS Code** (recommended terminal) | [Download](https://code.visualstudio.com/) |

### Check gcloud is Installed

Open PowerShell and run:

```powershell
gcloud version
```

Expected output:
```
Google Cloud SDK 500.x.x
bq 2.x.x
core 2024.xx.xx
```

If not installed, download from `https://cloud.google.com/sdk/docs/install`, run the installer accepting all defaults, then reopen PowerShell and verify again.

### Authenticate with GCP

```powershell
gcloud auth login
gcloud config set project dhg-vaccine-rateauto-nonpord
```

Verify:
```powershell
gcloud config list
```

Expected output:
```
[core]
account = bikram@gcpcloudhub.shop
project = dhg-vaccine-rateauto-nonpord
```

---

## 🔍 Phase 1 - Find the PSC Endpoint IP

> **One-time step.** The PSC endpoint IP is fixed once provisioned.

The Cloud SQL instance has no traditional private IP. Run this in **Cloud Shell** to find the PSC endpoint:

```bash
gcloud compute addresses list \
  --project=dhg-vaccine-rateauto-nonpord \
  --filter="purpose=GCE_ENDPOINT OR purpose=PRIVATE_SERVICE_CONNECT" \
  --format="table(name, address, purpose, status)"
```

Expected output:
```
NAME                              ADDRESS    PURPOSE      STATUS
dhg-rateauto-postgres-dev-psc-ip  10.10.0.3  GCE_ENDPOINT  IN_USE
```

> ✅ **PSC Endpoint IP: `10.10.0.3`** - use this in all tunnel commands.

---

## 🔐 Phase 2 - Grant IAM Permissions

> **One-time step.** Run in **Cloud Shell** as a project owner.

Three permissions are required for IAP tunneling to work:

### Step 2.1 - Grant IAP Tunnel Access

```bash
gcloud projects add-iam-policy-binding dhg-vaccine-rateauto-nonpord \
  --member="user:bikram@gcpcloudhub.shop" \
  --role="roles/iap.tunnelResourceAccessor"
```

### Step 2.2 - Grant Compute OS Login

```bash
gcloud projects add-iam-policy-binding dhg-vaccine-rateauto-nonpord \
  --member="user:bikram@gcpcloudhub.shop" \
  --role="roles/compute.osLogin"
```

### Step 2.3 - Verify Roles Are Assigned

```bash
gcloud projects get-iam-policy dhg-vaccine-rateauto-nonpord \
  --flatten="bindings[].members" \
  --filter="bindings.members:bikram@gcpcloudhub.shop" \
  --format="table(bindings.role)"
```

Expected output must include:
```
ROLE
roles/iap.tunnelResourceAccessor
roles/compute.osLogin
roles/owner
```

---

## 🔥 Phase 3 - Create the IAP Firewall Rule

> **One-time step.** Run in **Cloud Shell**.

IAP requires an ingress firewall rule allowing traffic from Google's IAP IP range (`35.235.240.0/20`) on port 22 to reach the bastion VM.

```bash
gcloud compute firewall-rules create allow-iap-ingress \
  --project=dhg-vaccine-rateauto-nonpord \
  --network=dhg-rateauto-dev-vpc \
  --direction=INGRESS \
  --source-ranges=35.235.240.0/20 \
  --allow=tcp:22,tcp:5432 \
  --description="Allow IAP TCP tunneling to bastion and postgres"
```

Verify the rule was created:

```bash
gcloud compute firewall-rules describe allow-iap-ingress \
  --project=dhg-vaccine-rateauto-nonpord
```

Expected:
```
NAME: allow-iap-ingress
NETWORK: dhg-rateauto-dev-vpc
DIRECTION: INGRESS
ALLOW: tcp:22,tcp:5432
DISABLED: False
```

---

## 🔑 Phase 4 - Verify SSH Key on Bastion

> **One-time step.** Run in **Cloud Shell**.

SSH keys must be generated and propagated to the bastion VM before the tunnel will work.

```bash
gcloud compute ssh postgres-bastion \
  --zone=us-central1-a \
  --project=dhg-vaccine-rateauto-nonpord \
  --command="whoami"
```

Expected output:
```
bikram
```

If prompted to generate SSH keys, type `Y` and press Enter. This is a one-time step - keys are saved to `~/.ssh/google_compute_engine` on the machine where this command is run.

> ✅ Once `whoami` returns `bikram` the bastion is reachable and keys are propagated.

---

## 🚇 Phase 5 - Open the IAP Tunnel

> **Required every session.** Run on your **local machine** in VS Code terminal (PowerShell).

Open VS Code, press **Ctrl + `** to open the terminal, ensure it shows **PowerShell** in the top-right dropdown, then run:

```powershell
gcloud compute ssh postgres-bastion --project=dhg-vaccine-rateauto-nonpord --zone=us-central1-a --tunnel-through-iap --ssh-flag="-L 5433:10.10.0.3:5432" --ssh-flag="-N"
```

### What the Command Does

| Part | Meaning |
|---|---|
| `--tunnel-through-iap` | Routes SSH over Google IAP (HTTPS/443) - no VPN needed |
| `--ssh-flag="-L 5433:10.10.0.3:5432"` | Forwards local port `5433` → bastion → DB at `10.10.0.3:5432` |
| `--ssh-flag="-N"` | Do not execute a remote command - tunnel only |

### What to Expect After Running

- A **PuTTY window** will open showing:
  ```
  Using username "bikram".
  Authenticating with public key "OPTIPLEX-PC\user@OptiPlex-PC"
  ```
- The **VS Code terminal** will appear to hang with no output
- **Both the PuTTY window AND the VS Code terminal must remain open** for the entire database session

> ⚠️ Do NOT close the PuTTY window. Do NOT press Ctrl+C in the terminal. Either action will drop the tunnel and disconnect pgAdmin.

<img width="940" height="289" alt="image" src="https://github.com/user-attachments/assets/2ba5e6ad-f737-42bb-9a39-c902cf6c2782" />


---

## ✔️ Phase 6 - Verify Tunnel is Active

Open a **second terminal** in VS Code by clicking the `+` icon in the terminal panel, then run:

```powershell
Test-NetConnection -ComputerName localhost -Port 5433
```

Expected output:
```
ComputerName     : localhost
RemoteAddress    : ::1
RemotePort       : 5433
InterfaceAlias   : Loopback Pseudo-Interface 1
SourceAddress    : ::1
TcpTestSucceeded : True
```

> ✅ `TcpTestSucceeded : True` confirms the tunnel is live and pgAdmin can connect.

If `TcpTestSucceeded : False`, the tunnel is not running - go back to [Phase 5](#-phase-5---open-the-iap-tunnel) and re-run the tunnel command.

<img width="940" height="342" alt="image" src="https://github.com/user-attachments/assets/de6c26de-352f-4080-a2b3-504aceda8b23" />


---

## 🐘 Phase 7 - Connect via pgAdmin 4

> With the tunnel running, open pgAdmin 4.

### Step 7.1 - Register a New Server

Right-click **Servers** in the left panel → **Register** → **Server...**

### Step 7.2 - General Tab

| Field | Value |
|---|---|
| **Name** | `DHG-dev-postgres` |

### Step 7.3 - Connection Tab

| Field | Value |
|---|---|
| **Host name/address** | `localhost` |
| **Port** | `5433` |
| **Maintenance database** | `postgres` |
| **Username** | `dhg-user` |
| **Password** | *(your DB password)* |
| **Save password?** | Yes |

> ⚠️ **Critical:** pgAdmin defaults the port to `5432`. You MUST manually change it to `5433`. If you see a connection timeout error mentioning port `5432`, this is the cause - edit server properties and change the port.

### Step 7.4 - SSL Tab

| Field | Value |
|---|---|
| **SSL mode** | `Require` |

### Step 7.5 - Click Save

pgAdmin connects through the tunnel. You should see the server expand in the left panel showing:

```
DHG-dev-postgres
└── Databases (3)
    ├── cloudsqladmin
    ├── dhg-db
    └── postgres
```

> ✅ **Connected.** You are now querying the Cloud SQL PostgreSQL 17 instance via IAP tunnel.

---

## 📅 Daily Workflow

Each day when you need database access, follow these steps in order:

**1.** Open VS Code → open a new PowerShell terminal (`Ctrl + '`)

**2.** Run the tunnel command and keep this terminal open:

```powershell
gcloud compute ssh postgres-bastion --project=dhg-vaccine-rateauto-nonpord --zone=us-central1-a --tunnel-through-iap --ssh-flag="-L 5433:10.10.0.3:5432" --ssh-flag="-N"
```

**3.** A PuTTY window opens and authenticates - leave it open

**4.** Open a second terminal and verify the tunnel:

```powershell
Test-NetConnection -ComputerName localhost -Port 5433
```

**5.** Open pgAdmin 4 → connect to `DHG-dev-postgres` (localhost:`5433`)

**6.** Work with the database

**7.** When done: close the PuTTY window OR press `Ctrl+C` in the tunnel terminal

---

## 🖥️ Windows-Specific Notes

| Observation | Explanation |
|---|---|
| PuTTY window opens when running the tunnel command | Normal - Windows `gcloud` uses PuTTY (plink) instead of OpenSSH. Leave it open. |
| `-q` flag causes `plink: unknown option "-q"` error | PuTTY does not support the `-q` quiet flag. Do not use it. |
| `--` separator causes `unrecognized arguments` error | PowerShell does not pass `--` correctly to `gcloud`. Use `--ssh-flag` syntax instead. |
| `gcloud components update` fails with permission error | Not required. Skip it. The existing `gcloud` version works fine. |
| SSH key generation prompt on first run | Type `Y` - one-time only. Keys are saved to `C:\Users\USERNAME\.ssh\`. |
| pgAdmin shows connection timeout on port `5432` | pgAdmin defaults to 5432. Always manually set port to `5433` in Connection tab. |
| `set GCLOUD_SSH_CLIENT=openssh` in CMD has no effect | The `set` command only applies to that CMD session and gcloud may ignore it. Use `--tunnel-through-iap --ssh-flag` approach instead - it works reliably. |

---

## 🔐 IAM Reference

### Roles Required for the Connecting User

| Role | Level | Purpose |
|---|---|---|
| `roles/iap.tunnelResourceAccessor` | Project | Allows IAP TCP tunneling to VMs |
| `roles/compute.osLogin` | Project | Allows SSH login via OS Login |

### Firewall Rule Required

| Field | Value |
|---|---|
| **Name** | `allow-iap-ingress` |
| **Network** | `dhg-rateauto-dev-vpc` |
| **Direction** | Ingress |
| **Source IP ranges** | `35.235.240.0/20` |
| **Protocol/Port** | `tcp:22`, `tcp:5432` |
| **Target** | All instances in network |

### Grant Commands (for GCP Admin)

```bash
# IAP tunnel access
gcloud projects add-iam-policy-binding dhg-vaccine-rateauto-nonpord \
  --member="user:bikram@gcpcloudhub.shop" \
  --role="roles/iap.tunnelResourceAccessor"

# OS Login
gcloud projects add-iam-policy-binding dhg-vaccine-rateauto-nonpord \
  --member="user:bikram@gcpcloudhub.shop" \
  --role="roles/compute.osLogin"

# IAP firewall rule
gcloud compute firewall-rules create allow-iap-ingress \
  --project=dhg-vaccine-rateauto-nonpord \
  --network=dhg-rateauto-dev-vpc \
  --direction=INGRESS \
  --source-ranges=35.235.240.0/20 \
  --allow=tcp:22,tcp:5432 \
  --description="Allow IAP TCP tunneling to bastion and postgres"
```

---

## 🔗 Related Repositories

| Repository | Purpose |
|---|---|
| [`Access-PostgreSQL-from-Your-Local-Machine`](https://github.com/bikram-singh/Access-PostgreSQL-from-Your-Local-Machine) | **This repo** - Local DB access runbook |

---

<div align="center">

**Maintained by Bikram Singh**
`dhg-vaccine-rateauto-nonpord` · `us-central1` · Cloud SQL PostgreSQL 17

</div>
