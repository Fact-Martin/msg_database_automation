# Multi-Campus High-Availability Database Infrastructure Automation
### Malawi School of Government (MSG) — Enterprise Systems Administration Framework

This repository contains a modular, production-ready Infrastructure-as-Code (IaC) framework designed to automatically deploy, optimize, secure, and maintain an enterprise relational database cluster across two campuses. 

The automation partitions architecture execution natively, designating the **Lilongwe (Kanengo) Campus** as the Primary Master database node and the **Blantyre (Mpemba) Campus** as the Hot Standby replication node.

---

## Architectural Topology Map

```text
       [ Ansible Control Node ] (Workstation Orchestrator)
                  │
        ┌─────────┴─────────┐
        ▼ (Port 22 / SSH)   ▼ (Port 22 / SSH)
┌───────────────────────┐   ┌───────────────────────┐
│    kanengo-primary    │   │    mpemba-standby     │
│    (Lilongwe Site)    │   │    (Blantyre Site)    │
│  IP: 192.168.106.132  │   │  IP: 192.168.106.131  │
│  Mode: Read-Write     │   │  Mode: Hot Read-Only  │
└───────────┬───────────┘   └───────────▲───────────┘
            │                           │
            └─────── WAL Streaming ─────┘
                    (Port 5432 / TCP)
```

---

##  Core Technical Design Concepts

### 1. Modular Configuration Governance (Ansible Roles)
Instead of a single, bulky script file, the infrastructure is broken down into structured, isolated components under a standard Ansible Role (`msg_postgres_cluster`). This layout isolates package delivery, performance optimization profiles, primary seeding steps, and standby streaming hooks into dedicated task files, ensuring zero configuration drift.

### 2. High-Availability Streaming Replication
The cluster utilizes PostgreSQL Write-Ahead Log (WAL) streaming. The Primary master node processes live student registrations and changes, continuously broadcasting transaction changes over network port 5432. The Standby node receives these changes and runs in hot standby (read-only) mode, ready to instantly take over if the primary campus node loses power or connectivity.

### 3. Strict Network Perimeter Hardening (Airtight Security)
To protect sensitive academic and institutional transcripts, out-of-the-box password-less loopbacks are disabled. System firewalls and Host-Based Authentication files (`pg_hba.conf`) are programmatically modified to accept connections **exclusively** from the local `192.168.106.0/24` subnet. All unknown network requests are dropped instantly.

### 4. Zero Plain-Text Credentials (AES-256 Encryption)
Following industry security baselines, no master passwords, replication secrets, or database credentials live in clear text within the repository code. All parameters are fully encrypted using **Ansible Vault (AES-256 encryption)** and decrypted securely in volatile memory only during the deployment phase.

### 5. Compliance Session Auditing (pgAudit Extension)
To comply with the strict regulatory and data privacy frameworks mandated for Malawian statutory bodies, the framework natively embeds the **pgAudit (PostgreSQL Audit)** extension. By binding directly to system RAM on boot, pgAudit creates detailed, unalterable log trails tracking all system changes, roles adjustments, data writes, and structural modifications (DDL) across the entire institution.

### 6. Automated Disaster Recovery & Space Management
The Primary master node automatically builds a non-chained logical backup engine utility script to dump and compress database records into a secure storage directory (`/var/backups/postgres/`). This process is registered to run natively every single night at 1:00 AM using system chronometers (`cron`).

---

##  Relational Data Structure Layout
The automation configures an interconnected relational schema utilizing cascading integrity rules (`ON DELETE CASCADE` / `ON DELETE SET NULL`) to maintain accurate references across institutional tracks:

*   **`departments`**: Tracks academic divisions and faculty heads.
*   **`staff`**: Stores instructor profiles linked to respective departments.
*   **`students`**: Tracks multi-campus student records, enrollment campuses, and years of study.
*   **`students_grades`**: Holds course transcripts, linked to respective student records and evaluating instructors.

---

