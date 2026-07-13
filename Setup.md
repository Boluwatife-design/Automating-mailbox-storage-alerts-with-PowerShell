# Setup - the exact order this was built in
## Step 1: Create the working folder
```powershell
New-Item -Path "C:\Scripts" -ItemType Directory -Force
```
All scripts, logs, and diagnostic files live here.

## Step 2: Create the certificate
Run 

      `Scripts/CreateSelfSignedCertificate.ps1`. 
      
This generates a self-signed
certificate in your personal certificate store and exports the public `.cer` file.

Full walkthrough: `CertificateSetup.md`
Note the Thumbprint printed by the script - you'll need it in Step 4.

## Step 3: Install the required PowerShell modules
```powershell
Install-Module ExchangeOnlineManagement -Scope AllUsers -Force
Install-Module Microsoft.Graph.Authentication -Scope AllUsers -Force -AllowClobber
Install-Module Microsoft.Graph.Users.Actions -Scope AllUsers -Force -AllowClobber
```
Using `-Scope AllUsers` installs to `C:\Program Files\WindowsPowerShell\Modules`,
which is available regardless of which user account or context (e.g. Task
Scheduler, a service account) runs the script - it doesn't depend on a
user-profile path like OneDrive-redirected Documents.


## Step 4: Register the app in Entra ID
- Create an App Registration
- Upload the `.cer` file from Step 2
- Add API permissions:
  * Exchange.ManageAsApp (Office 365 Exchange Online, Application permission)
  * Mail.Send (Microsoft Graph, Application permission)
- Grant admin consent
- Assign the app the Exchange Administrator role (via Entra ID roles, since
  `Exchange.ManageAsApp` alone isn't enough for full mailbox access)
- Full walkthrough with screenshots: `AzureAppRegisteration.md`


## Step 5: Update the script variables
Open `Scripts/MailboxQuotaReport.ps1` and set:
```powershell
$Tenant         = "yourtenant.onmicrosoft.com"
$TenantId       = "your-tenant-id-guid"
$ClientID       = "your-app-registration-client-id"
$CertThumbprint = "your-certificate-thumbprint"

$From = "admin@yourtenant.com"
$To   = "admin@yourtenant.com"

$ThresholdGB = 45
$QuotaGB     = 50
```

## Step 6: Test it manually
```powershell
& "C:\Scripts\MailboxQuotaReport.ps1"
```
Check the log:
```powershell
Get-Content C:\Scripts\MailboxQuotaReport.log -Tail 20
```
You should see a clean sequence ending in `Storage report emailed successfully (job).` and `Finished.` - or `No mailboxes above threshold.` if nothing has hit
the limit yet.

## Step 7: Schedule it
Create a Windows Scheduled Task:
Setting	Value
Program/script	`powershell.exe`
Add arguments	`-NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\MailboxQuotaReport.ps1"`
Start in	`C:\Scripts`
Security option	Run whether user is logged on or not
Full walkthrough with screenshots: `TaskScheduler.md`

## Logging
All runs are logged to:
```
C:\Scripts\MailboxQuotaReport.log
```
Diagnostic snapshots are also written on each run to help debug module/path
issues:
```
C:\Scripts\PSVersion.txt
C:\Scripts\ModulePaths.txt
C:\Scripts\GraphModules.txt
```
