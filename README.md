# Claude No Reboot

Fix **Claude Desktop stuck on Windows after an update** without rebooting.

This targets a specific failure where Claude Desktop will not open and Windows reports errors such as:

- **"Another program is currently using this file."**
- **0x80070020**
- AppModel-Runtime **Event 208 / 215**
- Claude Desktop silently refuses to launch after an update

The problem can be a stale Desktop AppX Job Object left behind by the previous Claude version. The visible Claude process is already gone, so Task Manager, `handle.exe`, and ordinary parent/child process inspection may not reveal what is keeping the old container alive.

`ClaudeNoReboot.ps1` reads the actual Windows Job Object membership, shows the processes trapped in old Claude containers, and can stop only those stale members after you confirm.

## Quick start

Open PowerShell and run:

```powershell
$p = "$env:TEMP\ClaudeNoReboot.ps1"
irm "https://raw.githubusercontent.com/audioses/claude-no-reboot/main/ClaudeNoReboot.ps1" -OutFile $p
powershell -NoProfile -ExecutionPolicy Bypass -File $p
```

The script requests Administrator access if needed.

## Scan only

```powershell
.\ClaudeNoReboot.ps1 -ScanOnly
```

## Non-interactive fix

```powershell
.\ClaudeNoReboot.ps1 -Yes
```

Add `-NoRelaunch` if you do not want it to reopen Claude after cleanup.

## What it does

The script:

1. Finds the currently installed Claude Desktop MSIX/AppX version.
2. Enumerates `Container_Claude_*` Windows Job Objects.
3. Ignores the current `PackagedService` job.
4. Selects only old-version Claude user jobs belonging to your current Windows SID.
5. Shows every PID, executable, parent PID, start time, path, and command line inside those stale jobs.
6. Asks before terminating anything.
7. Stops only the stale-job members.
8. Verifies the stale container disappeared.
9. Relaunches Claude through its registered AppX identity.

## Safety

This is intentionally conservative.

It does **not**:

- uninstall Claude
- delete Claude settings or user data
- reset the AppX package
- modify package registration
- kill the current `CoworkVMService` packaged-service job
- terminate anything before showing you the process list and asking for confirmation

An old Claude job can contain WSL, Node, Python, SSH, Git, local servers, or other processes Claude started. Some may be work you intentionally left running. Read the list before approving cleanup.

## Why normal tools can miss it

Windows Desktop AppX applications use Job Objects to group processes. A child process can survive after the visible Claude process exits while remaining a member of the old Claude job.

The script uses Windows native APIs:

- `NtQueryDirectoryObject`
- `NtOpenJobObject`
- `QueryInformationJobObject`
- `JobObjectBasicProcessIdList`

That exposes the real process membership of the stale Claude container instead of guessing based on process names.

## Requirements

- Windows 10 or Windows 11
- Claude Desktop installed through its current MSIX/AppX package
- Windows PowerShell 5.1 or newer
- Administrator rights for cleanup

## Tested failure mode

This utility was created after reproducing a Claude Desktop update failure where the old Claude AppX job remained alive because of an orphaned WSL process tree. Claude failed to launch with `0x80070020` and AppModel-Runtime Event 215 even though `handle.exe` reported no matching file handles. Killing only the processes belonging to the old Claude Job Object immediately released the stale container and Claude launched normally, with no reboot.

## Disclaimer

Claude No Reboot is an independent community utility and is not affiliated with Anthropic.

It targets one specific Windows/Claude Desktop failure mode. If Claude changes its package or Job Object naming, the script may need an update.

## License

MIT