##  Live Validation Verification Scripts (Testing Samples)

Use these commands on your nodes to verify that your automation pipeline has built the cluster flawlessly:

### 1. Test Live Replication Streaming (Run on Primary Node: `.132`)
Verify that the primary node sees the standby node and is successfully streaming transaction logs across the network:
```bash
sudo -u postgres psql -c "SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;"
```
**Expected Clean Output:**
```text
 application_name |   client_addr   |   state   | sync_state
------------------+-----------------+-----------+------------
 16/main          | 192.168.106.131 | streaming | async
(1 row)
```

### 2. Test Cross-Table Data Integrity (Run on Standby Node: `.131`)
Log into your read-only replica database and run an advanced relational join query to confirm that tables are fully populated and synced across your network:
```bash
psql -h localhost -U msg_admin -d msg_student_records -c "
SELECT 
    s.student_id,
    s.first_name || ' ' || s.last_name AS student_name,
    s.campus,
    d.department_name,
    g.course_name,
    g.numeric_grade
FROM students s
INNER JOIN departments d ON s.department_id = d.department_id
INNER JOIN students_grades g ON s.student_id = g.student_id;"
```
**Expected Clean Output:**
```text
 student_id |   student_name    | campus  |        department_name         |         course_name         | numeric_grade 
------------+-------------------+---------+--------------------------------+-----------------------------+---------------
          1 | Chikondi Phiri    | Mpemba  | Public Administration          | Public Policy Analysis      |            78
          2 | Limbani Banda     | Kanengo | Information Systems Management | Database Systems Management |            85
```

### 3. Test Read-Only Database Hardening (Run on Standby Node: `.131`)
Attempt to execute a database write command directly on the standby machine to verify that data integrity controls are operating correctly:
```bash
psql -h localhost -U msg_admin -d msg_student_records -c "CREATE TABLE test_breach (id INT);"
```
**Expected Safe Refusal Output:**
```text
ERROR: cannot execute CREATE TABLE in a read-only transaction
```

### 4. Test Live Security Auditing Compliance (Run on Primary Node: `.132`)
Execute a database transaction query modifying high-sensitivity fields directly on the Primary node to test the operational state of the pgAudit tracking engine:
```bash
psql -h localhost -U msg_admin -d msg_student_records -c "UPDATE students SET year_of_study = 4 WHERE national_id = 'MW-BL-101';"
```
Now, parse the active system log files to view the cryptographic forensic audit trail generated natively in the background:
```bash
sudo tail -n 10 /var/lib/postgresql/16/main/log/postgresql-*.log
```
**Expected Clean Output Statement:**
```text
2026-07-30 11:41:44.318 UTC msg_admin@msg_student_records LOG:  AUDIT: SESSION,1,1,WRITE,UPDATE,TABLE,public.students,UPDATE students SET year_of_study = 4 WHERE national_id = 'MW-BL-101';
```
*(The generated trace explicitly isolates the evaluating account user (`msg_admin`), target context framework (`public.students`), operation score (`WRITE,UPDATE`), and raw query syntax executed, ensuring total security accountability for external audit inspections).*

### 5. Test Disaster Recovery Backup Extraction (Run on Primary Node: `.132`)
Manually trigger the backup engine utility to confirm it cleanly captures data structures, saves processing logs, and applies compressed storage compression files:
```bash
# Trigger the automated script profile
sudo -u postgres /usr/local/bin/pg_backup.sh

# Inspect the backup file generation metrics
ls -lh /var/backups/postgres/
```
**Expected Clean Output:**
```text
total 4.0K
-rw-r--r-- 1 postgres postgres    0 Jul 29 14:45 backup_error.log
-rw-r--r-- 1 postgres postgres 2.0K Jul 29 14:45 msg_student_records_20260729_144512.sql.gz
```
*(The `2.0K` `.gz` archive file proves all tables and transcripts were securely packed, and the `0` byte error log size guarantees a fault-free run).*
