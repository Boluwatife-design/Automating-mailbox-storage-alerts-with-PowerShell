# Entra ID App Registration Setup
This app registration is what allows the script to authenticate to Exchange
Online and Microsoft Graph without any interactive sign-in or stored password -
just a certificate.
## 1. Create the App Registration
- Go to entra.microsoft.com > Applications > App registrations > New registration.
- Name it something identifiable, e.g. `Mailbox Quota Reporter`.
- Supported account types: Single tenant (Accounts in this organizational
directory only).
- Leave Redirect URI blank - this app never signs in interactively.
- Click Register.
- From the app's Overview page, note down:
  * Application (client) ID → this is your `$ClientID`
  * Directory (tenant) ID → this is your `$TenantId`
    <img width="1818" height="916" alt="Screenshot 2026-07-08 103037" src="https://github.com/user-attachments/assets/b647d6b7-2347-4575-9e23-7c1e913e4521" />

# 2. Upload the certificate
- In your app registration, go to Certificates & secrets > Certificates
tab.
- Click Upload certificate.
- Upload the `.cer` file exported by `Scripts/CreateSelfSignedCertificate.ps1`.
- Once uploaded, confirm the Thumbprint shown matches the one printed by
- the script - this is your `$CertThumbprint`.   
    <img width="1909" height="1100" alt="Screenshot 2026-07-08 102242" src="https://github.com/user-attachments/assets/67243483-6472-4b5e-96ef-b33bdd2ac49a" />
    <img width="1188" height="652" alt="Screenshot 2026-07-08 164342" src="https://github.com/user-attachments/assets/b83449f5-969d-4668-a14c-b0a744a3d4fc" />

# 3. Add API permissions
- Go to API permissions > Add a permission.
- Microsoft Graph (Application permission)
- Search for Mail.Send
- Select Application permissions (not Delegated)
- Add permission
- Office 365 Exchange Online (Application permission)
- Under APIs my organization uses, search for Office 365 Exchange Online
- Select Application permissions
- Add Exchange.ManageAsApp

<img width="1705" height="512" alt="Screenshot 2026-07-08 102635" src="https://github.com/user-attachments/assets/593411cf-feff-4441-a007-688bb814c1a6" />
<img width="1220" height="844" alt="Screenshot 2026-07-08 102724" src="https://github.com/user-attachments/assets/4d5f69ba-ed31-4fea-bede-3a560641cc2b" />

# 4. Grant admin consent
- Back on the API permissions page, click Grant admin consent for
[Your Org] and confirm. Both permissions should show a green checkmark under
Status once granted.

<img width="1460" height="887" alt="Screenshot 2026-07-08 114934" src="https://github.com/user-attachments/assets/872a3c71-c5bb-4140-a9ef-48c777cae1cc" />


# 5. Assign the Exchange Administrator role
`Exchange.ManageAsApp` alone only grants the ability to call
Exchange-management APIs as an app - it still needs an actual role assigned to
work against mailboxes:
- Go to Entra ID > Roles and administrators.
- Find and select Exchange Administrator.
- Click Add assignments > search for your app registration by name
(this requires the app to have a service principal - it's created
automatically the first time you register it).
Assign.
Without this step, `Connect-ExchangeOnline` may succeed but calls like
`Get-EXOMailbox` can fail or return incomplete results.
Summary of values you now have
Variable	Where to find it
 * `$Tenant`	Your tenant's `.onmicrosoft.com` domain
* `$TenantId`	App registration > Overview > Directory (tenant) ID
* `$ClientID`	App registration > Overview > Application (client) ID
* `$CertThumbprint`	Certificates & secrets > Certificates tab
- Plug these into `Scripts/MailboxQuotaReport.ps1` per the main README.
