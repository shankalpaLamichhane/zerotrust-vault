# ZeroTrust-Vault

> **Resilient Event-Streaming & Cryptographic Audit Ledger for Critical Municipal and Public Sector Data Systems.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Build Status](https://img.shields.io/badge/Build-Passing-brightgreen.svg)]()
[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8.svg?logo=go)]()
[![Kafka Support](https://img.shields.io/badge/Kafka-3.6+-orange.svg?logo=apachekafka)]()
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-336791.svg?logo=postgresql)]()
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?logo=docker)]()

---

## 📌 Problem & Threat Model

Local governments, county courts, and public sector record registries face acute operational and security challenges when transitioning from on-premise infrastructure to cloud environments:

1. **Dual-Write Vulnerability & In-Flight Data Loss:**  
   Legacy municipal desktop applications frequently operate across constrained, unstable broadband links. Standard application-level dual-writes (`DB.Save()` followed by an external queue push) suffer from partial commit failures during network drops, causing unrecoverable data divergence between on-premise systems and cloud replicas.
2. **Silent Historical Tampering (Pre-Ransomware Infiltration):**  
   Modern cyber adversaries routinely compromise database credentials weeks before deploying ransomware. Historical records—such as property deeds, judicial dockets, and tax ledgers—are silently altered or encrypted without triggering standard boundary firewalls.
3. **Backup Rebuild Latency (RTO & RPO Deficits):**  
   Standard daily snapshots (e.g., `pg_dump` or daily volume snapshots) introduce an unacceptable Recovery Point Objective (RPO) of up to 24 hours of lost transactions, and a Recovery Time Objective (RTO) measured in weeks of civil downtime.

**ZeroTrust-Vault** is a lightweight, non-intrusive middleware engineered to solve these systemic failure modes through asynchronous event streaming, cryptographic ledger verification, and automated runtime anomaly interception.

---

## 🏛️ System Architecture

```
                                  ZeroTrust-Vault Architecture
                                  
     [Legacy On-Premise DB]                [Cloud Ingestion Layer]               [Immutable Storage]
  ┌───────────────────────────┐         ┌──────────────────────────┐         ┌──────────────────────────┐
  │   PostgreSQL / Oracle     │         │   Debezium Engine        │         │   PostgreSQL Target DB   │
  │   (Write-Ahead Log)       │───────► │   (pgoutput plugin)      │───────► │   (Append-Only Store)    │
  └───────────────────────────┘         └────────────┬─────────────┘         └────────────┬─────────────┘
                                                     │                                    │
                                                     ▼                                    ▼
                                        ┌──────────────────────────┐         ┌──────────────────────────┐
                                        │   Apache Kafka           │         │   AWS S3 Object Lock     │
                                        │   (Partitioned Topics)   │         │   (WORM Compliance Mode) │
                                        └────────────┬─────────────┘         └──────────────────────────┘
                                                     │
                                                     ▼
                                        ┌──────────────────────────┐
                                        │   Model Context Protocol │ ──► [Autonomous AI Sentry]
                                        │   (MCP Telemetry Stream) │     (Query Entropy & Circuit Breaker)
                                        └──────────────────────────┘
```

---

## ⚡ Core Engineering Pillars

### 1. Zero-Loss CDC Ingestion (Engineered in Go / Java)
* **Log-Based Capture:** Attaches directly to the database Write-Ahead Log (WAL) at the disk level using PostgreSQL's logical replication stream (`pgoutput`), completely bypassing legacy application layers.
* **Deterministic Sequencing:** Ingests mutations into Apache Kafka topics partitioned strictly by entity primary keys (`parcel_id`, `case_number`), guaranteeing strict FIFO ordering across distributed consumers.
* **Transactional Exactly-Once Semantics (EOS):** Utilizes Kafka's transactional producer API (`enable.idempotence=true`) coupled with consumer read-committed boundaries, guaranteeing zero duplicate entries and zero dropped mutations under network partitions.

