# Entra ID Identity & Access Governance Lab

Eight scenarios covering identity, access control, and governance in a live Microsoft 365 / Entra ID tenant, built for a fictional logistics and supply chain company. Unlike a mailbox/mail-flow admin lab, this project focuses on how identity is provisioned, secured, and governed at the tenant level — dynamic groups, Conditional Access, emergency access, self-service password reset, and (in later scenarios) Privileged Identity Management, scoped delegation, access reviews, and enterprise app SSO.

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

## Scenarios (4/8 completed)

| # | Scenario | Area |
|---|---|---|
| 01 | [Dynamic groups & group-based licensing](#scenario-01--dynamic-groups--group-based-licensing) | Identity provisioning |
| 02 | [Block legacy authentication](#scenario-02--block-legacy-authentication-conditional-access) | Conditional Access |
| 03 | [Break-glass admin account](#scenario-03--break-glass-admin-account) | Emergency access |
| 04 | [SSPR + Conditional Access MFA + locked-out user ticket](#scenario-04--sspr--conditional-access-mfa--locked-out-user-ticket) | Self-service reset / MFA |

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
Checked Sign-in logs for a normal browser-based sign-in and confirmed the policy's Conditional Access result showed **Not applied** — correctly confirming the policy does not affect modern authentication and only targets legacy client types, as designed. (Note: since virtually no modern client still uses legacy authentication protocols, and Exchange Online has legacy auth disabled tenant-wide by default, live blocked-sign-in evidence isn't practically reproducible in this environment — the policy configuration and this "not applied to modern auth" confirmation serve as the evidence instead.)

![Sign-in logs showing Conditional Access result: Not Applied](screenshots/scenario02-05-signin-logs-not-applied.png)

**Outcome:** Legacy authentication is blocked tenant-wide for any client attempting it, while modern authentication (standard browser and app sign-ins) is confirmed unaffected.

---

## Scenario 03 — Break-Glass Admin Account

**Goal:** Provision an emergency access account that remains usable even if the tenant's normal Conditional Access or MFA systems fail or are misconfigured.

### Build
Created a dedicated Global Administrator account, **Emergency Access Admin** (`breakglass@cloudpulsesolutions.onmicrosoft.com`), with a long, randomly generated password and no license assigned — kept to a minimal footprint with no mailbox or standing services attached.

![Break-glass account basic info](screenshots/scenario03-00-breakglass-basic-info.png)
![Assigning the Global Administrator role](screenshots/scenario03-01-assigning-role.png)
![Role assignment confirmed](screenshots/scenario03-02-role-assigned-global-admin.png)
![Review + create](screenshots/scenario03-03-review-create.png)
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
![SSPR enabled](screenshots/scenario04-01-sspr-enabled.png)
![SSPR requiring 2 methods](screenshots/scenario04-02-sspr-two-methods-required.png)

### Build — Conditional Access requiring MFA
Created a new Conditional Access policy, **Require MFA For All Users**, applying to all users (with the admin and break-glass accounts excluded, learned from the previous scenario), Grant control set to **Require multifactor authentication**. Rolled out in Report-only first, then switched to On.

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

### Verify — re-registration and sign-in
Signed in as Charles again. With no methods on file, he was walked through registering Microsoft Authenticator and a phone number from scratch.

![Charles re-registering Microsoft Authenticator](screenshots/scenario04-08-charles-reregister-authenticator.png)
![Authenticator registration complete](screenshots/scenario04-09-charles-authenticator-complete.png)
![Setting up SMS as the second method](screenshots/scenario04-10-charles-sms-setup.png)
![SMS method added](screenshots/scenario04-11-charles-sms-added.png)
![Charles signed in successfully after MFA setup](screenshots/scenario04-12-charles-signed-in.png)

### Bonus verify — SSPR works end-to-end
To confirm the SSPR configuration from earlier was fully functional (not just configured), walked through the self-service "Get back into your account" flow as Charles using his newly registered methods.

![SSPR step 1 — phone verification](screenshots/scenario04-13-sspr-test-step1.png)
![SSPR step 2 — authenticator app verification](screenshots/scenario04-14-sspr-test-step2.png)
![Setting a new password](screenshots/scenario04-15-sspr-new-password.png)
![Password reset confirmed](screenshots/scenario04-16-sspr-reset-complete.png)
![Signed in successfully with the new password](screenshots/scenario04-17-signin-after-sspr.png)

**Root cause:** Charles had no way to complete MFA after losing his registered device, and had no fallback registered method for SSPR to fall back on either. **Resolution:** Clearing his registered methods as admin let him re-register cleanly, and testing SSPR afterward confirmed the whole authentication chain (registration → CA-enforced MFA → self-service reset) was working end-to-end, not just individually configured.

---

## Notes on Tooling

All configuration in this lab was performed through the Microsoft 365 admin center and the Entra admin center GUIs. In a production environment, this configuration is more commonly deployed via Microsoft Graph PowerShell for consistency and repeatability — for example:

```powershell
# Scenario 01 equivalent — dynamic group with membership rule
New-MgGroup -DisplayName "IT Department" -MailEnabled:$false -SecurityEnabled `
  -GroupTypes "DynamicMembership" -MembershipRule '(user.department -eq "IT")' `
  -MembershipRuleProcessingState "On"

# Scenario 02 equivalent — Conditional Access policy blocking legacy auth (via Graph)
New-MgIdentityConditionalAccessPolicy -DisplayName "Block Legacy Authentication" `
  -State "enabled" -Conditions @{ ClientAppTypes = @("exchangeActiveSync","other") } `
  -GrantControls @{ Operator = "OR"; BuiltInControls = @("block") }

# Scenario 03 equivalent — creating the break-glass account
New-MgUser -DisplayName "Emergency Access Admin" -UserPrincipalName "breakglass@cloudpulsesolutions.onmicrosoft.com" `
  -PasswordProfile @{ Password = "<securely-generated>"; ForceChangePasswordNextSignIn = $false } -AccountEnabled

# Scenario 04 equivalent — clearing a user's authentication methods
Get-MgUserAuthenticationMethod -UserId "cspence@cloudpulsesolutions.onmicrosoft.com" |
  ForEach-Object { Remove-MgUserAuthenticationMethod -UserId "cspence@cloudpulsesolutions.onmicrosoft.com" -AuthenticationMethodId $_.Id }
```

## Planned Additions

Scenarios 05–08 are planned for this lab: Privileged Identity Management (PIM) for time-boxed role activation, Administrative Units for department-scoped delegation, Access Reviews for recurring governance of privileged access, and Enterprise Application/SSO integration.
