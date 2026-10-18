# Lab 7: IAM Automation with Microsoft Graph PowerShell

## Overview
This lab automated common identity and access management tasks with Microsoft Graph PowerShell instead of performing each one by hand in the Microsoft Entra admin center. I used it to:

- Identify stale enabled accounts
- Inventory external guest users
- Report active privileged role assignments
- Dry-run user onboarding before making changes
- Create multiple users from a CSV file
- Disable a departing user and issue a session-revocation request
- Check the session revocation in the Microsoft Entra audit log
- Review the delegated Microsoft Graph permissions consented in the tenant

The lab used **delegated authentication**, meaning the automation ran as a signed-in administrator. It did not use unattended app-only authentication.

## Lab Goals
- Microsoft Graph PowerShell administration
- IAM reporting and lifecycle (joiner and leaver) automation
- Least-privilege scope selection and permission review
- Dry-run safeguards before write operations
- Audit evidence for an automated identity action
- Delegated vs. application permission concepts

---

# Phase 1: Connect to Microsoft Graph
I connected from PowerShell 7 using delegated authentication. For the reporting tasks I requested read-oriented scopes:

- `User.Read.All`
- `AuditLog.Read.All`
- `RoleManagement.Read.Directory`

`Get-MgContext` confirmed:
- Authentication type: **Delegated**
- Application: **Microsoft Graph Command Line Tools**
- Context scope: **Process**
- Signed-in administrator: Lab Admin

An important lesson emerged during the lab: delegated permissions consented earlier remained visible on the Microsoft Graph Command Line Tools enterprise application even when a later connection requested fewer scopes. This lab therefore separates **scopes requested for a task** from **permissions already consented to the shared Graph client in the tenant**.

![Get-MgContext showing the delegated Graph session](screenshots/72-graph-context.png)
*Screenshot 72: `Get-MgContext` output.*

---

# Phase 2: Stale Account Reporting
I built a report of enabled accounts with no successful sign-in for at least **90 days**, using `LastSuccessfulSignInDateTime` rather than the last sign-in attempt. An account that had never signed in was flagged only if the account itself was older than the 90-day threshold, so new accounts are not reported as stale.

The report returned several older lab accounts, which was expected in a tenant used for several months. It also listed the Emergency Break-Glass account, which reinforced a key point:

> A stale-account report is investigation input, not an automatic disablement list.

Emergency-access accounts may intentionally show little or no routine sign-in activity. Results were exported to CSV and reviewed in a spreadsheet.

![Stale account report console output](screenshots/73-stale-report-console.png)
*Screenshot 73: Console output of the stale-account report.*

![Stale account report CSV](screenshots/74-stale-report-csv.png)
*Screenshot 74: The exported report opened in a spreadsheet.*

---

# Phase 3: Guest and Privileged Access Inventory

## Guest Inventory
To demonstrate external identity reporting, I created two controlled guest accounts. One guest redeemed the invitation and the other remained unredeemed. Graph reported:

- **Olivia Grant (Finance Vendor):** Accepted
- **Daniel Kim (Marketing Partner):** PendingAcceptance

This shows Graph can report not only guest identities but also their external-user state.

![Guest inventory](screenshots/75-guest-inventory.png)
*Screenshot 75: Guest inventory with invitation state.*

## Active Privileged Role Report
I queried directory role assignments through Microsoft Graph. The report returned active assignments including Application Administrator, Global Administrator, and Helpdesk Administrator, along with the principal holding each role.

This query reports **current active directory role assignments**. It does not enumerate PIM eligibility. Eligible assignments are modeled separately and would need to be queried through the role eligibility and schedule APIs.

![Privileged role report](screenshots/76-privileged-roles.png)
*Screenshot 76: Active directory role assignments by role and principal.*

---

# Phase 4: Joiner Automation
I created a CSV with three fictional employees: Jordan Miles (Finance), Ava Patel (Marketing), and Noah Bennett (IT).

For this phase I disconnected from the read-only session and requested `User.Create` and `User.Read.All`. Microsoft documents `User.Create` as the least-privileged permission for creating users. However, Microsoft Graph Command Line Tools already had `User.ReadWrite.All` consented in the tenant, so this run should **not** be read as an isolated test proving that only `User.Create` authorized the operation.

Before creating anything, I ran the onboarding logic in **dry-run mode**. The script checked whether each UPN already existed and displayed what it would create without modifying Microsoft Entra ID. All three passed.

![Joiner dry-run](screenshots/77-joiner-dryrun.png)
*Screenshot 77: Dry-run output showing the users that would be created.*

After validating the dry run, I executed the onboarding. Graph created all three users with their Department attributes. Temporary passwords were generated in memory and never printed or captured.

![Created users](screenshots/78-joiner-created.png)
*Screenshot 78: The new users in Microsoft Entra ID.*

---

# Phase 5: Leaver Automation
I used the newly created **Jordan Miles** account as the lab leaver. I reconnected requesting only the scopes this task needed. Microsoft documents `User.EnableDisableAccount.All` with `User.Read.All` as the least-privileged combination for changing `accountEnabled`, and `User.RevokeSessions.All` as the least-privileged delegated permission for `revokeSignInSessions`. As in Phase 4, broader permissions previously consented to the shared client were still present.

The automation disabled the account, issued a session-revocation request, and queried the account again. The output showed:

- `DisplayName : Jordan Miles`
- `AccountEnabled : False`
- `SessionsRevoked : True`

Jordan was a newly created account that may not have had an active session. This therefore demonstrates successful execution of the revocation operation, not proof that a live session was terminated. Microsoft also notes there can be a short delay before revocation takes effect.

This connects to the session-revocation concepts in Lab 6.

![Leaver result](screenshots/79-leaver-result.png)
*Screenshot 79: Account disabled and session revocation request completed.*

---

# Phase 6: Audit Log Verification
I checked the Microsoft Entra audit log for the session revocation. It showed **Update StsRefreshTokenValidFrom Timestamp**, identifying Jordan Miles as the target and the signed-in administrator as the initiating identity, with a successful status. The Modified Properties view showed the change to the user's refresh-token validity timestamp.

This is audit evidence that the revocation request was processed. The screenshots document this event; the user-creation and account-disable events are not included as evidence here.

![Audit event activity](screenshots/80A-audit-log-graph-activity.png)
*Screenshot 80A: The audit activity.*

![Audit event target](screenshots/80B-audit-log-graph-target.png)
*Screenshot 80B: The target and initiating identity.*

![Audit event modified properties](screenshots/80C-audit-log-graph-modified-properties.png)
*Screenshot 80C: Modified Properties showing the timestamp change.*

---

# Phase 7: Delegated Permission Review
I reviewed **Enterprise applications → Microsoft Graph Command Line Tools → Permissions**. Delegated permissions granted through admin consent included `User.Read.All`, `User.Create`, `User.EnableDisableAccount.All`, `User.RevokeSessions.All`, `AuditLog.Read.All`, `RoleManagement.Read.Directory`, `User.ReadWrite.All`, `openid`, `profile`, and `offline_access`.

`User.ReadWrite.All` is broader than any task in this lab required. It was already consented in the tenant but was not among the scopes these tasks requested. Although I requested narrow scopes per task, the shared Graph client retained earlier consent. For stricter isolation in production automation, a dedicated application registration with carefully scoped permissions would give more control than the shared Microsoft Graph Command Line Tools client.

![Consented permissions](screenshots/81-consented-permissions.png)
*Screenshot 81: Delegated permissions consented for Microsoft Graph Command Line Tools.*

---

# Key Lessons
1. **Dry-run before write operations.** The joiner workflow showed exactly what it would create before changing anything.
2. **Requested scopes and accumulated consent are not the same.** The permission review made that visible.
3. **Reporting should not automatically trigger remediation.** The stale report produced review candidates, and emergency-access accounts may legitimately look inactive.
4. **Verify automation output independently.** A script printing `True` is not enough on its own; the audit log confirmed the revocation was processed. Extending this to the create and disable events would be a natural improvement.
5. **Design note (not tested here):** Graph permissions and Entra roles are separate controls. A delegated call is also limited by the signed-in administrator's Entra role.

# Limitations
- Delegated authentication only. No app-only authentication, certificates, managed identities, or scheduled unattended automation.
- The privileged-role report covers active assignments only, not PIM eligibility.
- The session-revocation test did not prove termination of a live session.
- The `User.Create` run is not an isolated test of that permission (see Phase 4).
- Sign-in activity reflects what Entra had recorded when the report ran.

# Outcome
Connect to Graph → inventory identities → report stale accounts → review privileged access → dry-run onboarding → create users → disable a leaver and request session revocation → verify the revocation in the audit log → review delegated permissions

# Skills Demonstrated
Microsoft Graph PowerShell · Microsoft Entra ID · IAM Automation · Joiner and Leaver Workflows · Identity Lifecycle Management · PowerShell · Delegated Permissions · Least-Privilege Scope Selection · Access Reporting · Guest Identity Reporting · Privileged Access Reporting · Audit Logs · Session Revocation

## Files
- `joiners.csv`
- PowerShell scripts used during the lab
- `screenshots/` (72-81)
