# 🏥 Pulse Pharmacy – Smart Pharmacy Management System

**A comprehensive ServiceNow scoped application for end-to-end pharmacy operations automation.**

---

## 📌 Project Overview

**Pulse Pharmacy** is a fully automated, scoped ServiceNow application built to digitize and streamline pharmacy operations. It replaces manual, error-prone processes with a unified digital platform that handles:

- Medicine inventory tracking with expiry monitoring
- Customer self-service requests via Service Portal
- Automated admin approvals with SLA enforcement
- Pharmacist fulfillment tasks with timeout logic
- Proactive low-stock alerts and incident creation
- Complaints management with ITSM integration
- Real-time dashboards and operational reporting

The system supports **3 distinct personas**: Customer, Pharmacist, and Admin.

---

## 🎯 Problem Statement

Manual pharmacy operations introduce critical risks:

- **Stock Blindness** – No alerts when medicines run low; shortages are discovered only when a customer requests.
- **Expiry Risks** – Expired stock is checked manually; missed entries cause compliance violations and patient safety risks.
- **Paper Requests** – Customers must call or visit physically; no automated tracking or status visibility.
- **No SLA Visibility** – Complaints and requests sit without defined deadlines; no escalation when service levels breach.

---

## ✅ Key Features

| Feature | Description |
| :--- | :--- |
| **Medicine Inventory** | Custom `x_pharmacy_medicine` table with fields: name, quantity, expiry, batch, storage, status, reorder level. Form with 2 sections, list view with 7 columns. |
| **Customer Service Portal** | Self-service portal with dynamic welcome, search widget, catalog item, and quick links ("My Requests", "Report a Problem"). |
| **Service Catalog** | Catalog item with 5 variables: medicine name (mandatory), quantity (mandatory), pickup date (mandatory), notes, search availability. |
| **RITM Stage Flow** | New → Under Review → Approved → Ready for Pickup → Completed / Rejected. |
| **Admin Approval** | Flow Designer approval assigned to Admin group only, with 1-hour SLA. |
| **Stock Deduction** | On approval, stock is deducted instantly (prevents double selling). Restored on cancellation or timeout. |
| **Pharmacist Task** | Catalog Task created for Pharmacist group with email containing dynamic medicine details (via Mail Script). |
| **Pickup Timeout** | 2-business-day wait condition; auto-cancel + stock restore if customer no-shows. |
| **Low-Stock Alerts** | Business Rule triggers event → Email Notification to Admin (dynamic: medicine name, current qty, suggested reorder). |
| **Low-Stock Incident Automation** | Auto-creates Incident assigned to Pharmacy Admin; priority = Critical (qty=0) / Moderate (qty>0); no duplicates. |
| **Expiry Monitoring** | Scheduled job daily at 06:00 flags `near_expiry` (≤30 days) and `expired`; blocks approvals; UI Policy makes expired records read-only. |
| **Expired Disposal Flow** | Auto-creates Incident for Pharmacist on expiry; sends email with medicine details; on resolve: quantity=0, status=Disposed; Admin notification. |
| **SLAs** | Two SLAs: Medicine Request (1h, start=Under Review, stop=Approved/Rejected); Complaint (2h, start=New, stop=Resolved). Breach notifications at 75% and 100% to Admin. |
| **Notifications** | 3 email notifications: Request Received (customer), Pickup Ready (customer), New Request (Admin). All dynamic recipients. |
| **Dashboard** | 4 widgets: inventory by status, requests by stage, top 10 medicines, low-stock count. |
| **Incident Management** | Complaints auto-assigned to Admin; resolution notes hidden until Resolved (mandatory). Full lifecycle: New → In Progress → Resolved → Closed. |

---

## 🔐 Roles & Security (ACLs)

| Role | Responsibilities | ACL Enforcement |
| :--- | :--- | :--- |
| **Customer** | Submit requests, track status, report complaints | `opened_by == gs.getUserID()` – sees only own records |
| **Pharmacist** | Manage inventory, prepare orders, complete tasks | Full read/write on medicine, RITMs, and tasks |
| **Admin** | Approve/reject requests, receive alerts, configure SLAs, monitor dashboard | Full CRUD across all tables |

- **Key ACL Rule:** `query_range` + `query_match` on medicine table enables Customer role to see dropdown values without exposing backend records.

---

## 🛠️ Technical Implementation

### Scripting & Logic

| Component | Description |
| :--- | :--- |
| **Client Scripts** | onLoad: GlideAjax for price fetch; onChange: real‑time cost projection; onSubmit: validation (non‑integer, past dates). |
| **Business Rules** | LowStockAlert (queues event on qty ≤ reorder_level); BlockExpiredMedicine (aborts approval if expired); Revert Stock on Cancel. |
| **Script Includes** | `MedicineScripts` class with `client_callable = true`; methods: `getUnitPrice()`, `checkAvailability()`. |
| **Mail Script** | Fetches catalog variables from RITM for Pharmacist email notifications. |
| **UI Policies** | Resolution notes hidden until State=Resolved (mandatory); expired medicine records read‑only. |
| **Scheduled Job** | `ExpiryCheck` runs daily at 06:00; updates statuses `near_expiry` / `expired`. |

