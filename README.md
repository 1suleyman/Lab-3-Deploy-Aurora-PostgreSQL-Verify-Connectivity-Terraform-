# 🐘 Lab 3 — Deploy Aurora PostgreSQL & Verify Connectivity (Terraform)

In this lab, I extended the existing Terraform-based EC2 + Docker PostgreSQL setup by **deploying an Aurora PostgreSQL cluster** and **verifying end-to-end connectivity from EC2**.

The focus of this lab was not data migration yet, but **infrastructure correctness**:

* Engine version compatibility
* Terraform debugging
* Network reachability
* Security group validation
* Authentication verification
* Positive and negative connectivity tests

This lab proves that **Aurora is reachable, correctly secured, and ready for migration**.

---

## 📋 Lab Overview

**Goal:**

* Deploy an Aurora PostgreSQL cluster using Terraform
* Resolve engine version compatibility issues
* Verify cluster availability and outputs
* Test network connectivity from EC2 to Aurora
* Validate PostgreSQL authentication
* Perform negative security testing
* Clean up infrastructure safely

**Learning Outcomes:**

* Understand Aurora PostgreSQL engine version constraints
* Debug Terraform variable vs provider mismatches
* Use Terraform outputs for operational workflows
* Verify VPC routing and security group rules
* Distinguish **network reachability** from **authentication**
* Apply positive and negative connectivity testing

---

## 🛠 Step-by-Step Journey

### Step 1: Initialize & Apply Terraform

**Commands:**

```bash
terraform init
terraform plan
terraform apply
```

* Terraform provisions:

  * EC2 instance (Docker + Postgres container already configured)
  * Aurora PostgreSQL cluster
  * Subnet groups
  * Security groups
  * KMS encryption
  * Outputs for connectivity

---

### Step 2: Debug Aurora Engine Version Errors

**Initial Error:**

```
Cannot find version 15.4 for Aurora PostgreSQL
```

Attempts with:

* `15.4` ❌
* `15.1` ❌

**Root Cause:**
Not all Aurora PostgreSQL engine versions are available in every region or account.

---

### Step 3: Identify Supported Engine Versions

**Command:**

```bash
aws rds describe-db-engine-versions \
  --engine aurora-postgresql \
  --query "DBEngineVersions[].EngineVersion" \
  --output table
```

**Result:**
Confirmed that **15.13** is supported in the region.

**Fix Applied:**
Updated `engine_version` to:

```hcl
engine_version = "15.13"
```

After re-applying, Terraform succeeded.

---

### Step 4: Confirm Successful Deployment

**Command:**

```bash
terraform output
```

Verified outputs included:

* Aurora cluster endpoint
* Reader endpoint
* Database name
* Subnet group
* KMS key ARN
* Security group IDs
* VPC ID
* EC2 public IP
* SSH command

Checked AWS Console:
✅ Aurora cluster status = **Available**

<img width="1109" height="145" alt="Screenshot 2026-01-20 at 13 40 12" src="https://github.com/user-attachments/assets/a1b8faff-f5db-4ed3-bf08-3b8a854f3efb" />

---

### Step 5: SSH into EC2 Using Terraform Output

Using the generated SSH command:

```bash
ssh -i labec2.pem ec2-user@<public-ip>
```

Successful SSH confirms:

* EC2 provisioning is correct
* Networking is functional

---

### Step 6: Install PostgreSQL Client on EC2

**Command:**

```bash
sudo dnf install -y postgresql15
```

**Verify:**

```bash
psql --version
```

Result:

```
psql (PostgreSQL) 15.x
```

---

### Step 7: Test Network Connectivity (TCP Layer)

#### 7.1 Attempt `nc` (Initial Failure)

```bash
nc -vz <aurora-endpoint> 5432
```

Result:

```
nc: command not found
```

This confirms:
✅ Aurora is **not the problem**

---

#### 7.2 Install Netcat

```bash
sudo dnf install -y nmap-ncat
```

---

#### 7.3 Re-test Connectivity

```bash
nc -vz <aurora-endpoint> 5432
```

**Output (key lines):**

```
Connected to 10.x.x.x:5432
0 bytes sent, 0 bytes received
```

<img width="849" height="66" alt="image" src="https://github.com/user-attachments/assets/5ccb506f-a8e6-43da-a997-abbb38ebaf27" />

---

### 🧠 What This Proves

* DNS resolution works
* VPC routing is correct
* Security groups allow inbound 5432 from EC2
* Aurora writer instance is reachable
* No NACL blocking
* TCP handshake succeeds

📌 **Important:**
`nc -vz` checks **port availability**, not PostgreSQL authentication.

---

### Step 8: Verify PostgreSQL Authentication

Attempted `psql` connection with incorrect credentials → ❌

**Root Cause:**

* Master username is `postgres`
* Incorrect username/password initially used

After correcting credentials:

✅ Successful PostgreSQL connection

This confirms:

* Network connectivity ✅
* Authentication ✅
* Aurora PostgreSQL is fully reachable

---

### Step 9: Negative Security Test (External Access)

Exited EC2 and attempted connection **from laptop**:

```bash
nc -vz <aurora-endpoint> 5432
```

**Result:**

```
Connection refused
```

<img width="969" height="34" alt="image" src="https://github.com/user-attachments/assets/00ad8e3f-85bc-4205-848b-e96bb1974258" />

✅ **Expected behavior**

This proves:

* Aurora is **not publicly accessible**
* Security groups correctly restrict access
* Only EC2 inside the VPC can connect

---

### Step 10: Cleanup

To avoid costs:

```bash
terraform destroy
```

Confirmed with `yes`.

✅ All resources removed cleanly.

<img width="316" height="177" alt="Screenshot 2026-01-20 at 15 37 29" src="https://github.com/user-attachments/assets/b2d45e08-8b21-4fc1-807c-264e9339d742" />

---

## ✅ Key Commands Summary

| Task                      | Command                               |
| ------------------------- | ------------------------------------- |
| Initialize Terraform      | `terraform init`                      |
| Apply infrastructure      | `terraform apply`                     |
| List Aurora versions      | `aws rds describe-db-engine-versions` |
| View Terraform outputs    | `terraform output`                    |
| SSH into EC2              | `ssh -i key.pem ec2-user@ip`          |
| Install PostgreSQL client | `dnf install postgresql15`            |
| Install netcat            | `dnf install nmap-ncat`               |
| Test port connectivity    | `nc -vz endpoint 5432`                |
| Destroy resources         | `terraform destroy`                   |

---

## 📌 Lab Summary

| Area                          | Result |
| ----------------------------- | ------ |
| Aurora deployed via Terraform | ✅      |
| Engine version issue resolved | ✅      |
| EC2 → Aurora connectivity     | ✅      |
| Security groups validated     | ✅      |
| Authentication verified       | ✅      |
| External access blocked       | ✅      |
| Infrastructure cleaned up     | ✅      |

---

## ⚡ Takeaway

This lab proves **infrastructure correctness before migration**.

Key lessons:

* Terraform errors often come from **provider constraints**, not syntax
* Always verify supported engine versions per region
* `nc` validates **network**, not **authentication**
* Positive + negative tests build confidence
* Outputs turn Terraform into an operational tool
