# 5G NWDAF & OpenDaisy Federated Learning Design Documentation

This repository contains the complete technical design documents, system architecture, SBI integration specs, and operational workflows for **5G NWDAF (Network Data Analytics Function)**, **ADRF (Analytics Data Repository Function)**, **MTLF-subp (Gateway Microservice)**, and **OpenDaisy Federated Learning (FL)**.

## 📚 Document Index

### 1. System Architecture & Operation Flow
- [System Architecture and Operation Flow](<NWDAF with different FL/system_architecture_and_flow.md>)
  - Complete 5G Core, UPF-EES (`reportPeriod`), SMF, ADRF (TTL `idx_expiry_time`), NWDAF ANLF, MTLF-subp, and OpenDaisy (FedAvg) architecture and 5-phase end-to-end operational sequence diagram.

### 2. Integration & Design Analysis
- [Go NWDAF and OpenDaisy Integration Analysis](go_nwdaf_opendaisy_integration_analysis.md)
  - Detailed integration analysis between Go NWDAF and OpenDaisy FL framework.
- [ANLF / MTLF / OpenDaisy Flow](anlf_mtlf_opendaisy_flow.md)
  - Deep-dive into ANLF model drift detection, retrain triggering, and Webhook callback flow.
- [NWDAF Refactor & ADRF Sync Strategy](<NWDAF with different FL/refactor NWDAF and adrf sync.md>)
  - Design specifications for decoupled MTLF microservice and ADRF data synchronization.

### 3. SBI & Compliance Specs
- [NWDAF SBI and FL Compliance Design](nwdaf-sbi-and-fl-compliance-design.md)
  - 3GPP SBI (Service-Based Interface) compliance and Federated Learning MLOps integration.
- [Ideal MLModelProvision Flow](ideal_mlmodelprovision_flow.md) & [MLModelProvision Mapping](mlmodelprovision_mapping.md)
  - 3GPP TS 29.520 `Nnwdaf_MLModelProvision` data model mapping and lifecycle.

---

## 🛠️ System Overview

```text
+-------------------------------------------------------------------------------+
|                             5G Core & UPF-EES                                 |
|      (UPF-EES 1 & 2 via 3GPP TS 29.564 reportPeriod Tag -> SMF -> NRF)          |
+---------------------------------------+---------------------------------------+
                                        | Event Notifications
                                        v
+-------------------------------------------------------------------------------+
|                       ADRF & NWDAF Analytics Pipeline                         |
|   (ADRF Mongo Storage w/ TTL idx_expiry_time -> NWDAF ANLF Drift Monitoring)  |
+---------------------------------------+---------------------------------------+
                                        | POST /upload_data & /publish_task
                                        v
+-------------------------------------------------------------------------------+
|                          MTLF-subp Gateway Router                             |
|          (FastAPI Gateway, ADRFDataToOpenDaisyAdapter, Staging Manager)       |
+---------------------------------------+---------------------------------------+
                                        | Trigger FL Training
                                        v
+-------------------------------------------------------------------------------+
|                     OpenDaisy FL Training Core (Host)                         |
|  (Master 0.0.0.0:8887/9887 <-> gRPC <-> Client 0: 10087 & Client 1: 10088)    |
+-------------------------------------------------------------------------------+
```

---
*Maintained by hokusai0603*