### Flows (Flow Designer)

- **Medicine Approval Flow:** Triggered on RITM creation; stages: Under Review → Ask Approval → Approved (stock deduction + Catalog Task + email) / Rejected (refusal email).
- **Timeout Flow:** Wait for condition (2 business days); auto‑cancel + stock restore on no‑show.

### SLA Definitions

| SLA Name | Table | Duration | Start | Stop |
| :--- | :--- | :--- | :--- | :--- |
| MedicineRequestSLA | `sc_req_item` | 1 hour | Stage = Under Review | Stage = Approved / Rejected |
| ComplaintSLA | `incident` | 2 hours | State = New | State = Resolved |

### Reports & Dashboard

| Report | Type | Group By |
| :--- | :--- | :--- |
| InventoryByStatus | Pie chart | Status |
| RequestsByStage | Bar chart | Stage (filtered to catalog item) |
| TopMedicines | Bar chart | `medicine_name` variable (limit 10) |
| Low Stock Count | Single Score | `qty ≤ reorder_level` |

**Dashboard:** All 4 widgets displayed on a single interactive page.

---

## 🚀 Enhancements (Beyond the Core)

| Enhancement | Description |
| :--- | :--- |
| **Reservation Logic** | Stock deducted immediately after approval (no double selling). Restored on cancellation or timeout. |
| **Pickup Timeout** | 2-business-day wait; auto‑cancel + stock restore on no‑show. |
| **Pharmacist Catalog Task** | Clear separation: Admin approves, Pharmacist prepares. |
| **Mail Script** | Dynamically fetches medicine name & quantity from RITM for Pharmacist email. |
| **UI Policy on Expired** | Expired medicine records become read‑only. |
| **Low‑Stock Incident Automation** | Auto‑creates Incident for Admin with priority Critical/Moderate; no duplicates. |
| **Expired Disposal Flow** | Auto‑Incident → Pharmacist email → quantity=0, status=Disposed → Admin notification. |

---

## 🧪 Testing & Demo Scenarios

| Scenario | Medicine | Expected Behavior |
| :--- | :--- | :--- |
| **Low‑Stock Alert** | Panadol (qty=5, reorder=10) | Email to Admin; Incident created (if implemented). |
| **Near Expiry** | Amoxicillin (expiry within 30 days) | Scheduled Job updates status to `near_expiry`. |
| **Expired – Approval Block** | Ibuprofen (expired) | Admin gets error when approving; UI Policy makes record read‑only. |
| **Expired – Disposal Flow** | Ibuprofen (expired) | Auto‑Incident → Pharmacist email → resolve → qty=0, status=Disposed. |
| **Full Demo** | Vitamin C (active) | Customer submits → Admin approves → stock deducted → Pharmacist prepares → pickup email → complete. |

---

## 📁 Setup & Import

1. Clone the repository or download the Update Set.
2. Import the Update Set into your ServiceNow PDI (or development instance).
3. Ensure the following tables exist:
   - `x_pharmacy_medicine` (custom)
   - `sc_req_item` (extended)
   - `sc_task` (extended)
   - `incident` (native)
4. Activate the flows and business rules.
5. Publish the catalog item to Service Portal.
6. Run the scheduled job manually for initial expiry checks.

---

## 🛡️ Challenges & Resolutions

| Challenge | Solution |
| :--- | :--- |
| **Medicine Dropdown Hidden** | Applied `query_range` + `query_match` ACLs on medicine table. |
| **Cross‑Scope Actions Blocked** | Enabled cross‑scope access in Application Properties. |
| **GlideAjax Calls Failing** | Set `client_callable = true` in Script Include. |
| **Duplicate Catalog Tasks** | Added `u_task_created` flag to RITM with condition check. |
| **Variables Missing in Pharmacist Email** | Implemented Mail Script to fetch RITM variables. |
| **Expired Medicine Approval** | Created Business Rule to abort on `stage = approved` if medicine expired. |
| **Flow Stuck on Pickup** | Added Wait for condition (2 business days) + auto‑cancel + stock restore. |

---

## 📊 Conclusion & Impact

- **10 user stories** fully delivered across **3 roles**.
- **100% automated workflows** replacing manual, paper‑based processes.
- **SLAs enforced** to improve customer experience and response times.
- **Patient safety guaranteed** by systematically preventing expired stock from reaching customers.

---

## 🙏 Acknowledgments

This project was developed as an **ITI Graduation Project** in 2026, utilizing ServiceNow PDI.

---

## 📬 Contact

**Mariam Mohammed**  
ServiceNow Developer | ITI Graduate  
[LinkedIn Profile] | [GitHub Repository]

---

> *"Transforming pharmacy operations—one automation at a time."* ❤️ Pulse Pharmacy
