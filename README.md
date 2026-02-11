1. AWS Zero Downtime with Oracle + GoldenGate + AWS + Cicso UCS
end-to-end step-by-step setup guide for a 3-region active-active Oracle 19c design on AWS using GoldenGate (Microservices or Classic).

Reference architecture (what you’re building)

3 AWS Regions (example):

us-east-1 (N. Virginia)

eu-central-1 (Frankfurt)

ap-southeast-1 (Singapore)

In each region

VPC + subnets (public + private)

EC2 instances running RHEL 9 + Oracle Database 19c (19.19+)

One or more EC2 instances running Oracle GoldenGate (can be “hub” style or local-per-region)

Route 53 / Global Accelerator to route clients to nearest region

TLS + KMS encryption + monitoring

GoldenGate active-active

Bidirectional replication (A↔B, B↔C, C↔A) or hub-and-spoke

Conflict Detection & Resolution (CDR) configured (required for active-active)

Reality check: multi-write is doable, but you must design conflict strategy, key generation, and often data ownership rules (more on this below).

Step 0 — Decide your multi-write model (critical)

Pick ONE of these, or your project will hurt later:

Model A: “Same tables writable everywhere” (true active-active)

Needs GoldenGate CDR

Hardest (conflicts possible)

Model B: “Write-local / own-your-record” (recommended)

Each region owns specific rows/tenants (by region/tenant key)

Replication distributes changes globally

Conflicts are rare by design

Model C: “Single-writer + fast failover”

Use Data Guard; not multi-write

Best consistency, simpler ops

If you still want multi-write “regardless of location,” Model B is the best way to deliver that safely.

Step 1 — Build AWS networking (3 regions)

For each region:

Create VPC (non-overlapping CIDRs across regions)

Create subnets:

Public subnet(s): bastion / LB (optional)

Private app subnet(s)

Private DB subnet(s)

Private GG subnet(s)

Connectivity between regions

Easiest: Transit Gateway + inter-region peering, or VPN/Direct Connect between regions

Ensure low-latency paths and no asymmetric routing issues

Security:

Security groups: allow DB (1521), GoldenGate ports (depends on deployment), SSH (restricted)

NACLs: keep simple unless required

Step 2 — Provision EC2 for Oracle DB (each region)

Launch EC2 (sizing depends on workload), attach EBS volumes:

Data

Redo

Archive

FRA / backup

Install RHEL 9, patch baseline, set:

hostname, DNS, NTP/chrony

kernel params and limits for Oracle

Install Oracle 19c 19.19+ on RHEL 9

Create CDB/PDB (recommended) and your application schema.

Step 3 — Configure Oracle DB for GoldenGate (each region)

On each database:

Set archivelog + force logging:

shutdown immediate;
startup mount;
alter database archivelog;
alter database force logging;
alter database open;


Enable supplemental logging (minimum + table-level as needed):

alter database add supplemental log data;
alter database add supplemental log data (primary key) columns;
-- If tables lack PKs, you’ll need ALL columns (expensive) or add keys.


Create a GoldenGate admin user (least privilege approach varies by GG mode; typical example):

create user ggs_admin identified by "StrongPwd...";
grant create session to ggs_admin;
-- plus required privileges per Oracle GG docs for integrated extract/replicat


Ensure unique row identifiers everywhere:

Primary keys on all replicated tables

Avoid triggers that generate non-deterministic values unless handled carefully

Step 4 — GoldenGate topology choice
Option 1: GoldenGate Microservices Architecture (recommended)

Web UI + REST services

Easier lifecycle and HA patterns

Oracle’s install flow is: install → env vars → deploy instance via config assistant

Option 2: Classic (ggsci)

Works fine; more CLI-centric

The AWS blog posts show HA patterns for GG hubs on EC2 and Microservices HA guidance.

Step 5 — Install GoldenGate (each region)

Provision EC2 for GG (can be separate from DB; recommended separate)

Install prerequisites (Java for Microservices, OS libs)

Install GG software

Create GG deployment(s):

dep-use1, dep-euc1, dep-apse1

Open required ports (Admin Server, Receiver Server, Distribution Server, etc.) and lock them down to private networks.

Step 6 — Initial load (get all regions consistent)

Pick a system of record to seed the other regions (initially).

Approach A (common): Data Pump + start replication from a point-in-time

Stop application writes briefly (or use consistent snapshot method)

Export schema from Region A

Import into Regions B and C

Record SCN/time

Start Extract from that SCN so changes after export are replicated

Approach B: GoldenGate initial load (supported patterns)

Use GG to do initial load then switch to CDC.

Step 7 — Configure replication paths (3 regions)

You have two sane patterns:

Pattern 1: “Full mesh” (A↔B, B↔C, C↔A)

Lowest dependency

More streams to manage

Pattern 2: “Hub-and-spoke”

Region A is hub

Simpler streams, hub is critical

For “regardless of location,” full mesh is common.

Step 8 — Create Extract + Pump (CDC capture)

On each region’s GG deployment:

Create credential alias to the local DB (Microservices)

Create Integrated Extract capturing changes from the local DB (best for Oracle)

Create a trail

Create Distribution/Receiver paths to other regions

Repeat per region.

Step 9 — Create Replicat (apply) + mapping rules

For each inbound stream:

Create Replicat to apply remote changes into local DB

Define table mappings:

Include/exclude tables

Include DDL rules if needed (be careful with DDL in active-active)

Handle sequences/identity:

Prefer region-coded key ranges (best)

Or use UUIDs at app layer

Avoid “same sequence everywhere” without strategy

Step 10 — Enable Active-Active conflict handling (CDR)

Because active-active can create collisions, you must decide:

Conflicts you must plan for

Update/update same row in different regions

Delete/update

Insert/insert (PK collision)

GoldenGate provides Conflict Detection and Resolution specifically for active-active.

Common resolution policies:

“Last update wins” (timestamp-based)

“Region priority” (US wins, else EU, else APAC)

“Custom handlers” (business logic)

Best practice: avoid conflicts by design (Model B: row ownership/partitioning) and use CDR as a safety net.

Step 11 — Routing: “write anywhere” user experience

To make it feel location-independent:

Route clients to nearest region

Route 53 latency-based routing OR Global Accelerator

Application connects to local Oracle in-region

Writes happen locally

GoldenGate replicates changes to other regions

Important app considerations:

Read-your-writes: may require session affinity or local reads

Global uniqueness: IDs must be globally unique without coordination

Step 12 — HA for GoldenGate itself

GoldenGate is part of your write path (for convergence), so make it HA:

Run GG hub/deployment on 2 EC2 instances (active/passive or active/active where supported)

Put them behind a Network Load Balancer for Microservices endpoints (where appropriate)

Monitor and auto-restart Extract/Replicat

AWS explicitly notes the GG “hub on EC2” is not managed by RDS and should be monitored to ensure processes resume after failover.
AWS also provides reference HA patterns for GG hubs on EC2.

Step 13 — Observability + operations

Minimum monitoring:

GG lag per stream (seconds + bytes)

Extract/Replicat status

Trail space usage

DB metrics: log switches, archive destination space, redo rates

Network latency between regions

Runbooks:

How to pause replication for maintenance

How to resync a table

How to rebuild a region from another region

Step 14 — Testing checklist (must pass before go-live)

Write in US → appears in EU + APAC within SLA

Write in EU → appears in US + APAC

Simulate conflict → confirm resolution behavior

Kill GG node → automatic failover + resume

Network partition between 2 regions → understand what happens (this is where conflicts spike)

Reconcile after partition → verify data correctness
