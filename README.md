# Entra ID Identity & Access Governance Lab

Seven scenarios covering identity, access control, and governance in a live Microsoft 365 / Entra ID tenant, built for a fictional logistics and supply chain company. This project focuses on how identity is provisioned, secured, and governed at the tenant level — dynamic groups, Conditional Access, emergency access, self-service password reset, Privileged Identity Management, scoped delegation, and enterprise app SSO.

Each scenario is documented as a build → configure → verify walkthrough, and where applicable, as a reproduce → diagnose → fix → verify support ticket, consistent with a real Tier 1/2 identity administration workload.

## Environment

| | |
|---|---|
| **Company** | CloudPulse Solutions (fictional, ~350 employees) — logistics and supply chain management |
| **Tenant** | `cloudpulsesolutions.onmicrosoft.com` (Office 365 E5 trial + Enterprise Mobility + Security E5 trial) |
| **Admin tools used** | Microsoft 365 admin center, Entra admin center, Conditional Access, Identity Protection |
| **Departments** | IT, Finance, Operations |
| **Users** | Derek Chan (IT Support Specialist), Priya Nair (IT Systems Administrator), Janice Carey (Finance Manager), Charles Spence (Accounts Payable Specialist), Mitchell Godfrey (Operations Coordinator), Grace Okafor (Logistics Analyst) |
| **Admin account** | Rooble Dahir — Global Administrator |
| **Emergency access account** | Emergency Access Admin (`breakglass@cloudpulsesolutions.onmicrosoft.com`) — Global Administrator, excluded from Conditional Access |

## Scenarios 

