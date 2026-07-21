Create a TXT document >> save in c:scripts as All files(file type)

       <#
    .SYNOPSIS
    Audits Exchange Online mailbox sizes and emails an HTML report when mailboxes
    approach their storage quota.
    .DESCRIPTION
    Connects to Exchange Online using an Entra ID App Registration with certificate
    authentication, calculates mailbox size, and sends a styled HTML report via the
    Microsoft Graph Mail API.

    Graph is deliberately loaded inside a background job (Start-Job), NOT in the
    main session. This avoids a real assembly-loading conflict between the
    ExchangeOnlineManagement module and the Microsoft.Graph modules when both are
    imported into the same PowerShell session. See README > Troubleshooting.
    .NOTES
    Tested and confirmed working on Windows PowerShell 5.1, run both interactively
    and via Windows Task Scheduler.
    #>

    # ---------------------------
    # VARIABLES - update these for your tenant
    # ---------------------------
    $Tenant         = "yourtenant.onmicrosoft.com"
    $TenantId       = "YOUR-TENANT-ID-GUID"
    $ClientID       = "YOUR-APP-REGISTRATION-CLIENT-ID"
    $CertThumbprint = "YOUR-CERTIFICATE-THUMBPRINT"
    
    $From = "admin@yourtenant.com"
    $To   = "admin@yourtenant.com"
    
    $ThresholdPercent = 90   # alert when a mailbox crosses this % of ITS OWN quota
    
    $LogFile = "C:\Scripts\MailboxQuotaReport.log"
    
    # ---------------------------
    # LOG FUNCTION (must be defined BEFORE it's called)
    # ---------------------------
    function Write-Log {
        param([string]$Message)
        $Time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        "$Time - $Message" | Out-File $LogFile -Append
    }
    
    # ---------------------------
    # Diagnostics - useful the first time you set this up, safe to keep
    # ---------------------------
    Write-Log "PowerShell version: $($PSVersionTable.PSVersion)"
    Write-Log "User: $env:USERNAME"
    Write-Log "Module Path:"
    Write-Log $env:PSModulePath
    
    $PSVersionTable | Out-File C:\Scripts\PSVersion.txt
    $env:PSModulePath -split ';' | Out-File C:\Scripts\ModulePaths.txt
    
    Get-Module Microsoft.Graph.Authentication -ListAvailable |
        Select Version, ModuleBase |
        Out-File C:\Scripts\GraphModules.txt
    
    # ---------------------------
    # Only import Exchange here - Graph is imported inside the job below
    # ---------------------------
    Import-Module ExchangeOnlineManagement -ErrorAction Stop
    
    try {

    Write-Log "Starting mailbox quota report."

    # ---------------------------
    # Connect Exchange Online
    # ---------------------------
    Connect-ExchangeOnline `
        -Organization $Tenant `
        -AppId $ClientID `
        -CertificateThumbprint $CertThumbprint `
        -ShowBanner:$false

    Write-Log "Connected to Exchange Online."

    # ---------------------------
    # Get Mailboxes
    # ---------------------------
    $Mailboxes = Get-EXOMailbox -ResultSize Unlimited

    $FullMailboxes = foreach ($Mailbox in $Mailboxes) {

        $Stats = Get-EXOMailboxStatistics -Identity $Mailbox.UserPrincipalName

        if ($Stats.TotalItemSize -and $Stats.TotalItemSize.Value) {

            $SizeGB = [math]::Round($Stats.TotalItemSize.Value.ToGB(), 2)

            # Pull this mailbox's ACTUAL quota rather than assuming everyone
            # has the same one. Licensing tiers (e.g. Exchange Online Plan 1
            # vs Plan 2) and custom overrides mean quotas can differ per user.
            #
            # NOTE: Objects returned over the Exchange Online remote session
            # are "deserialized" - .Value and .IsUnlimited don't reliably work
            # on them. Parsing the .ToString() text representation instead
            # (e.g. "50 GB (53,687,091,200 bytes)" or "Unlimited") is the
            # robust way to get the real quota regardless of object type.
            $MailboxDetail = Get-Mailbox -Identity $Mailbox.UserPrincipalName
            $QuotaString = $MailboxDetail.ProhibitSendReceiveQuota.ToString()

            if ($QuotaString -eq "Unlimited") {
                # Skip mailboxes with no defined quota - percentage math doesn't apply
                continue
            }

            if ($QuotaString -match '\(([\d,]+)\s*bytes\)') {
                $QuotaBytes = [int64]($Matches[1] -replace ',', '')
                $QuotaGB    = [math]::Round($QuotaBytes / 1GB, 2)
            }
            else {
                Write-Log "Could not parse quota for $($Mailbox.UserPrincipalName): '$QuotaString'"
                continue
            }

            $PercentUsedValue = [math]::Round(($SizeGB / $QuotaGB) * 100, 1)

            if ($PercentUsedValue -ge $ThresholdPercent) {

                [PSCustomObject]@{
                    DisplayName = $Mailbox.DisplayName
                    Email       = $Mailbox.UserPrincipalName
                    UsedGB      = $SizeGB
                    QuotaGB     = $QuotaGB
                    PercentUsed = "{0}%" -f $PercentUsedValue
                }
            }
        }
    }

    # ---------------------------
    # Send only if required
    # ---------------------------
    if ($FullMailboxes.Count -gt 0) {

        $Header = @"
    <style>
    body{
    font-family:Calibri;
    }
    
    table{
    border-collapse:collapse;
    width:800px;
    }
    
    th{
    background:#d83b01;
    color:white;
    padding:10px;
    }
    
    td{
    border:1px solid #ddd;
    padding:8px;
    }
    
    tr:nth-child(even){
    background:#f2f2f2;
    }
    
    h2{
    color:#333;
    }
    </style>
    
    <h2>Microsoft 365 Mailbox Storage Report</h2>
    
    <p>
    The following mailboxes are using more than <b>90%</b> of their storage quota.
    </p>
    
    <p>
    Generated:
    <b>$(Get-Date)</b>
    </p>
    
    "@

        $Table = $FullMailboxes |
            Sort-Object UsedGB -Descending |
            ConvertTo-Html -Fragment

        $Body = $Header + $Table

        Write-Log "Handing off to Graph job to send email."

        # ---------------------------
        # Run Graph connect + send in a SEPARATE job/process
        # This avoids the ExchangeOnlineManagement / Microsoft.Graph
        # assembly version conflict in the same session.
        # ---------------------------
        $graphJob = Start-Job -ScriptBlock {
            param($TenantId, $ClientID, $CertThumbprint, $From, $To, $Body, $LogFile)

            function Write-Log {
                param([string]$Message)
                $Time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                "$Time - $Message" | Out-File $LogFile -Append
            }

            try {
                Import-Module Microsoft.Graph.Authentication -ErrorAction Stop
                Import-Module Microsoft.Graph.Users.Actions -ErrorAction Stop

                Connect-MgGraph `
                    -TenantId $TenantId `
                    -ClientId $ClientID `
                    -CertificateThumbprint $CertThumbprint `
                    -NoWelcome

                Write-Log "Connected to Microsoft Graph (job)."

                $Params = @{
                    Message = @{
                        Subject      = "Microsoft 365 Mailbox Storage Report"
                        Body         = @{
                            ContentType = "HTML"
                            Content     = $Body
                        }
                        ToRecipients = @(
                            @{
                                EmailAddress = @{
                                    Address = $To
                                }
                            }
                        )
                    }
                    SaveToSentItems = $true
                }

                Send-MgUserMail -UserId $From -BodyParameter $Params

                Write-Log "Storage report emailed successfully (job)."
            }
            catch {
                Write-Log "Graph job error: $($_.Exception.Message)"
            }
            finally {
                if (Get-Command Disconnect-MgGraph -ErrorAction SilentlyContinue) {
                    Disconnect-MgGraph
                }
            }

        } -ArgumentList $TenantId, $ClientID, $CertThumbprint, $From, $To, $Body, $LogFile

        # Wait for the job and pull back any output/errors into this session's log
        Receive-Job -Job $graphJob -Wait -AutoRemoveJob

    }
        else {

            Write-Log "No mailboxes above threshold."

        }

    }
    catch {
    
        Write-Log $_.Exception.Message
    
    }
    finally {

    try {
        Disconnect-ExchangeOnline -Confirm:$false
    }
    catch {}

    Write-Log "Finished."
    }

<img width="1916" height="316" alt="Screenshot 2026-07-08 172824" src="https://github.com/user-attachments/assets/0027a68d-9b1a-4f75-8c0f-a38cfddbb336" />

