# OCI SKU Pairing Rules

This file defines which SKUs must appear together in a valid OCI pricing configuration.

---

## Compute — Resource-Based Shapes (OCPU + Memory must both be present)

These shape families use separate OCPU and Memory SKUs. If an OCPU SKU appears,
a corresponding Memory SKU for the same family MUST also be present.

| Shape Family | OCPU SKU keyword      | Memory SKU keyword         |
|--------------|-----------------------|----------------------------|
| E3           | "E3 - OCPU"           | "E3 - Memory"              |
| E4           | "E4 - OCPU"           | "E4 - Memory"              |
| E5           | "E5 - OCPU"           | "E5 - Memory"              |
| X9 Standard  | "X9 - OCPU"           | "X9 - Memory"              |
| X9 Optimized | "Optimized - X9"      | "Optimized - X9 - Memory"  |
| A1 (Ampere)  | "Ampere A1 - OCPU"    | "Ampere A1 - Memory"       |
| A2 (Ampere)  | "A2 - OCPU"           | "A2 - Memory"              |

**Rule:** For each OCPU line item found, scan the entire sheet/section for a
Memory line item of the same family. If absent → ❌ MISSING PAIR.

---

## Compute — Fixed/Bundled Shapes (no separate Memory SKU needed)

These shapes have memory bundled in — do NOT flag as missing pair:

- All BM.GPU.* (bare metal GPU) shapes
- VM.GPU.* shapes
- BM.Standard.* bare metal shapes (fixed memory)
- X7 virtual machines (fixed OCPU/memory bundles)
- Preemptible instances (sub-core, memory included)

---

## Block Volume

Block Volume has up to three separate SKUs — all are optional but commonly appear together:

| SKU type            | Description                              |
|---------------------|------------------------------------------|
| Storage             | Capacity in GB/month                     |
| Performance Units   | VPUs/GB/month (10 VPUs = balanced tier)  |
| Data Transfer Out   | GB transferred out of the volume         |

No mandatory pairing rule — but if Storage is present without Performance Units,
add a ⚠️ NOTE: "Performance Units SKU not found — defaulting to 0 VPUs (lower perf tier)."

---

## Database Services

### DBaaS / Base Database Service (VM or BM)
- Database OCPU SKU + Block Storage SKU should both be present
- If only DB OCPU is present with no storage → ⚠️ NOTE: "No attached Block Storage SKU found"

### Autonomous Database (ATP/ADW)
- Billed by ECPU or OCPU (not both) + Storage
- If OCPU and ECPU SKUs both appear for same DB → ⚠️ WARNING: "Both OCPU and ECPU billing found for same service — verify intent"

---

## Networking

- **Load Balancer**: Base (port-hour) SKU + Bandwidth (Mb/sec-hour) SKU should both appear
  - If only Base found → ⚠️ NOTE: "Load Balancer Bandwidth SKU not found"
- **Site-to-Site VPN**: Free service — no SKU needed. If priced → ⚠️ WARNING
- **FastConnect**: Port-hour SKU only (data transfer covered separately under networking)

---

## Services with No Pairing Requirements

These are single-SKU services — no pairing check needed:
- Object Storage
- Archive Storage
- OCI Generative AI (on-demand)
- Functions
- Container Engine (OKE)
- DNS
- Email Delivery
- Certificates
- Vault / Key Management
- Notifications / Events / Queues
