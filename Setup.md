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

Full walkthrough: `Docs/CertificateSetup.md`
Note the Thumbprint printed by the script - you'll need it in Step 4.