| # | Scenario | Area |
|---|---|---|
| 01 | [Dynamic groups & group-based licensing](#scenario-01--dynamic-groups--group-based-licensing) | Identity provisioning |
| 02 | [Block legacy authentication](#scenario-02--block-legacy-authentication-conditional-access) | Conditional Access |
| 03 | [Break-glass admin account](#scenario-03--break-glass-admin-account) | Emergency access |
| 04 | [SSPR + Conditional Access MFA + locked-out user ticket](#scenario-04--sspr--conditional-access-mfa--locked-out-user-ticket) | Self-service reset / MFA |
| 05 | [Privileged Identity Management (PIM)](#scenario-05--privileged-identity-management-pim) | Just-in-time access |
| 06 | [Administrative Units — scoped delegation](#scenario-06--administrative-units--scoped-delegation) | Least privilege |
| 07 | [Enterprise Application & SSO](#scenario-07--enterprise-application--sso) | App integration |
| 08 | [Key Takeaways](#key-takeaways) | What I learned |

---

## Scenario 01 — Dynamic Groups & Group-Based Licensing

**Goal:** Provision departmental security groups whose membership updates automatically based on each user's Department attribute, and license users at the group level instead of assigning licenses individually.

### Build
Created three security groups with **Dynamic User** membership — IT Department, Finance Department, and Operations — each with a dynamic membership rule matching the user's Department property (e.g. `user.department -eq "IT"`).

![Groups created](screenshots/scenario01-00-groups-created.png)

### Verify — membership populated automatically
After creating the 6 users (2 per department) with their Department attribute set, each group's membership populated on its own with no manual additions.

![IT Department members](screenshots/scenario01-01-members-it.png)
![Finance Department members](screenshots/scenario01-02-members-finance.png)
![Operations members](screenshots/scenario01-03-members-operations.png)
![All 6 users created](screenshots/scenario01-04-all-users-created.png)

### Configure — group-based licensing
Rather than assigning Office 365 E5 and Enterprise Mobility + Security E5 to each of the 6 users individually, both licenses were assigned directly to the three groups. Any current or future member of a group is automatically licensed.

![Assigning EMS E5 to the groups](screenshots/scenario01-05-assigning-ems-licenses.png)
![EMS E5 assigned to all three groups](screenshots/scenario01-06-ems-licenses-assigned.png)
![Assigning Office 365 E5 to the groups](screenshots/scenario01-07-assigning-o365-licenses.png)
![Office 365 E5 assigned to all three groups](screenshots/scenario01-08-o365-licenses-assigned.png)

### Verify — licenses flowed down to users
Checked the Active Users list to confirm all 6 department users received both licenses automatically through their group membership, with no per-user license assignment required.

![All 6 users show both licenses](screenshots/scenario01-09-licenses-flowed-down-final.png)

**Outcome:** Departmental group membership and licensing are now fully automatic — adding a new hire to a department going forward requires only setting their Department attribute; group membership and licensing follow without any manual admin steps.

---

## Scenario 02 — Block Legacy Authentication (Conditional Access)

**Goal:** Build a Conditional Access policy blocking legacy/basic authentication protocols tenant-wide — a widely recommended baseline policy, since legacy protocols can't support MFA and are a common target for credential-stuffing attacks.

### Build
Created a new Conditional Access policy, **Block Legacy Authentication**, targeting all cloud apps and scoped to legacy client types only (Exchange ActiveSync clients and Other clients), with a Grant control of **Block access**.

![Client apps configuration — legacy client types only](screenshots/scenario02-02-client-apps-config.png)
![Grant control set to Block access](screenshots/scenario02-03-grant-block-access.png)

### Safety net
Before enabling anything, the admin account (Rooble Dahir) was excluded from the policy to prevent a lockout if the configuration behaved unexpectedly.

![Admin account excluded](screenshots/scenario02-01-admin-excluded.png)

### Rollout — Report-only first
Following standard Conditional Access rollout practice, the policy was enabled in **Report-only** mode first rather than switched straight to enforced, so its impact could be reviewed before anyone was actually blocked.

![Policy created in Report-only mode](screenshots/scenario02-00-policy-report-only.png)

After reviewing, the policy was switched to **On**.

![Policy fully configured and turned on](screenshots/scenario02-04-policy-turned-on.png)

### Verify
Checked Sign-in logs for a normal browser-based sign-in and confirmed the policy's Conditional Access result showed **Not applied** — correctly confirming the policy does not affect modern authentication and only targets legacy client types, as designed. 

![Sign-in logs showing Conditional Access result: Not Applied](screenshots/scenario02-05-signin-logs-not-applied.png)

**Outcome:** Legacy authentication is blocked tenant-wide for any client attempting it, while modern authentication (standard browser and app sign-ins) is confirmed unaffected.

---

## Scenario 03 — Break-Glass Admin Account

**Goal:** Provision an emergency access account that remains usable even if the tenant's normal Conditional Access or MFA systems fail or are misconfigured.

### Build
Created a dedicated Global Administrator account, **Emergency Access Admin** (`breakglass@cloudpulsesolutions.onmicrosoft.com`), with a long, randomly generated password and no license assigned — kept to a minimal footprint with no mailbox or standing services attached.

![Break-glass account basic info](screenshots/scenario03-00-breakglass-basic-info.png)

Assigning the Global Administrator Role

![Assigning the Global Administrator role](screenshots/scenario03-01-assigning-role.png)
![Role assignment confirmed](screenshots/scenario03-02-role-assigned-global-admin.png)
![Review + create](screenshots/scenario03-03-review-create.png)

The Break-Glass Admin account has been created and now appears in Users.

![Account created](screenshots/scenario03-04-account-created.png)

### Configure — excluded from Conditional Access
Since the entire purpose of a break-glass account is to work even when the tenant's normal auth controls don't, it must sit outside those controls rather than be protected by them. Added the account to the Exclude list on the Block Legacy Authentication policy from Scenario 02, alongside the admin account.

![Break-glass account excluded from the legacy-auth policy](screenshots/scenario03-05-excluded-from-legacy-auth-policy.png)
![Policy now shows 2 excluded users](screenshots/scenario03-06-policy-shows-2-excluded.png)

### Hardening without blocking access
Since Conditional Access can't be the protection layer for this account, a second, narrowly-scoped policy was built to reduce its standing exposure without risking a lockout: **Break-Glass Account: Session Controls**, applying only to this account, with no Grant control (nothing is blocked or required) but Session controls set to force re-authentication **every time** and disable persistent browser sessions.

![Session controls policy created](screenshots/scenario03-07-session-policy-created.png)
![Session controls configuration](screenshots/scenario03-08-session-controls-config.png)

### Verify
Signed in once as the break-glass account to confirm it authenticates successfully and bypasses the legacy-auth policy as expected.

![Signed in as the break-glass account](screenshots/scenario03-09-signed-in-as-breakglass.png)

Confirmed the sign-in event is visible and traceable in Sign-in logs — critical for a break-glass account, since credential secrecy and monitoring (not access policy) are its real protection model.

![Break-glass sign-in shown in logs](screenshots/scenario03-10-shown-in-signin-logs.png)

**Outcome:** A working emergency access account exists, correctly excluded from Conditional Access, hardened against lingering/idle sessions, and monitorable via sign-in logs. Its ongoing governance (periodic review of whether it's still needed and appropriately scoped) is picked up later in Scenario 07 (Access Reviews).

---

## Scenario 04 — SSPR + Conditional Access MFA + Locked-Out User Ticket

**Goal:** Enable self-service password reset and enforce MFA tenant-wide via Conditional Access, then resolve a realistic ticket where a user has lost their registered MFA device and is locked out.

### Build — authentication methods & SSPR
Enabled Microsoft Authenticator and SMS as available authentication methods tenant-wide, then enabled SSPR requiring **2 methods** to reset a password.

![Authentication methods enabled (Authenticator + SMS)](screenshots/scenario04-00-auth-methods-enabled.png)

Enabling SSPR

![SSPR enabled](screenshots/scenario04-01-sspr-enabled.png)

Note: Requiring 2 methods instead of 1 is a much better practice. If an attacker were to gain one method of authentication, they could reset your password leading to your account being compromised.

![SSPR requiring 2 methods](screenshots/scenario04-02-sspr-two-methods-required.png)

### Build — Conditional Access requiring MFA
Created a new Conditional Access policy, **Require MFA For All Users**, applying to all users (with the admin and break-glass accounts excluded, learned from the previous scenario), and set the Grant control to **Require multifactor authentication**. Rolled out in Report-only first, then switched to On.

![Grant control requiring MFA](screenshots/scenario04-03-ca-policy-grant-mfa.png)
![Policy switched from Report-only to On](screenshots/scenario04-04-ca-policy-on.png)

### Reproduce — user prompted for MFA
Signed in as Charles Spence (who had not yet registered MFA) and confirmed he was prompted to add a sign-in method, as required by the new Conditional Access policy.

![Charles prompted to add a sign-in method](screenshots/scenario04-05-charles-prompted-mfa.png)

### Ticket scenario
**Charles Spence is locked out.** He previously registered MFA on his personal phone but lost the device and can no longer approve sign-in prompts or complete verification — and can't self-serve through SSPR either, since he can't complete a second verification method.

Checked his registered authentication methods (Microsoft Authenticator + phone number) to confirm what was on file before making any change.

![Charles's authentication methods before the fix](screenshots/scenario04-06-charles-methods-before.png)

### Fix
As admin, deleted both of Charles's registered authentication methods, clearing the way for him to re-register fresh on a new device.

![Both methods deleted — no usable methods remain](screenshots/scenario04-07-charles-methods-deleted.png)

### Verify — Re-registration and sign-in
Signed in as Charles again. With no methods on file, he was walked through registering Microsoft Authenticator and a phone number from scratch.

![Authenticator registration complete](screenshots/scenario04-08-charles-authenticator-complete.png)
![SMS method added](screenshots/scenario04-09-charles-sms-added.png)
![Charles signed in successfully after MFA setup](screenshots/scenario04-10-charles-signed-in.png)

### Bonus verify — SSPR works end-to-end
To confirm the SSPR configuration from earlier was fully functional (not just configured), walked through the self-service "Get back into your account" flow as Charles using his newly registered methods.

![SSPR step 1 — phone verification](screenshots/scenario04-11-sspr-test-step1.png)
![SSPR step 2 — authenticator app verification](screenshots/scenario04-12-sspr-test-step2.png)

After completing both methods (SMS and Auth App), Charles can now create a new password.

![Setting a new password](screenshots/scenario04-13-sspr-new-password.png)
![Password reset confirmed](screenshots/scenario04-14-sspr-reset-complete.png)

Charles has now completed SSPR and is signed in.

![Signed in successfully with the new password](screenshots/scenario04-15-signin-after-sspr.png)

**Root cause:** Charles had no way to complete MFA after losing his registered device, and had no fallback registered method for SSPR to fall back on either. **Resolution:** Clearing his registered methods as admin let him re-register cleanly, and testing SSPR afterward confirmed the whole authentication chain (registration → CA-enforced MFA → self-service reset) was working end-to-end, not just individually configured.

---

## Scenario 05 — Privileged Identity Management (PIM)

**Goal:** Replace a permanent, standing role assignment with just-in-time access — a user is eligible for an admin role but holds no actual privilege until they deliberately activate it, with a justification, for a limited window.

### Build
In Privileged Identity Management (Microsoft Entra roles), located the **Helpdesk Administrator** role and added Priya Nair (IT Systems Administrator) as **Eligible** rather than Active.

![PIM overview](screenshots/scenario05-00-pim-overview.png)
![Helpdesk Administrator role in PIM](screenshots/scenario05-01-helpdesk-admin-role.png)
![Adding Priya as a new assignment](screenshots/scenario05-02-adding-priya.png)

Set the eligibility window to 6 months (time-bound rather than permanent) — separate from the activation duration used later — so the eligibility itself is subject to periodic renewal rather than persisting indefinitely unreviewed.

![Eligible assignment with a 6-month window](screenshots/scenario05-03-eligible-6month-window.png)
![Assignment confirmed](screenshots/scenario05-04-assigned-confirmation.png)

### Verify — no standing access
Signed in as Priya and confirmed Helpdesk Administrator appeared under **Eligible assignments**, not Active. She has zero privilege until she activates it herself.

![Priya sees the role as eligible only](screenshots/scenario05-05-priya-sees-eligible.png)

### Activate with justification
Priya activated the role for 1 hour to reset a Finance user's lost MFA methods, which SSPR alone can't do and genuinely requires elevated access.

![Activation request with justification entered](screenshots/scenario05-06-priya-activation-justification.png)

(Her role is now **Active**)

![Priya's role now shows Active](screenshots/scenario05-08-priya-active-assignment.png)

### Verify — audit trail
Checked PIM's audit history in the admin account and confirmed the activation, including Priya's full justification text and additional details.

![Audit history showing the activation and justification](screenshots/scenario05-09-audit-history-justification.png)

**Outcome:** Priya holds no standing administrative privilege day-to-day. Elevated access exists only for as long as a specific, justified task requires it, is fully logged, and reverts automatically (or can be ended early) with no manual cleanup required. This is picked up again directly in Scenario 06, where Priya's tenant-wide eligibility is compared against a department-scoped assignment.

---

## Scenario 06 — Administrative Units: Scoped Delegation

**Goal:** Prove that a **restricted Administrative Unit (AU)** enforces a real access boundary around Finance, one that even a tenant-wide holder of the same admin role cannot cross without a scope-specific assignment.

### Build — Finance AU
Created a **Finance AU**, set to **Restricted management**, and added Janice Carey and Charles Spence as members.

![Creating the Finance AU (Restricted = Yes)](screenshots/scenario06-00-creating-finance-au.png)

Finance AU was successfully created.

![Finance AU created](screenshots/scenario06-01-finance-au-created.png)

For this AU, I'll assign Derek Chan as the Helpdesk Administrator.

![Derek Chan assigned Helpdesk Administrator scoped to the AU](screenshots/scenario06-02-assigning-derek-scoped-role.png)

Showing Janice and Charles in the created Finance AU.

![Janice and Charles added as AU members](screenshots/scenario06-03-finance-members-added.png)

### Verify — the scoped role works
Signed in as Derek Chan, who should be able to successfully reset Charles Spence's password.

![Derek successfully resets Charles's password](screenshots/scenario06-04-derek-resets-charles-succeeds.png)

And it worked as expected.

### The real comparison: same role, different scope
Rather than only showing the boundary from one angle, this scenario directly compares two people holding the identical role name — **Helpdesk Administrator** — at the same time, differing only in how that role is scoped:

- **Derek Chan:** Helpdesk Administrator scoped **to the Finance AU specifically**
- **Priya Nair:** Helpdesk Administrator activated **tenant-wide** via PIM (Scenario 05)

Priya activated her tenant-wide eligible role through PIM.

![Priya activates her tenant-wide role](screenshots/scenario06-05-priya-activates-tenant-wide-role.png)

Despite holding an **Active** Helpdesk Administrator assignment at that exact moment, Priya was blocked from resetting Charles Spence's password — the same action Derek performed successfully — because her assignment is scoped to the whole tenant, not to the Finance AU.

![Priya blocked from resetting Charles's password despite an active role](screenshots/scenario06-06-priya-blocked-on-finance-user.png)

For contrast, Priya's same tenant-wide role works normally on a user **outside** the Finance AU. To test, Priya tried to reset Mitchell Godfrey's password, which worked, confirming the restriction is specific to Finance AU membership and not a general permissions failure.

![Priya successfully resets a non-Finance user's password](screenshots/scenario06-07-priya-succeeds-outside-finance.png)

**Root cause/design point:** Restricted Administrative Units enforce scope over role name. Holding "Helpdesk Administrator" tenant-wide is not sufficient to manage a restricted AU's members — only an assignment scoped directly to that AU grants access, even for another Helpdesk Administrator holding the identical role elsewhere.

**Outcome:** A precise, working demonstration of least-privilege enforcement — the boundary holds not because of who has "more" access, but because of how that access is scoped.

---

## Scenario 07 — Enterprise Application & SSO

**Goal:** Integrate a real SaaS application with the tenant via SAML single sign-on, assign access through a security group, and validate a genuine end-to-end login — not just a configuration screen.

### Build — Add the gallery app
Added **Salesforce** from the Microsoft Entra App Gallery as a new enterprise application.

![Adding Salesforce from the app gallery](screenshots/scenario07-00-adding-salesforce-gallery-app.png)

Configured initial SAML settings with placeholder values, since no real Salesforce org existed yet at this point.

![Initial SAML configuration with placeholder values](screenshots/scenario07-01-saml-config-placeholder.png)

### Assign access via group
Assigned the **Operations** security group (from Scenario 01) to the application, consistent with the group-based access pattern used throughout this lab, along with a default app role.

![Operations group assigned to the app](screenshots/scenario07-02-operations-group-assigned.png)
![Assigning the app role](screenshots/scenario07-03-assigning-app-role.png)

### Creating a Salesforce Org
Signed up for a free Salesforce Developer Edition org to complete a genuine, live SAML handshake rather than an unverified one.

![Salesforce Developer org created](screenshots/scenario07-04-salesforce-dev-org-created.png)

In Salesforce, created a matching SAML Single Sign-On configuration, importing the real Issuer, Identity Provider Login URL, and signing certificate from Entra.

![Salesforce SSO settings](screenshots/scenario07-05-salesforce-sso-settings.png)

Filled out all necessary configurations.

![Salesforce SAML configuration form](screenshots/scenario07-06-salesforce-saml-config-form.png)

SSO settings are configured and saved.

![Salesforce SAML configuration saved](screenshots/scenario07-07-salesforce-saml-saved.png)

Updated Entra's Basic SAML Configuration with the real Entity ID and ACS URL generated by Salesforce, replacing the earlier placeholder values.

![Entra updated with real Salesforce values](screenshots/scenario07-08-entra-updated-with-real-values.png)

### Provisioning: Creating the user
Created a dedicated Salesforce user for Grace Okafor with a username matching her Entra UPN exactly, since the SSO configuration matches identity by username.

![Creating a matching Salesforce user for Grace](screenshots/scenario07-09-grace-salesforce-user-created.png)

### Verify — Conifrming that SSO Works
Signed in as Grace Okafor via `myapps.microsoft.com` and clicked the Salesforce tile. (Note: Salesforce app now pops up on the dashboard!)

![Signing in as Grace to test SSO](screenshots/scenario07-10-signing-in-as-grace.png)

The SAML handshake completed successfully, logging Grace into her own distinct Salesforce account, confirmed by her name and username in the top-right of Salesforce.

![SSO succeeds — Grace Okafor logged into her own Salesforce account](screenshots/scenario07-11-sso-success-grace-logged-in.png)

**Outcome:** A complete, live SSO integration — real IdP (Entra) and real SP (Salesforce), group-based access assignment, and a verified login as a specific, distinct user.

---

## Key Takeaways
 
A few things came up while building this that weren't part of the original plan, but ended up being some of the most useful parts of the process:
 
- **MFA in Entra ID isn't controlled by one setting** I spent a while confused about why the break-glass account kept getting hit with an MFA prompt even though it was excluded from every Conditional Access policy I'd built. Turned out Conditional Access, Security Defaults, legacy per-user MFA, and Identity Protection's MFA registration policy can all independently force MFA, and none of them tell you the others exist. 
- **A restricted Administrative Unit will lock out even a Global Admin.** I found this out the hard way, trying to assign a license directly to a Finance user and getting told I didn't have permission, as the Global Administrator. Turns out that's the AU working exactly as intended. The fix was licensing through the group instead of the user directly, which also made it click why I'd set up group-based licensing back in Scenario 01 in the first place.
- **A correct SAML config isn't the same as a working SSO login.** My first real test signed a test user straight into my own Salesforce account instead of theirs, because there was no Salesforce account for them to sign into at all. SSO can only log someone into an identity that already exists on the other end, or one that gets auto-created (Such a silly mistake). I ended up manually provisioning a matching user just to prove the whole thing actually worked end to end, not just that the config looked right.
