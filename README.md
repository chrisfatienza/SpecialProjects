<p align="center">
  <img src="https://raw.githubusercontent.com/chrisfatienza/SpecialProjects/main/awx-zero-downtime-oracle%20at%20AWS%20and%20cisco%20UCS.png" width="100%" />
</p>

# Oracle 19c Multi-Region Active-Active Architecture (Multi-Write)
### AWS + RHEL 9 + Oracle GoldenGate

---

## 📌 Overview

This repository documents the complete end-to-end setup of an **Oracle Database 19c multi-region active-active (multi-write) architecture** using:

- AWS (3 Regions)
- RHEL 9
- Oracle Database 19c (19.19+)
- Oracle GoldenGate (Microservices Architecture)
- Conflict Detection & Resolution (CDR)

This design allows **writes in any region** with near real-time replication to other regions.

---

# 🏗 Architecture Summary

### Regions
- `us-east-1` (N. Virginia)
- `eu-central-1` (Frankfurt)
- `ap-southeast-1` (Singapore)

### In Each Region
- VPC (non-overlapping CIDR)
- EC2 for Oracle Database
- EC2 for GoldenGate
- EBS storage
- Private subnets
- Monitoring & logging

### Replication Model
- Active-Active (multi-write)
- Full mesh replication (A↔B, B↔C, C↔A)
- Conflict Detection & Resolution enabled

---

# ⚠ Important Design Decision

Choose your model before deployment:

## Model A – True Active-Active (Same Tables Writable Everywhere)
- Requires GoldenGate CDR
- Higher operational complexity
- Must handle update conflicts

## Model B – Write-Local / Data Ownership Model (Recommended)
- Each region owns specific rows or tenants
- Conflicts minimized by design
- GoldenGate distributes data globally

## Model C – Single Writer + Data Guard
- Not multi-write
- Simpler
- Strong consistency

---

# 🛠 Step 1 – AWS Network Setup (Per Region)

1. Create VPC (unique CIDR per region)
2. Create subnets:
   - Public
   - Private App
   - Private DB
   - Private GoldenGate
3. Configure:
   - Transit Gateway OR inter-region VPC peering
   - Security groups (DB 1521, GG ports, SSH restricted)
   - NACLs (optional)

---

# 🖥 Step 2 – Provision Oracle 19c (Per Region)

## 1. Launch EC2
- RHEL 9
- Separate EBS volumes:
  - /u01 (Oracle Home)
  - Data
  - Redo
  - Archive
  - FRA

## 2. Install Oracle 19c (19.19+ minimum)

## 3. Create Database
Recommended:
- CDB + PDB model

---

# 🗄 Step 3 – Prepare Oracle for GoldenGate

## Enable Archivelog + Force Logging

```sql
shutdown immediate;
startup mount;
alter database archivelog;
alter database force logging;
alter database open;
```

## Enable Supplemental Logging

```sql
alter database add supplemental log data;
alter database add supplemental log data (primary key) columns;
```

## Create GoldenGate User

```sql
create user ggs_admin identified by "StrongPassword";
grant create session to ggs_admin;
-- Add required GoldenGate privileges per Oracle documentation
```

### Requirements
- All tables must have Primary Keys
- Avoid non-deterministic triggers
- Ensure global unique key strategy

---

# 🔁 Step 4 – Install Oracle GoldenGate (Microservices)

Recommended: GoldenGate Microservices Architecture

## On GoldenGate EC2

1. Install required OS libraries
2. Install GoldenGate software
3. Create deployment:
   - dep-use1
   - dep-euc1
   - dep-apse1

Open required ports securely (Admin Server, Distribution Server, Receiver Server).

---

# 📦 Step 5 – Initial Data Load

Choose one region as initial source (example: us-east-1)

## Option A – Data Pump Method (Recommended)

1. Stop application writes briefly
2. Export schema
3. Import into other regions
4. Record SCN
5. Start Extract from recorded SCN

---

# 🔄 Step 6 – Configure Replication

## Recommended Topology: Full Mesh

- US ↔ EU
- EU ↔ APAC
- APAC ↔ US

---

# 🧲 Step 7 – Configure Extract (Per Region)

1. Create credential store
2. Create Integrated Extract
3. Create local trail
4. Configure distribution path to other regions

---

# 🔄 Step 8 – Configure Replicat (Per Region)

For each inbound stream:

1. Create Replicat process
2. Define mappings:

```
MAP schema.table, TARGET schema.table;
```

3. Enable DDL replication (if required)

---

# ⚖ Step 9 – Conflict Detection & Resolution (CDR)

Required for Active-Active.

## Common Conflict Types

- Update vs Update
- Insert vs Insert (PK collision)
- Delete vs Update

## Resolution Strategies

- Last update wins (timestamp)
- Region priority
- Custom PL/SQL handlers

Recommended: Design to avoid conflicts rather than resolve them.

---

# 🔐 Step 10 – Key Strategy (Critical)

Recommended approaches:

- UUID at application layer
- Region-based numeric ranges
  - US: 1–1B
  - EU: 1B–2B
  - APAC: 2B–3B

Avoid shared sequence without coordination.

---

# 🌍 Step 11 – Client Routing

To allow “write anywhere”:

## Option 1 – Route 53 Latency-Based Routing
Clients connect to nearest region.

## Option 2 – AWS Global Accelerator
Improved global performance and failover.

Application writes locally; GoldenGate replicates globally.

---

# 🛡 Step 12 – High Availability for GoldenGate

GoldenGate is critical infrastructure.

Recommended:
- 2 EC2 instances per region (active/passive)
- NLB for Microservices endpoints
- Auto-restart policies
- Monitoring for lag and process health

---

# 📊 Step 13 – Monitoring

Monitor:

- Replication lag
- Extract status
- Replicat status
- Trail disk usage
- Redo generation rate
- Cross-region latency

---

# 🧪 Step 14 – Validation Checklist

- [ ] Write in US appears in EU + APAC
- [ ] Write in EU appears in US + APAC
- [ ] Conflict simulation works as expected
- [ ] Kill GoldenGate process → auto-recovery
- [ ] Region network partition test
- [ ] Full region rebuild test

---

# 🔄 Operational Runbooks Required

You must document:

- Resync table procedure
- Rebuild region from another region
- Handling replication lag
- Maintenance window process
- GoldenGate upgrade process

---

# 📈 Performance Considerations

- Monitor redo generation rate
- Cross-region latency impacts convergence time
- Avoid large batch commits during peak replication

---

# 🚨 Risks & Warnings

- Multi-write introduces conflict risk
- Network partitions can cause divergence
- Operational complexity significantly higher than Data Guard
- Requires strong DBA + replication expertise

---

# 📚 When NOT to Use Multi-Write

Do not use active-active if:

- Strict serial consistency required
- No tolerance for conflict logic
- Low operational maturity
- Team unfamiliar with GoldenGate

Use Data Guard instead.

---

# 📌 Summary

Multi-write across regions with Oracle 19c:

✅ Requires Oracle GoldenGate  
✅ Requires conflict handling  
✅ Requires global key strategy  
✅ Requires strong monitoring  

Recommended approach:
Design for conflict avoidance first.

---

# 👤 Author

Architecture prepared for production-grade multi-region Oracle 19c deployments on AWS.

---

