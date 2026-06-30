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
```powershell
Connect-SPOService -Url "[https://lab20250106-admin.sharepoint.com](https://lab20250106-admin.sharepoint.com)"
Set-SPOSite -Identity "[https://lab20250106.sharepoint.com/sites/Important-Information-ABC](https://lab20250106.sharepoint.com/sites/Important-Information-ABC)" -ConditionalAccessPolicy AuthenticationContext -AuthenticationContextName "TopSecret-File-Gate"

4. Architectural Conflict Analysis & Resolution Logs (Lessons Learned)
During integration testing, two distinct architectural blocks occurred. These technical incidents highlight the critical downstream configurations required for multi-layered security.

Conflict 1: The Data Plane vs. Management Plane 403 Catch-22
Incident Description: While attempting to map access permissions to the local SharePoint Online site container for the security group, the administrator executed the following data-plane cmdlet via the SharePoint Online Management Shell:

PowerShell
Add-SPOUser -Site "[https://lab20250106.sharepoint.com/sites/Important-Information-ABC](https://lab20250106.sharepoint.com/sites/Important-Information-ABC)" -LoginName "Project-TopSecret-Access" -Group "Visitors"
The terminal instantly terminated the execution and returned an error: Add-SPOUser : The remote server returned an error: (403) Forbidden.

Root Cause Evaluation: Because Step 8 had successfully bound the TopSecret-File-Gate Authentication Context directly to the site container data tier (lab20250106.sharepoint.com), the Microsoft Entra ID Conditional Access policy performed exactly as engineered. The policy forces a hard block on all users trying to touch that data layer unless they possess a group claim for Project-TopSecret-Access. Because the administrator identity was not a member of that exclusion group, the identity engine blocked the administrator's own script from accessing the site data layer to write the permission updates.

Resolution Protocol: To bypass the active data-plane policy intercept, the permission modification had to be re-routed through the management plane, which operates completely above individual site collection Conditional Access rules:

Navigated directly to the web-based SharePoint Admin Center (https://lab20250106-admin.sharepoint.com).

Selected Sites -> Active Sites -> Important-Information-ABC -> Membership tab.

Appended the Project-TopSecret-Access security group directly to the Site Members list via the portal UI. This successfully committed the group to the application's internal Access Control List (ACL) without interacting with the blocked data-plane endpoint.

Conflict 2: Operating System Token Bleed-Through & WAM Intercepts
Incident Description: During client-side verification on a Windows workstation, initiating an InPrivate or Incognito browser session failed to present a blank login prompt. Instead, the browser automatically authenticated as the Global Administrator (admin@lab20250106.onmicrosoft.com) and triggered the default-deny block page, preventing testers from logging in as important1.

Root Cause Evaluation: Modern Windows web browsers exhibit deep operating system integration through the Windows Account Manager (WAM) subsystem. When attempting to authenticate to a Microsoft tenant, the browser polls the local Windows registry for active workplace or school credentials and leaks the Primary Refresh Token (PRT) directly to login.microsoftonline.com. This automatic identity injection bypasses standard browser cookie isolation layers and forces single sign-on using the active Windows profile.

Resolution Protocol: Absolute process isolation was achieved by moving the testing boundary onto a non-Windows asset (macOS notebook). Because the macOS environment lacks the WAM registry-polling subsystem, opening an Incognito window guaranteed a 100% blank identity presentation layer, allowing the manual entry of the test user principal name.

5. Functional Verification Protocol & Live Validation Logs
Token Cache Mitigation
Cloud infrastructure replication scales across a delayed window (typically 15 to 45 minutes). When a security policy triggers a block, SharePoint Online aggressively caches that failure state inside the user's active session cookie. To bypass this latency window during active validation, a manual site-level sign-out command was appended to the URI to flush the local application cache and force an immediate token claims refresh:

Plaintext
[https://lab20250106.sharepoint.com/sites/Important-Information-ABC/_layouts/15/SignOut.aspx](https://lab20250106.sharepoint.com/sites/Important-Information-ABC/_layouts/15/SignOut.aspx)
Functional Validation Scenario Log
Scenario A: Default Deny Identity Gate (VERIFIED)

Active Context: admin@lab20250106.onmicrosoft.com

Action: Navigated directly to the target site collection URL.

Observed Output: The Microsoft identity engine intercepted the session and returned an absolute block page: "Your sign-in was successful, but you do not have permission to access this resource."

Architectural Logic: The identity matched the inclusion parameter of CA-TopSecret-AuthContext-Gate but lacked the Project-TopSecret-Access group claim exclusion, resulting in a successful hard block.

Scenario B: Automated Entitlement Onboarding (VERIFIED)

Active Context: important1@lab20250106.onmicrosoft.com (Executed via isolated macOS session).

Action: Logged into the My Access portal (https://myaccess.microsoft.com/lab20250106.onmicrosoft.com) and requested the TopSecret-Access-Package.

Observed Output: Request bypassed approval loops and transitioned from Processing to Delivered within the portal UI.

Architectural Logic: The MEIG governance engine successfully performed a directory write operation, appending the user identity directly into the Project-TopSecret-Access security group.

Scenario C: Data Access Verification (VERIFIED)

Active Context: important1@lab20250106.onmicrosoft.com

Action: Executed the manual SharePoint token cache geographic purge sequence, then re-authenticated directly to the root site collection URL.

Observed Output: Bypassed all Conditional Access blocks, passed the SharePoint Online internal ACL permission check, loaded the document library, and opened the target file object.

Verified Project Payload Signature
Upon completing the verification loop, the test identity successfully traversed the fully configured Zero-Trust pipeline, opening the sensitive text document and extracting the validation payload string:

"TOP SECRET: Steve Tuschman was born in Brooklyn."

Phase 2 is officially closed out as 100% functional, verified, and structurally sound.
