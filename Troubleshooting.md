# Connect-MgGraph was found in the module ... but the module could not be loaded

This is not a missing-module problem, even though the message implies it.
`ExchangeOnlineManagement` bundles its own internal copy of the Graph
authentication assembly. When you `Import-Module Microsoft.Graph.Authentication`
afterward in the same session, .NET refuses to load a second, different
version of the same assembly - so the module fails to load even though it's
correctly installed.
## Fix: never import Graph modules into the same session as`ExchangeOnlineManagement`. 
This script runs the Graph connect-and-send logic
inside a `Start-Job` block, which executes in an isolated child process with
its own runspace - avoiding the conflict entirely.

# The specified module 'Microsoft.Graph.Authentication' was not loaded because no valid module file was found"

This is a genuinely different error and means the module isn't visible in the
context the script is running in. Common causes:
The module was installed with `-Scope CurrentUser` under a OneDrive-redirected
Documents folder. If OneDrive isn't synced/mounted at the moment the
scheduled task fires (e.g. right after a reboot), the module path may not be
available.
## Fix: also install with `-Scope AllUsers` so a copy exists
under `C:\Program Files\WindowsPowerShell\Modules`, independent of any user
profile or OneDrive state.
Leftover session state from testing commands manually before running the
full script - close the PowerShell window and start a fresh session.

# Task Scheduler shows "Last Run: never" or nothing happens
Confirm Program/script and Add arguments exactly match Step 7 above.
Set Start in to `C:\Scripts` - a missing working directory can cause
relative-path issues.
Check the task's History tab (enable "Enable All Tasks History" if it's
greyed out) for Event ID 101 (failed to start) vs 102 (completed, check the
result code).
