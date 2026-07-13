# Windows Task Scheduler Setup
This is how the script runs unattended, on a schedule, without needing anyone
logged in.

## General tab
- Name: `Microsoft 365 Mailbox Quota Report`
- Description: `Checks Exchange Online mailbox usage and emails a report.`
- Security options: select Run whether user is logged on or not
  * This is important. "Run only when user is logged on" means the task
silently does nothing if the machine is locked or the user is logged off.
* Check Run with highest privileges.

<img width="878" height="764" alt="Screenshot 2026-07-08 133905" src="https://github.com/user-attachments/assets/bb49e544-31eb-42aa-8002-6f5e2fa737e1" />
<img width="1686" height="1057" alt="Screenshot 2026-07-08 134935" src="https://github.com/user-attachments/assets/4ae03f65-b492-4fa4-914a-84365105d02d" />

# Triggers tab
- New trigger, set to Daily (or your preferred frequency).
- Pick a time when the machine is reliably powered on and network-connected.

# Actions tab
- Action: Start a program
- Program/script:
```
  powershell.exe
  ```
- Add arguments:
```
  -NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\MailboxQuotaReport.ps1"
  ```
- Start in (optional):
```
  C:\Scripts
  ```
Setting this matters - without a defined working directory, relative paths
and some file operations can behave unexpectedly when Task Scheduler launches
the process.

# Conditions tab
- Uncheck Start the task only if the computer is on AC power if this runs
  on a laptop that's sometimes on battery.
- Leave Wake the computer to run this task checked if the machine sleeps
and you want the report to run regardless.

# Settings tab
- Check Allow task to be run on demand (useful for manual testing).
- Check If the task fails, restart every - e.g. every 10 minutes, up to 3
  attempts, in case of a transient network issue.

# Saving
- When you save after selecting "Run whether user is logged on or not," you'll be
   prompted for the Windows account password for the account the task runs as.
- Enter it - this is required for the task to authenticate outside an active

# logon session.
* Testing
- Right-click the task > Run.
- Go to the History tab (click "Enable All Tasks History" in the Actions
   pane on the right if it's greyed out).
Look for:
- Event ID 100 - task started
- Event ID 102 - task completed, check the result code (`0x0` = success)
Cross-check with the actual log file:
```powershell
   Get-Content C:\Scripts\MailboxQuotaReport.log -Tail 15
   ```
### A successful run's log should end with either `Storage report emailed successfully (job).` and `Finished.`, or `No mailboxes above threshold.` and
`Finished.` if nothing crossed the limit that day.
