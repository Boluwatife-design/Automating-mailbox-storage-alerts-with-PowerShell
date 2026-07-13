# Automating-mailbox-storage-alerts-with-PowerShell
Step by step guide to automating mailbox storage alerts using PowerShell

Microsoft 365 Mailbox Quota Reporter

A PowerShell solution that monitors Exchange Online mailbox storage and automatically
emails an HTML report for any mailbox approaching its quota - fully unattended,
scheduled via Windows Task Scheduler.
Built and tested end-to-end on Windows PowerShell 5.1, using certificate-based
app-only authentication (no stored passwords, no legacy SMTP AUTH).

# Features
- Exchange Online authentication via Entra ID App Registration + certificate
- Microsoft Graph app-only authentication (no interactive sign-in)
- Mailbox size calculated directly from `TotalItemSize` (accurate, in GB)
- Configurable alert threshold vs. actual quota
- Styled HTML email report, sent only when a mailbox crosses the threshold
- Logging to a local log file for every run
- Runs unattended via Windows Task Scheduler
- Known gotcha documented and fixed: `ExchangeOnlineManagement` + `Microsoft.Graph`

module conflict (see Troubleshooting)
Sample Report
```
------------------------------------------------------------
Microsoft 365 Mailbox Storage Report
------------------------------------------------------------
DisplayName      Email                       UsedGB   QuotaGB   PercentUsed
------------------------------------------------------------
John Doe         john.doe@company.com        47       50        94%
Jane Smith       jane.smith@company.com      49       50        98%
```
Architecture
```
Task Scheduler
      │
      ▼
MailboxQuotaReport.ps1  (main session)
      │
      ├─► Connect-ExchangeOnline  (cert auth)
      ├─► Get-EXOMailbox / Get-EXOMailboxStatistics
      │
      └─► Start-Job  (separate process)
                │
                ├─► Connect-MgGraph  (cert auth)
                └─► Send-MgUserMail  (HTML report)
```
Graph is intentionally run inside a background job rather than in the main
session. See Troubleshooting for why this matters.

# Technologies
- PowerShell (Windows PowerShell 5.1)
- ExchangeOnlineManagement module
- Microsoft.Graph.Authentication + Microsoft.Graph.Users.Actions modules
- Microsoft Entra ID (App Registration, certificate auth)
- Microsoft Graph API (Send-MgUserMail)
- Windows Task Scheduler

# Requirements
- Microsoft 365 tenant with Exchange Online
- An account with Exchange Administrator and Application Administrator (or Global
  Administrator) rights to complete setup
- A Windows machine to host the scheduled task (server or workstation, always on
  at the time the task is scheduled to run)

