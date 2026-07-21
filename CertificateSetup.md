# Certificate Setup
Certificate-based authentication is used instead of a client secret or stored
password, because it's more secure and doesn't need to be renewed as often (you
control the expiry when you create it, e.g. 2 years).

# 1. Create the self-signed certificate
Run `Scripts/CreateSelfSignedCertificate.ps1`, or manually:
```powershell
$Cert = New-SelfSignedCertificate `
    -Subject "CN=MailboxQuotaReportCert2" `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -KeyExportPolicy Exportable `
    -KeySpec Signature `
    -KeyLength 2048 `
    -Provider "Microsoft Enhanced RSA and AES Cryptographic Provider" `
    -HashAlgorithm SHA256 `
    -NotAfter (Get-Date).AddYears(2)

$Cert.Thumbprint
```
This creates the certificate directly in your Personal certificate store
(`Cert:\CurrentUser\My`) and prints its Thumbprint - copy this, you'll need
it both for the app registration and for `$CertThumbprint` in the script.

# 2. Export the certificate
You only need to export the public key (`.cer`) - this is what gets
uploaded to Entra ID. The private key stays on this machine and is what the
script uses locally to sign requests.
```powershell
Export-Certificate -Cert $Cert -FilePath "C:\Scripts\MailboxQuotaReportCert.cer"
```
Never export or commit the `.pfx` (which includes the private key) unless you
specifically need to move the certificate to another machine, and if you do,
protect it with a strong password and keep it out of source control entirely.

# 3. Import the certificate (only needed on a different machine)
If the script will run on a machine other than the one you generated the
certificate on, you need the private key there too:
```powershell
Import-PfxCertificate `
    -FilePath "C:\Scripts\MailboxQuotaReportCert.pfx" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -Password (Read-Host -AsSecureString "Enter PFX password")
```
If running the scheduled task under a specific service account rather than
your own user profile, the certificate needs to be in a store that account can
read - typically `Cert:\LocalMachine\My`, since the Local Machine store is
accessible system-wide (unlike `CurrentUser`, which is tied to one profile).

# 4. Confirm the certificate is where you expect
```powershell
Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*MailboxQuotaReportCert*" }
```
This should show the certificate along with its thumbprint - confirm it
matches what's in your script and what's uploaded to the app registration.
Personal Store vs. Trusted Root
You do not need to place this certificate in Trusted Root Certification
Authorities. It only needs to exist in the Personal store
(`Cert:\CurrentUser\My` or `Cert:\LocalMachine\My`) on the machine running the
script - Entra ID validates the uploaded public key against what the script
presents when authenticating, it doesn't require full chain trust for this
app-only certificate auth flow.

<img width="1188" height="652" alt="Screenshot 2026-07-08 164342" src="https://github.com/user-attachments/assets/39dce205-93b1-4f10-9623-58bdc3eb4b98" />
