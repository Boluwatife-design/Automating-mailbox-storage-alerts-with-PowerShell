Create a TXT document >> save in c:scripts as All files(file type)

    # ===========================================
    # Microsoft 365 Mailbox Quota Report
    # ===========================================

    # ---------------------------
    # VARIABLES
    # ---------------------------
    $Tenant = "domain name"
    $TenantId = "----------------------"
    $ClientID = "----------------------------"
    $CertThumbprint = "------------------------"

    $From = "username@domain.com"
    $To = "username@domain.com"

    $ThresholdGB = 45
    $QuotaGB = 50

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
    # Diagnostics (safe to keep, now that Write-Log exists)
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
    # Only import Exchange here — Graph gets imported inside the job below
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
    
                if ($SizeGB -ge $ThresholdGB) {
    
                    [PSCustomObject]@{
                        DisplayName = $Mailbox.DisplayName
                        Email       = $Mailbox.UserPrincipalName
                        UsedGB      = $SizeGB
                        QuotaGB     = $QuotaGB
                        PercentUsed = "{0}%" -f [math]::Round(($SizeGB / $QuotaGB) * 100, 1)
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

<img width="1331" height="466" alt="Screenshot 2026-07-08 165620" src="https://github.com/user-attachments/assets/0d5ba6a8-a821-41ee-9cac-fa5ebe6aa688" />
