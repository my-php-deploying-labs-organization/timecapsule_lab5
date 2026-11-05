# ADR-0004: Database hosting for TimeCapsule

- **Status:** Accepted
- **Date:** 2026-11-05
- **Project:** TimeCapsule
- **Environment:** `dev`
- **AWS Region:** `eu-central-1`

## Context

In previous labs, the TimeCapsule application ran on `app-server` (EC2) while PostgreSQL ran on a separate `db-server` (EC2) in the private subnet. This required us to manage database software and operating-system maintenance, backups and recovery ourselves. The environment is used for development and teaching, so cost and private network access matter more than uninterrupted 24/7 availability.

The existing application uses the hostname `db.timecapsule.internal` to reach the database. Attachments are stored in a private Amazon S3 bucket; this architecture remains unchanged.

## Decision

Host the TimeCapsule PostgreSQL database on **Amazon RDS for PostgreSQL, Single-AZ**, using the DB instance `timecapsule-dev-rds` in `eu-central-1a`.

- Configure the DB subnet group `timecapsule-dev-db-subnets` with `private-a` (`10.27.10.0/24`, `eu-central-1a`) and `private-b` (`10.27.11.0/24`, `eu-central-1b`). Inclusion of a second subnet **does not** make Single-AZ deployment highly available.
- Disable public access. Permit inbound TCP 5432 in `timecapsule-dev-rds-sg` only from the application server's Security Group.
- Use the application-specific PostgreSQL role and database `timecapsule`, rather than connecting the application as `dbadmin`.
- Point the private Route 53 `CNAME` record `db.timecapsule.internal` at the RDS endpoint. Keep the application's `DB_HOST` unchanged.
- Enable storage encryption and automated backups, with a **target retention period of 7 days** as required by Lab 5; the final configured retention period must be checked in RDS before reporting it as achieved.
- Schedule `StopDBInstance` every day at 23:00 `Europe/Chisinau` via EventBridge Scheduler, using a dedicated IAM role restricted to this DB instance.
- Keep Amazon S3 for file attachments and the existing application IAM role for object access.

## Alternatives considered

| Alternative | Advantages | Disadvantages | Decision |
|---|---|---|---|
| PostgreSQL on EC2 | Full control; potentially low direct instance cost | Manual patches, backup automation, recovery, monitoring and OS administration | Rejected for this lab |
| **RDS PostgreSQL Single-AZ** | Managed PostgreSQL, automatic backups/PITR, simple migration, lower cost than redundant managed deployments | Temporary downtime during reboot; no automatic standby in another AZ | **Selected** |
| RDS PostgreSQL Multi-AZ | Standby in another AZ and automatic failover, better availability | Greater cost; unnecessary for this development environment | Consider for production |
| Amazon Aurora PostgreSQL-compatible | Managed cluster storage, high availability and scaling options | More architecture/operational complexity and potentially higher costs for this small lab workload | Not selected |

## Consequences

### Backups, recovery and potential data loss

RDS automated backups and transaction logs support point-in-time recovery within the configured retention window. The intended retention is **7 days**, but the value must be verified in the actual instance configuration (a 1-day period was used for the earlier temporary `lab5-db` instance). RDS periodically uploads transaction logs, typically on an approximately five-minute cadence; therefore the latest restorable time may lag the present. This describes a potential recovery-point gap, **not a guaranteed maximum data-loss bound**. Actual recovery point and recovery time depend on the most recent available logs and restore operations. PITR creates a separate DB instance; application routing must then be updated if that recovered instance is to become active. Backups do not eliminate the need to test restores.

### Cost and operating schedule

Lab instructions estimate about **USD 13–14 for three months**, assuming approximately three operating hours per day and 20 GiB of `gp3` storage in Frankfurt (October 2026 estimate, not a current-price guarantee). Storage continues to incur cost while the DB is stopped. RDS automatically starts a stopped DB after seven consecutive days; the nightly Scheduler will attempt another stop at its next run. A stopped DB may already reject a repeated `StopDBInstance` request; this should be monitored rather than treated as a data-loss event. Scheduling does not guarantee zero cost and does not stop `app-server`.

### Application discovery and AZ failure

The application resolves `db.timecapsule.internal` through a **private Route 53 hosted zone**; that name is a CNAME to the RDS endpoint. It is not necessary to change `DB_HOST` when replacing the former EC2 database, although open connections may need to reconnect. When `eu-central-1a` fails, this **Single-AZ** RDS instance may become unavailable; merely including `private-b` in the subnet group does not provide automatic failover. Recovery may require restoring a backup or changing deployment to Multi-AZ, causing downtime and possibly a recovery-point gap.

### Security and responsibilities

AWS operates underlying DB infrastructure and handles supported automated backup/maintenance functions, while the team remains responsible for schema design, SQL, DB roles, application-level validation, network permissions, monitoring, recovery testing, scheduling and cost control. SSL/TLS should be used for DB connections; for production, validate server certificates rather than relying on encryption without identity verification.

## Verification / evidence

- `docs/lab5/images/chapter6_rds-connectivity.png` — private RDS connectivity and Security Group.
- `docs/lab5/images/chapter7_schedule.png` — Scheduler configuration.
- `docs/lab5/images/chapter9_row-counts.png` — row counts before/after migration.
- `docs/lab5/images/chapter10_route53.png` — private CNAME after cutover.
- `docs/lab5/images/chapter11_rds-events.png` and `chapter11_cloudtrail.png` — maintenance events and stop API call.
- `docs/lab5/timecapsule_lab5_architecture.png` — updated architecture diagram.

## Revisit when

Move to Multi-AZ or another highly available architecture if the system gains real users, a defined availability SLA, strict recovery-point objectives or usage levels that justify higher costs.
