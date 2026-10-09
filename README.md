# Red Hat Certified Middleware — Interconnected Fleet Demo

This repository contains the complete reference architecture and Ansible automation playbooks to deploy a realistic, enterprise-grade **E-Commerce Order Processing Platform** connecting six **Red Hat Certified Middleware downstream products** across 10+ servers using a single Ansible command.

Every playbook is explicitly named after its corresponding certified downstream collection for full clarity and traceability.

---

## 🏛️ Architecture & End-to-End Data Flow

```
+-----------------------------------------------------------------------------------+
| STEP 1: Customer Browser                                                          |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| STEP 2: Ingress & Load Balancing (Red Hat JBCS / Apache HTTPD + mod_cluster)       |
|         - Collection: redhat.jbcs | Playbook: playbooks/07_jbcs.yml               |
+-----------------------------------------------------------------------------------+
                         |                                         |
                         | OIDC Token / SSO                        | HTTP Requests
                         v                                         v
+---------------------------------------------+   +---------------------------------+
| STEP 3: Identity & Auth (RHBK)              |   | STEP 4: Application Server Fleet|
|         - Collection: redhat.rhbk           |   |         (Red Hat JBoss EAP)     |
|         - Playbook: playbooks/02_rhbk.yml   |   |         - Collection: redhat.eap|
|                                             |   |         - Playbook:             |
| Distributed Session Cache (Data Grid)       |   |           playbooks/05_eap.yml  |
| - Collection: redhat.data_grid              |<--+-- Remote HTTP Session Caching   |
| - Playbook: playbooks/01_data_grid.yml      |   |                                 |
+---------------------------------------------+   +---------------------------------+
                                                                   |
                                                                   | JMS Orders
                                                                   v
+-----------------------------------------------------------------------------------+
| STEP 5: Message Queue (Red Hat AMQ Broker Artemis HA Pair)                        |
|         - Collection: redhat.amq_broker | Playbook: playbooks/03_amq_broker.yml   |
+-----------------------------------------------------------------------------------+
                                         |
                                         | Event Streaming
                                         v
+-----------------------------------------------------------------------------------+
| STEP 6: Real-time Event Stream (Red Hat AMQ Streams / Apache Kafka KRaft)         |
|         - Collection: redhat.amq_streams | Playbook: playbooks/04_amq_streams.yml |
+-----------------------------------------------------------------------------------+
```

---

## 📦 Playbook to Downstream Collection Mapping

| Phase | Playbook File | Downstream Collection (Automation Hub) | Red Hat Product |
|---|---|---|---|
| **Phase 0** | `playbooks/00_prerequisites.yml` | `ansible.posix`, `community.postgresql` | Base OS, Java 17, PostgreSQL DB |
| **Phase 1A** | `playbooks/01_data_grid.yml` | `redhat.data_grid` | Red Hat Data Grid |
| **Phase 1B** | `playbooks/02_rhbk.yml` | `redhat.rhbk` *(or `redhat.sso`)* | Red Hat Build of Keycloak (RHBK) |
| **Phase 2A** | `playbooks/03_amq_broker.yml` | `redhat.amq_broker` | Red Hat AMQ Broker |
| **Phase 2B** | `playbooks/04_amq_streams.yml` | `redhat.amq_streams` | Red Hat AMQ Streams (Kafka KRaft) |
| **Phase 3A** | `playbooks/05_eap.yml` | `redhat.eap` | Red Hat JBoss EAP Cluster |
| **Phase 3B** | `playbooks/06_jws.yml` | `redhat.jws` | Red Hat JBoss Web Server (Tomcat) |
| **Phase 4** | `playbooks/07_jbcs.yml` | `redhat.jbcs` | Red Hat JBoss Core Services |
| **Phase 5** | `playbooks/08_validate.yml` | Built-in URI & health modules | Multi-Service Smoke Test Suite |
| **Day-2** | `playbooks/day2_rolling_upgrade.yml` | `redhat.eap` | Zero-Downtime EAP Rolling Upgrade |

---

## 🚀 Quick Start Guide

### 1. Configure Automation Hub Access & Install Collections
Ensure your `ansible.cfg` contains your Red Hat Automation Hub token, then install all certified downstream collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Inventory & Variables
Review `inventory/hosts` and update host IPs or FQDNs. Set your Red Hat Customer Portal credentials (`RHN_USERNAME` and `RHN_PASSWORD`) or pass them via `inventory/group_vars/all.yml`.

### 3. Deploy the Full Stack
Deploy the entire interconnected infrastructure with a single playbook command:

```bash
ansible-playbook site.yml
```

### 4. Validate Endpoints
Run the automated smoke test suite:

```bash
ansible-playbook playbooks/08_validate.yml
```

---

## 🔄 Day-2 Operations: Zero-Downtime Rolling Upgrade

Demonstrate fleet management by executing a live rolling upgrade of Red Hat JBoss EAP without dropping active user sessions:

```bash
ansible-playbook playbooks/day2_rolling_upgrade.yml -e "eap_patch_version=8.0.1"
```

Each EAP node is drained from `mod_cluster`, patched via `eap_install` (from `redhat.eap`), validated, and returned to the cluster pool sequentially (`serial: 1`).

---

## 📈 Fleet Scalability Story

To scale the JBoss EAP cluster from **3 nodes to 50 nodes**, simply modify the inventory range:

```ini
[eap]
eap-[01:50].demo.local
```

No code modifications required. Ansible handles the idempotent provisioning across all 50 target nodes in parallel.
