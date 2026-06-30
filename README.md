# Project Repository: Zero-Trust Data Pipeline Master Log (Phases 1 & 2)

## 1. Project Charter & System Architecture

### Architectural Objective
To design, implement, and validate an end-to-end Zero-Trust data access pipeline that isolates and secures a highly sensitive corporate file asset named `TOP SECRET` stored inside a dedicated SharePoint Online (SPO) site collection container. 

### Security Design Principles
* **Default-Deny Gating:** All inbound sessions to the data layer are blocked by default via identity policy execution, regardless of tenant-level administrative privileges.
* **Just-In-Time (JIT) Elgibility:** Direct user access to the data tier is completely decoupled from permanent assignment. Identities must dynamically onboard via time-bound, self-service entitlement mechanics that automatically enforce explicit lifespans and automated revoking.
* **Separation of Planes:** Structural configuration changes are cleanly divided between the management plane (unaffected by tenant data-layer isolation rules) and the data plane (actively restricted by identity perimeters).

---

## 2. Phase 1 Deployment Log: Foundation Layer Resource Provisioning

Phase 1 establishes the structural directory objects, storage repositories, and base data payloads within the tenant before configuring any logical access boundaries.

### Step 1: Initialize SharePoint Online Site Collection
* **Resource Name:** `Important-Information-ABC`
* **Resource Type:** SharePoint Online (SPO) Team Site Collection
* **Target Absolute URI:** `https://lab20250106.sharepoint.com/sites/Important-Information-ABC`
* **Administrative Provisioner Account:** `admin@lab20250106.onmicrosoft.com`

### Step 2: Create Target Sensitive Data Object
* **Object Name:** `TOP SECRET`
* **File Specification:** Flat Text Document (`TOP SECRET.txt`)
* **Storage Path Location:** `https://lab20250106.sharepoint.com/sites/Important-Information-ABC/Shared Documents/TOP SECRET.txt`
* **Embedded Validation Payload String:** `"TOP SECRET: Steve Tuschman was born in Brooklyn."`

### Step 3: Provision Directory Test Identities
Two unprivileged cloud user accounts were provisioned in the Microsoft Entra ID directory to act as clean verification vectors for boundary testing without inheriting administrative tenant-level permissions:
1. **Test User Principal Name 1:** `important1@lab20250106.onmicrosoft.com`
2. **Test User Principal Name 2:** `important2@lab20250106.onmicrosoft.com`

---

## 3. Phase 2 Deployment Log: Identity Governance & Isolation Gating

Phase 2 transitions the pipeline from a flat state to an active security boundary by layering automated governance mechanisms, policy enforcement blocks, and explicit data-tier tracking.

### Step 4: Role-Assignable Security Group Configuration
* **Object Class:** Group
* **Display Name:** `Project-TopSecret-Access`
* **Membership Type:** Assigned
* **Role Assignment Capability:** `Microsoft.Entra/rolesAssignableObjects/Groups = True` (This parameter is explicitly configured at instantiation to allow direct directory role binding if required downstream).
* **Baseline Directory State:** Empty (Zero active directory identities).

### Step 5: Microsoft Entra Identity Governance (MEIG) Access Package Deployment
* **Package Name:** `TopSecret-Access-Package`
* **Catalog Assignment:** General Catalog
* **Resource Roles Linkage:** `Project-TopSecret-Access` (Type: Security Group)
* **Target Role Assignment:** Member
* **Request Execution Policy:**
  * **Allowed Requestors:** Specific directory users (`important1@lab20250106.onmicrosoft.com` and `important2@lab20250106.onmicrosoft.com`).
  * **Require Approval:** No (Configured to ensure automated self-service fulfillment upon user token submission).
* **Lifecycle Boundary Settings:**
  * **Assignment Lifespan Maximum:** 1 Hour (Hard temporal expiration enforced automatically by the entitlement management engine).
  * **Access Reviews Lifecycle:** Enabled | Frequency: Weekly | Duration: 1 Day | Reviewer: Steven Tuschman | Non-Response Action: Remove Access.

### Step 6: SharePoint Online Authentication Context Registration
* **Object Class:** Authentication Context
* **Display Name:** `TopSecret-File-Gate`
* **Technical Purpose:** Establishes a unique, logical token identifier in the Microsoft Entra ID schema that bridges the identity policy engine to individual data objects inside external web applications.

### Step 7: Microsoft Entra ID Conditional Access (CA) Policy Architecture
* **Policy Name:** `CA-TopSecret-AuthContext-Gate`
* **Assignment Scope (Users):** Include: **All Users** | Exclude: `Project-TopSecret-Access` (Security Group).
* **Target Resource Scope:** Cloud Apps -> Select: **Authentication Context** -> Target: `TopSecret-File-Gate`.
* **Access Control Enforcement:** **Block Access** (Hard Boolean intercept).

### Step 8: Data Tier Enforcement Binding
The logical Microsoft Entra ID Conditional Access identity gate was physically bound to the target SharePoint Online site collection container by executing the SharePoint Online Management Shell cmdlet from an administrative workstation:


