
[Lab6-README.md](https://github.com/user-attachments/files/33604567/Lab6-README.md)
# Lab 6: Device-Code Phishing Defense and Identity Incident Response


**Platform:** Microsoft Entra ID · **Tools:** Conditional Access, Microsoft Graph PowerShell, Entra sign-in and audit logs

## Summary
Device-code authentication is a legitimate Microsoft sign-in flow that attackers abuse by tricking users into entering an attacker-generated code. This lab builds and validates a Conditional Access control that blocks the flow, then walks the incident-response side: reviewing the sign-in evidence and containing the account by revoking its sessions.

**What this lab did not do:** I did not simulate a phishing lure or use attacker tooling. I generated a legitimate Microsoft device-code authentication in my lab tenant to reproduce the authentication behavior and test the defensive controls.

**Workflow:** Device-code authentication → sign-in-log visibility → Report-only validation → Conditional Access enforcement → investigation → session revocation → audit verification

---

## 1. Baseline: Device-Code Sign-In
I signed in as a disposable test account (Rachel Green) with Microsoft Graph PowerShell:
`Connect-MgGraph -Scopes "User.Read" -UseDeviceAuthentication`

The Graph context confirmed Delegated authentication with TokenCredentialType: DeviceCode.

![Get-MgContext showing AuthType Delegated and TokenCredentialType DeviceCode](screenshots/62-get-mgcontext.png)
*Screenshot 62: `Get-MgContext` showing AuthType: Delegated and TokenCredentialType: DeviceCode.*

![Entra sign-in log Basic info showing Success](screenshots/63a-signin-success.png)
*Screenshot 63A: Entra Basic info showing the Microsoft Graph Command Line Tools sign-in as Success.*

![Original transfer method: Device code flow](screenshots/63b-original-transfer-method.png)
*Screenshot 63B: Lower Basic info showing Original transfer method: Device code flow.*

## 2. Preventive Control: Conditional Access in Report-only
I created **CA-Block-DeviceCodeFlow** for the test account, targeting All resources, with Authentication flows → Device code flow and Grant → Block access. I started in Report-only mode to validate the effect without disrupting authentication.

![Conditional Access policy in Report-only](screenshots/64-ca-policy-report-only.png)
*Screenshot 64: CA-Block-DeviceCodeFlow configured for the test account, targeting All resources, with Authentication flows → Device code flow, Grant → Block access, and state set to Report-only.*

## 3. Report-only Validation
Because Graph PowerShell caches authentication, I cleared the session first (`Disconnect-MgGraph`, confirmed `Get-MgContext` was empty) and reconnected with `-ContextScope Process` to force a genuinely new device-code flow. The sign-in still succeeded, while the Report only view showed the policy would have blocked it.

![Report only view showing the policy result](screenshots/65-report-only-result.png)
*Screenshot 65: Report only view showing CA-Block-DeviceCodeFlow's result while the sign-in succeeded.*

## 4. Enforcement
I changed the policy from Report-only to On and repeated the fresh-session device-code authentication. The user completed credential verification, but Conditional Access denied the access request. Entra recorded the overall sign-in attempt as Failure.

This demonstrated that valid credentials alone do not guarantee access. Conditional Access can deny a sign-in after authentication requirements are evaluated.

![Microsoft sign-in page showing access denied](screenshots/66-enforced-block.png)
*Screenshot 66: Microsoft sign-in page showing “You don’t have access to this” after repeating the device-code authentication with CA-Block-DeviceCodeFlow enabled.*

![Failed sign-in after enforcement](screenshots/67a-signin-failure.png)
*Screenshot 67A: Refreshed Entra sign-in log showing the new Microsoft Graph Command Line Tools attempt with Status: Failure and Conditional Access: Failure.*

![Conditional Access enforcement evidence](screenshots/67b-ca-failure-detail.png)
*Screenshot 67B: Conditional Access details for the failed sign-in showing CA-Block-DeviceCodeFlow evaluated as Failure, confirming the enforced policy blocked the device-code flow.*

## 5. Investigation
Using the existing sign-in evidence, I reconstructed the test timeline:
- **Baseline:** device-code authentication succeeded before the control existed.
- **Report-only:** authentication succeeded, while Entra evaluated that the policy would block it.
- **Enforced:** the same flow was denied after the policy was enabled.

## 6. Containment and Audit Verification
Treating the initial successful device-code authentication as a simulated compromise, I revoked the test user's sessions and verified the action in the audit log.

![Revoke sessions confirmation](screenshots/69-revoke-sessions.png)
*Screenshot 69: Revoke sessions confirmation for the test user.*

![Audit log activity](screenshots/70a-audit-log.png)
*Screenshot 70A: Audit log showing the Update StsRefreshTokenValidFrom Timestamp activity.*

![Audit event modified properties](screenshots/70b-audit-details.png)
*Screenshot 70B: Audit event details showing the StsRefreshTokensValidFrom timestamp change in Modified Properties.*

The audit log confirmed Entra processed the revocation and updated the user's refresh-token validity timestamp, the mechanism Microsoft uses to invalidate previously issued refresh tokens. I did not perform a post-revocation token-redemption test, so this lab documents the remediation action rather than claiming an independently verified token failure.

## 7. Final State
![Policy list showing CA-Block-DeviceCodeFlow On](screenshots/71-policy-on.png)
*Screenshot 71: CA-Block-DeviceCodeFlow remaining set to On.*

---

## Residual Risk
Session revocation does not instantly destroy every access token already issued. An existing access token can remain usable until it expires unless the protected resource supports mechanisms such as Continuous Access Evaluation.

In a real incident, additional steps could include password reset, authentication-method review (removing unfamiliar MFA or passkey methods), OAuth/application-consent investigation, group and role-change review, mailbox rule and forwarding investigation, and temporary account disablement when warranted.

**Production scope:** For lab safety, the policy was scoped to a single test user. A production rollout should first inventory legitimate device-code usage, validate in Report-only mode, document approved exceptions, exclude emergency-access accounts where appropriate, and then expand enforcement.

## Skills Demonstrated
Microsoft Entra ID · Conditional Access · Microsoft Graph PowerShell · Device Code Flow · Sign-in Logs · Audit Logs · Session Revocation · Identity Incident Response · Zero Trust Principles
