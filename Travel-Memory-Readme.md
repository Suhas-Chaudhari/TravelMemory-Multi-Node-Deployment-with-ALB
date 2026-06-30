# TravelMemory — Multi-Node Deployment with Application Load Balancer

**Project date:** 29 June 2026
**Stack:** AWS VPC, EC2 (Ubuntu 26.04, t3.micro), Node.js 22, MongoDB Atlas, Application Load Balancer

## Overview

This project deploys the [TravelMemory](https://github.com/UnpredictablePrashant/TravelMemory) full-stack application (Node.js/Express backend + React frontend) across two EC2 instances in a custom VPC, fronted by an AWS Application Load Balancer for high availability. Application data is stored in MongoDB Atlas.

## Architecture

Internet traffic on port 80 reaches the ALB, which forwards to a target group on port 3000 spanning two EC2 instances placed in separate Availability Zones inside the public subnet. Both EC2 instances connect outbound to MongoDB Atlas for data persistence.

```
Internet (port 80)
      |
      v
travelmemory-alb (Internet-facing)
  Listener: HTTP :80
  Target group: travelmemory-tg :3000
      |
      +----------------------+----------------------+
      v                                              v
  app1 (EC2, us-east-1a)                    app2 (EC2, us-east-1b)
  34.232.50.157 / 10.0.12.187                52.91.115.85 / 10.0.29.200
  Backend :3001  |  Frontend :3000           Backend :3001  |  Frontend :3000
      |                                              |
      +----------------------+----------------------+
                             v
                     MongoDB Atlas
                cluster0.hfeojne.mongodb.net
```

## 1. VPC and Networking Setup

A custom VPC (`travl-mem-vpc-vpc`) was created spanning two Availability Zones for redundancy.

**Resources created:**

| Resource | Count | Detail |
|---|---|---|
| VPC | 1 | `travl-mem-vpc-vpc` |
| Subnets | 4 | 2 public + 2 private, across `us-east-1a` and `us-east-1b` |
| Route tables | 4 | Including `travl-mem-vpc-rtb-public` with IGW route |
| Internet Gateway | 1 | `travl-mem-vpc-igw` |
| VPC endpoint | 1 | `travl-mem-vpc-vpce-s3` |

**Subnets used for this deployment:**

| Subnet name | AZ | CIDR | Type |
|---|---|---|---|
| travl-mem-vpc-subnet-public1-us-east-1a | us-east-1a | 10.0.0.0/20 | Public |
| travl-mem-vpc-subnet-public2-us-east-1b | us-east-1b | 10.0.16.0/20 | Public |

Both public subnets have Internet Gateway routes and ALLOW rules on inbound/outbound Network ACLs, satisfying ALB subnet requirements (minimum 8 available IPs, IGW-routed).

## 2. Security Group Configuration

**Security group:** `travl-mem-sg` (`sg-0d17f8c3465821ea2`)

| Type | Protocol | Port range | Source |
|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| Custom TCP | TCP | 3000 | 0.0.0.0/0 |
| Custom TCP | TCP | 3001 | 0.0.0.0/0 |

> **Hardening note:** once the ALB is live and healthy, the recommended next step is to restrict ports 3000/3001 on this SG to only accept traffic from the ALB's security group rather than `0.0.0.0/0`, so the app is reachable only through the load balancer.

## 3. EC2 Instance Provisioning

Two identical EC2 instances were launched in the VPC above, one per AZ.

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 26.04 LTS (HVM), SSD volume |
| Instance type | t3.micro (free tier eligible) |
| Key pair | trvl-mem-tst |
| Storage | 8 GiB gp3 |
| Security group | travl-mem-sg |

| Instance | Name | Subnet / AZ | Private IP | Public IP |
|---|---|---|---|---|
| app1 | app1 | public1-us-east-1a | 10.0.12.187 | 34.232.50.157 |
| app2 | app2 | public2-us-east-1b | 10.0.29.200 | 52.91.115.85 |

SSH connectivity to both instances was verified successfully post-launch.

## 4. Server Setup — Node.js Installation

On **both** instances:

```bash
mkdir app1   # (app2 on the second instance)
cd app1

sudo apt update
sudo apt upgrade -y

curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

node -v   # v22.23.1
npm -v    # 10.9.8
```

## 5. Application Deployment

### Clone the repository (both instances)

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory
```

### Backend configuration

Create `backend/.env`:

```env
MONGO_URI='mongodb+srv://username:password@*******.hfeojne.mongodb.net/********?retryWrites=true&w=majority'
PORT=3001
```

Install dependencies and start:

```bash
npm install
nohup node index.js &
```

Verified working on both nodes:

```bash
curl http://34.232.50.157:3001/hello   # app1 → Hello World!
curl http://52.91.115.85:3001/hello    # app2 → Hello World!
```

### Frontend configuration

Create `frontend/.env`, pointing at the backend on the **same** instance:

```bash
# app1
echo "REACT_APP_BACKEND_URL=http://34.232.50.157:3001" > .env

# app2
echo "REACT_APP_BACKEND_URL=http://52.91.115.85:3001" > .env
```

Install and run:

```bash
npm install
npm start
```

Both frontends confirmed loading the TravelMemory UI on port 3000, pulling live data from MongoDB Atlas (e.g. "Incredible India" experience cards visible on both nodes).

## 6. Application Load Balancer Setup

### Step 1 — Confirm both backends are reachable (done above)

### Step 2 — Create target group

| Field | Value |
|---|---|
| Target type | Instances |
| Name | travelmemory-tg |
| Protocol / Port | HTTP / 3000 |
| VPC | travl-mem-vpc-vpc |
| Health check protocol | HTTP |
| Health check path | `/` |
| Healthy threshold | 2 |
| Interval | 30 seconds |

Both `app1` and `app2` instances registered as targets.

### Step 3 — Create the Application Load Balancer

| Field | Value |
|---|---|
| Name | travelmemory-alb |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | travl-mem-vpc-vpc |
| Availability Zones | us-east-1a + us-east-1b (both public subnets) |
| Security group | travl-mem-sg |
| Listener | HTTP : 80 → forward to travelmemory-tg |

### Step 4 — Verify target health

`EC2 → Target Groups → travelmemory-tg → Targets` — both `app1` and `app2` reported **Healthy**.

## 7. Result

| Item | Value |
|---|---|
| ALB DNS name | `travelmemory-alb-1556311436.us-east-1.elb.amazonaws.com` |
| Test URL | http://travelmemory-alb-1556311436.us-east-1.elb.amazonaws.com |
| Status | Working — TravelMemory UI loads and serves data from MongoDB Atlas via either backend node |

## 8. Remaining will do later / Optional Steps for this excercise

- [ ] **Lock down EC2 security group**: restrict inbound on ports 3000/3001 to the ALB's security group only, instead of `0.0.0.0/0`.
- [ ] **Custom domain (optional)**: create a CNAME record in Route 53 (or external DNS provider) pointing to the ALB DNS name. Do not use an A record, since ALB IPs are not static.
- [ ] **Process manager**: replace `nohup` with a supervisor such as `pm2` or a systemd service so the Node processes survive instance reboot and crash automatically.
- [ ] **HTTPS**: add an ACM certificate and an HTTPS:443 listener on the ALB, with an HTTP→HTTPS redirect rule.
