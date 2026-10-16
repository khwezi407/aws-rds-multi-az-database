# Business-Critical Contact Database: Multi-AZ RDS Migration

Migrating a single-point-of-failure database off an application server and onto
a managed, highly available Amazon RDS platform — with zero data loss on failover
and no dedicated DBA required to operate it.

---

## Context

This is a scenario-based case study built on a hands-on AWS build. The business
brief, requirements, and constraints below represent a realistic engagement; the
architecture, configuration, and verification are from the actual deployment.

---

## The Business Problem

A professional services firm runs an internal address book application that staff
use daily to reach clients. The application is business-critical — if it is
unavailable, client-facing work stops.

The existing setup:

- A single EC2 instance running both the web application and a locally installed
  MySQL server
- Single Availability Zone
- Manual, ad-hoc backups
- Manual OS and database patching

Three risks were raised by the business:

1. **Single point of failure.** If the instance or its Availability Zone fails,
   the contact database is offline with no defined recovery path and no defined
   recovery time.
2. **Data loss exposure.** Backups are infrequent and untested. There is no clear
   answer to "how much data would we lose?"
3. **Operational burden.** Patching, backups, and failover are manual and depend
   on one person. This is a bus-factor risk as much as a technical one.

The ask was not "set up RDS." The ask was: **make the contact database survive
infrastructure failure without hiring a DBA to run it.**

---

## Requirements

### Functional
- Web application continues to read and write contact records with no code change
- Data model and existing data preserved

### Non-Functional
| Requirement | Target | Rationale |
| :--- | :--- | :--- |
| Availability | Survive loss of a single Availability Zone | Removes the single point of failure |
| Recovery Time Objective (RTO) | < 2 minutes, automatic, no manual intervention | Staff cannot wait on a person being paged |
| Recovery Point Objective (RPO) | 0 committed transactions | Contact records are edited live; losing them erodes trust in the system |
| Security | Database not reachable from the internet | Contact data is commercially sensitive |
| Operational overhead | No dedicated DBA; no manual failover runbook | The firm has no database specialist |
| Cost | Predictable, justified against cost of downtime | Small firm, fixed budget |

---

## Constraints

- Existing VPC and subnet layout had to be reused — no greenfield network build
- Small team, no dedicated database administrator
- Application code could not be rewritten for this change
- Cost increase had to be defensible, not assumed acceptable

---

## Architecture

![Architecture](screenshots/01-architecture.png)

**Flow:**

1. Web server runs in a public subnet in Availability Zone B
2. RDS primary instance runs in a private subnet in Availability Zone A
3. RDS standby instance runs in a private subnet in Availability Zone B
4. The DB Security Group permits MySQL (3306) **only** from the Web Security Group
5. Synchronous replication keeps the standby current with the primary
6. Both database subnets have no route to the internet

The two Availability Zones are the core of the design: primary and standby never
share a failure domain.

---

## Design Decisions

Each decision below was made against at least one rejected alternative. The
reasoning is the substance of the design, not the configuration.

| Decision | Why | Alternative rejected | Trade-off accepted |
| :--- | :--- | :--- | :--- |
| **Multi-AZ over Single-AZ + snapshots** | Meets the < 2 min RTO automatically. Snapshots restore in minutes-to-hours and require a human to trigger them. | Single-AZ with frequent snapshots | Roughly 2x the database cost. Justified: the cost of a lost business day exceeds the annual delta. |
| **Multi-AZ over Read Replicas** | The requirement is availability, not read scaling. Read Replicas are asynchronous and require manual promotion — wrong tool for an RTO problem. | Read Replica promoted manually on failure | Cannot offload read traffic to the standby. Acceptable: this workload is not read-bound. |
| **Managed RDS over self-managed MySQL on EC2** | Removes patching, backup, and failover from the firm's plate — directly addresses the bus-factor risk. | MySQL on EC2 with a replication setup | Less control over engine-level tuning. Acceptable: the team has no DBA to use that control. |
| **Security group referencing over IP allow-listing** | Referencing the Web Security Group means access survives instance replacement or IP change. No hardcoded addresses to rot. | Allow-list the web server's private IP or subnet CIDR | Requires the web tier to consistently carry the Web Security Group. Enforced as a standard. |
| **Private subnets, public access disabled** | Removes the database from internet reachability entirely, eliminating brute-force and scanning exposure. | Public subnet with a restrictive security group | Application must live in the VPC to connect. Already true. |
| **Storage encryption enabled** | Protects data at rest with no performance or operational cost. | Unencrypted storage | Negligible. Encryption is free and effectively transparent. |
| **gp3 storage over gp2** | Baseline 3,000 IOPS independent of volume size, at lower cost per GB. | gp2 | None meaningful for this workload. |

---

## Failure Modes & Recovery Behaviour

This is the part that matters to the business — not that the database is
"highly available," but what actually happens on the day something breaks.

