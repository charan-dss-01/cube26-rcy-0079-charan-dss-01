# ARCHITECTURE · RCY RECOVERY MANAGER

## Complete Architectural Blueprint

The **RCY Recovery Manager** is an evidence-grounded AI financial recovery operations center. It ingests marketplace channel fee and reimbursement reports, matches them against upstream physical operational logs (Receiving, Prep, Pack, Returns), conservatively analyzes whether evidence contradicts or supports each deduction, and produces formal, defensible claim packages.

---

## 1. System Topology & Data Flow

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Next.js 14 Enterprise UI                               │
│  - Multi-tenant workspace switcher (org_demo_alpha, org_demo_bravo)             │
│  - Real-time Operations Dashboard (Fees, Recoverable Pipeline, Claim Precision) │
│  - Charges Explorer with search & multi-filter                                  │
│  - Interactive Evidence Graph (Charge ↔ Unit ↔ Shipment ↔ Order ↔ Evidence)     │
│  - Chronological Evidence Timeline with authentic timestamps                    │
│  - "Why Not Claim?" Conservative Audit Drawer                                   │
│  - Claim Package Builder (Frozen audit snapshot, JSON/PDF export)               │
│  - Data Ingestion & Manual Record Entry Center                                  │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │ REST API (JSON)
┌──────────────────────────────────────▼──────────────────────────────────────────┐
│                            FastAPI Backend Core                                 │
│  ├── Multi-Tenant Middleware & Tenant-Scoped SQLAlchemy 2.0 ORM                 │
│  ├── Ingestion Engine (CSV, XLSX, PDF, JSON normalizers with validation preview)│
│  ├── Hybrid RAG Engine (Multi-Hop Traversal + Semantic Cosine Embeddings)       │
│  ├── Conservative Recovery Agent (Deterministic Gates + Heuristic/LLM Reasoner) │
│  ├── Claim Assembly & Audit Service (Immutably freezes claim evidence packages) │
│  └── Storage Service (Cloudinary integration with tenant-isolated local fallback│
└──────────────────────────┬──────────────────────────────────────────┬───────────┘
                           │                                          │
┌──────────────────────────▼───────────────┐ ┌────────────────────────▼───────────┐
│     Relational & Vector Storage Layer    │ │          Cloudinary Storage        │
│  - PostgreSQL with pgvector              │ │  - /company/{id}/charges/          │
│    (Auto-fallback to SQLite + Cosine)    │ │  - /company/{id}/evidence/         │
│  - Complete tenant isolation (company_id)│ │  - /company/{id}/claims/           │
└──────────────────────────────────────────┘ └────────────────────────────────────┘
```

---

## 2. Core Entities & Relational Schema

```text
┌──────────────┐       1:N       ┌──────────────┐
│  companies   │────────────────▶│    users     │
└──────┬───────┘                 └──────────────┘
       │ 1:N
       ├─────────────────────────┐
       │ 1:N                     │ 1:N
┌──────▼───────┐          ┌──────▼──────────────┐
│   charges    │          │  evidence_records   │
└──────┬───────┘          └──────┬──────────────┘
       │ 1:1                     │ 1:N
┌──────▼───────┐          ┌──────▼──────────────┐
│investigations│          │   evidence_chunks   │
└──────┬───────┘          └─────────────────────┘
       │ 1:N
┌──────▼──────────────────┐
│ investigation_evidence  │
└─────────────────────────┘
       │ 1:1
┌──────▼───────┐
│    claims    │
└──────────────┘
```

### Table Definitions & Roles
1. **`companies`**: Tenant container guaranteeing strict organization boundary (`id`, `name`, timestamps).
2. **`users`**: Platform users scoped by role (`Admin`, `Analyst`, `Viewer`).
3. **`charges`**: Ingested channel deduction lines (`charge_id`, `unit_id`, `shipment_id`, `order_id`, `sku`, `reason`, `amount`, `currency`, `charge_date`, `status`).
4. **`evidence_records`**: Upstream physical operational logs from Receiving, Prep, Pack, and Returns (`evidence_id`, `source_type`, `unit_id`, `finding`, `raw_payload`, `photo_refs`, `operator_id`, `timestamp`).
5. **`evidence_chunks`**: Semantic embeddings for natural language terminology search.
6. **`investigations`**: Forensic analysis trail (`assessment`: `CONTRADICTED` | `SUPPORTED` | `SILENT` | `UNCERTAIN` | `DUPLICATE` | `ALREADY_REIMBURSED`, `claim_supported`, `claim_amount`, `reasoning`, `coverage_summary`).
7. **`investigation_evidence`**: Junction entity detailing relevance, what was established, and what was *not* established.
8. **`claims`**: Frozen dispute dossier containing immutable evidence citations and formal legal defense statement.
9. **`source_files`**: Ingested document records tracking Cloudinary URLs and parse status.

---

## 3. Hybrid RAG & Multi-Hop Traversal

Instead of basic document question-answering, Recovery Manager employs **structured relational multi-hop traversal**:

```text
FINANCIAL CHARGE (e.g. Inbound Defect Fee on FBA Shipment FBA-DUMMY-101)
       │
       ▼ [Hop 1: Exact Key Resolution]
   Unit ID: UNIT-0014
       │
       ▼ [Hop 2: Upstream Physical Record Traversal]
   ├── Prep Log: PRP-0014 (polybag_present_sealed: not_required, original_barcode_covered: yes, fnsku_label_placement: flat)
   ├── Receiving Log: RCV-0014 (carton_damage: none, unit_damage: none, qty_received: 24/24)
   └── Returns Desk: RTN-0014 (observed_state: signs_of_use, disposition: liquidate)
       │
       ▼ [Hop 3: Semantic Alignment & Evidence Relevance]
   What is established?   → Prep compliance verified flat & covered prior to dispatch.
   What is NOT established? → Condition during carrier handling post-dispatch.
```

---

## 4. The 5 Non-Negotiable Recovery Agent Rules

1. **Rule 1 — No Computer Vision:** Does not process camera streams. Consumes structured physical logs generated by upstream managers.
2. **Rule 2 — Never Invent Evidence:** Zero hallucinated inspection records. All reasoning statements link directly to database rows.
3. **Rule 3 — Never Invent Claim Amounts:** Potential recovery is mathematically bound to the documented charge ($38 fee = exactly $38 claim).
4. **Rule 4 — SILENT Is a Valid Result:** When evidence is missing, the agent refuses to speculate, returning \$0 recovery and an explicit missing proof audit checklist.
5. **Rule 5 — UNCERTAIN Is Valid:** When evidence is ambiguous or custody is split, the case is flagged for human review.

---

## 5. Security & Multi-Tenancy Isolation

- **Tenant Boundary:** Every SQL query and API endpoint explicitly filters on `company_id`.
- **Zero Leakage:** Automated tests (`tests/test_recovery.py::test_tenant_isolation`) verify that `org_demo_alpha` cannot query or see `org_demo_bravo` records.
- **Storage Isolation:** Uploaded files and Cloudinary directories are strictly partitioned by `/company/{company_id}/`.