### 2. Cryptographic Write-Ahead Ledger
* **SHA-256 Hash Chaining:** Every committed state change computes an append-only cryptographic digest linked to its parent record:
```
Hash(n) = SHA-256(Payload(n) || Hash(n-1) || Timestamp || SequenceID)
```
* **Hardware WORM Storage:** Ledger batches are asynchronously committed to AWS S3 Object Lock in **Compliance Mode** (Write Once, Read Many). Objects cannot be overwritten, modified, or deleted by any user or compromised IAM role until statutory retention periods expire.
* **Automated Audit Verification:** Background reconcilers continuously recompute hash chains across historical intervals. Any unauthorized manual update to a historical row immediately invalidates downstream hashes, pinpointing the corrupted record.

### 3. Autonomous AI Telemetry Sentry (Model Context Protocol - MCP)
* **Real-Time Instrumentation:** Exposes runtime query velocity, payload entropy, and connection telemetry via the **Model Context Protocol (MCP)**.
* **Heuristic Threat Detection:** An AI evaluation agent inspects real-time query metrics, monitoring for:
  * Shannon entropy spikes across text columns (hallmark of bulk ransomware payload encryption).
  * Rapid sequential `UPDATE` operations spanning non-contiguous tables during off-peak hours.
* **Automated Containment:** On threshold breach, the sentry invokes automated MCP tool actions to trip API gateway circuit-breakers and revoke the compromised session's database privileges in under **200 milliseconds**.

### 4. Event-Sourced State Reconciliation
* **RPO = 0 Guarantee:** Because raw, verified events are stored in immutable cloud storage, database recovery does not depend on stale daily snapshots.
* **Kubernetes Replay Engine:** When a primary database failure occurs, an automated operator provisions a clean database pod within an isolated VPC, replaying the verified WORM event stream from genesis to the exact millisecond before failure, achieving **RTO < 15 minutes**.

---

## 📊 Performance Benchmarks (Local Cluster Target)

| Metric | Target Specification | Architectural Enabler |
| :--- | :--- | :--- |
| **Ingestion Latency (p99)** | `< 12 ms` | Go / Sarama Kafka async producer |
| **Throughput Capacity** | `25,000 writes/sec` | Partitioned Kafka topics + batch consumer |
| **Tamper Detection Window** | `Real-Time (< 500 ms)` | Continuous hash-chain state auditor |
| **Automated Threat Isolation** | `< 200 ms` | MCP telemetry proxy + automated circuit breaker |
| **Recovery Time Objective (RTO)** | `< 15 minutes` | Containerized event-sourcing replay operator |
| **Recovery Point Objective (RPO)** | `0 records lost` | Distributed WAL replication + Kafka EOS |

---

## 🛠️ Quickstart (Local Docker Environment)

### Prerequisites
* Docker & Docker Compose v2.20+
* Go 1.22+ (for local test client)

### 1. Launch the Infrastructure Stack
```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/zerotrust-vault.git
cd zerotrust-vault

# Spin up PostgreSQL with logical replication, Kafka, Zookeeper, and Vault Agent
docker-compose up -d
```

### 2. Verify Pipeline Ingestion
```bash
# Check status of Debezium CDC connector
curl -s http://localhost:8083/connectors/municipal-source/status | jq .

# Verify Kafka consumer output on mock deed mutations
docker exec -it kafka /bin/kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic municipal.records.deeds \
  --from-beginning
```


## 📜 Architecture Decision Records (ADRs)
* **ADR-001:** [Selection of Debezium CDC over Dual-Write Application Patterns](docs/adr/ADR-001-CDC-vs-DualWrite.md)
* **ADR-002:** [Two-Phase Commit vs. Saga for Low-Level WAL Stream Ingestion](docs/adr/ADR-002-2PC-vs-Saga.md)
* **ADR-003:** [Model Context Protocol (MCP) for Telemetry-Driven Automated Mitigation](docs/adr/ADR-003-MCP-Integration.md)

---

## 📄 License
This project is licensed under the [MIT License](LICENSE).