| Failure | What happens | Recovery | Manual action required |
| :--- | :--- | :--- | :--- |
| Primary AZ outage | RDS detects the unhealthy primary and promotes the standby | Automatic, typically 60–120 seconds | None |
| Primary instance failure | Same automatic promotion path | Automatic, typically 60–120 seconds | None |
| Storage or host degradation | AWS-managed; RDS replaces the underlying resource | Automatic | None |
| Application server failure | Web tier is replaceable independently; database unaffected | Rebuild web instance | Reattach Web Security Group |
| Accidental data deletion | Covered by automated backups and point-in-time recovery | Restore to a point in time | Yes — deliberate restore |

**Connection behaviour during failover:** the RDS endpoint is a DNS name, not an
IP. On failover, the CNAME is repointed to the promoted instance. Applications
that reconnect on connection loss recover without configuration change. This is
why the application uses the endpoint and never a resolved IP.

---

## Security Model

Defence in depth, in the order a connection must pass:

1. **Network** — RDS resides in private subnets with no internet route
2. **Public access** — explicitly disabled; the instance has no public IP
3. **Security group** — inbound MySQL only, and only from the Web Security Group
4. **Authentication** — database credentials required at the engine layer
5. **Encryption at rest** — storage encrypted with an AWS managed KMS key

No step assumes the previous one held. The security group is not the only control;
it is one of five.

![DB Security Group](screenshots/06-db-security-group.png)

---

## Cost Position

Multi-AZ approximately doubles the database instance cost — the firm pays for a
standby that serves no traffic.

The comparison that justified it was not "RDS vs EC2." It was **the annual cost
of the standby against the cost of one lost business day.** For a firm whose
staff cannot reach clients without this system, a single avoided outage pays for
the redundancy. The decision was presented to the business in those terms, not as
an infrastructure preference.

---

## Evidence

| | |
| :--- | :--- |
| Architecture | ![Architecture](screenshots/01-architecture.png) |
| DB Subnet Group (two AZs) | ![DB Subnet Group](screenshots/02-db-subnet-group.png) |
| RDS configuration | ![RDS Configuration](screenshots/03-rds-configuration.png) |
| Instance available, Multi-AZ | ![RDS Available](screenshots/04-rds-available.png) |
| Connectivity and endpoint | ![RDS Endpoint](screenshots/05-rds-endpoint.png) |
| DB Security Group and replication | ![DB Security Group](screenshots/06-db-security-group.png) |
| Web Security Group | ![Web Security Group](screenshots/07-web-security-group.png) |
| Application working against RDS | ![Address Book Working](screenshots/08-address-book-working.png) |

---

## Configuration Reference

| Setting | Value |
| :--- | :--- |
| Engine | MySQL Community 8.4.11 |
| Instance class | db.t3.medium (2 vCPU, 4 GB) |
| Deployment | Multi-AZ (primary + standby) |
| Storage | 20 GiB gp3, 3,000 IOPS baseline |
| Encryption | Enabled, AWS managed KMS key |
| Public access | Disabled |
| Initial database | `lab` |
| Master username | `main` |
| VPC | Lab VPC |
| Subnets | Private Subnet 1 (10.0.1.0/24), Private Subnet 2 (10.0.3.0/24) |
| Security group | DB Security Group |

---

## Limitations & What I Would Do Next

Stated deliberately — a design without known limits is an unexamined design.

**Known limitations of the current build:**

- **Failover has not been load-tested.** The 60–120 second figure is AWS's
  documented behaviour, not a number measured against this application's
  connection handling. Next step would be a deliberate failover during synthetic
  load to confirm the application reconnects cleanly and to measure real RTO.
- **No connection pooling or retry logic in the application.** The app relies on
  the driver reconnecting. A production build would add explicit retry with
  backoff to make failover invisible rather than merely survivable.
- **Single region.** Multi-AZ protects against AZ failure, not regional failure
  or a bad deployment. Cross-region disaster recovery is out of scope here but
  would be the next tier for a stricter RPO/RTO.
- **No monitoring or alerting configured.** There is no CloudWatch alarm on
  failover events, replication lag, or storage headroom. Availability you cannot
  observe is availability you cannot prove.
- **Credentials are not in a secrets manager.** Moving to AWS Secrets Manager
  with automatic rotation would be the next security iteration.
- **Backup retention is default.** Retention period and restore procedures should
  be defined against the RPO and actually tested, not assumed.

**What I would do differently at larger scale:** introduce Read Replicas for read
offload once the workload justifies it, add a multi-region DR strategy if the RPO
tightens, and formalise the security group chaining as an organisational standard
rather than a per-deployment choice.

---

## What This Demonstrates

- Translating a business risk ("we can't afford to be down") into specific,
  measurable requirements (RTO < 2 min, RPO = 0)
- Selecting between Multi-AZ and Read Replicas on the basis of the actual
  requirement rather than the more familiar option
- Applying defence-in-depth rather than a single control
- Reasoning about failure modes and recovery behaviour, not just steady state
- Justifying cost in business terms
- Recognising and documenting the limits of your own design

---

## Author

**Khwezi Maphumulo**
AWS re/Start Participant at Praesignis
Building in public on LinkedIn

## License

MIT License. Free to use for learning.
