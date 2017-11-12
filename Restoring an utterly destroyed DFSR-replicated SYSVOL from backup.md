# Restoring an utterly destroyed DFSR-replicated SYSVOL from backup

Warning: this is not official Microsoft documentation and some of these steps might not actually be supported.

This guide is provided “as is”, without warranty of any kind, express or implied. In no event shall the authors or
copyright holders be liable for any claim, damages or other liability arising from, out of or in connection with
applying the steps outlined in this guide.

## When to use

Use this if none of your domain controllers has any meaningful content (i.e. the `Policies` and `scripts`
subdirectories) in its `%WinDir%\SYSVOL\domain` directory, but you do have a fairly recent backup.

Use this if your SYSVOL is replicated using DFSR (*Distributed File System Replication*) instead of the older FRS
(*File Replication Service*). If your domain is still using FRS, simply follow Microsoft's
[KB315457](https://support.microsoft.com/en-us/help/315457/how-to-rebuild-the-sysvol-tree-and-its-content-in-a-domain).

This guide does not describe how to restore potentially broken junction points in the subdirectories of
`%WinDir%\SYSVOL`. The aforementioned KB315457 has all the necessary info; the junction points appear not to have
changed during the move from FRS to DFSR.

Do not use this guide if your domain controllers replicate other folders via DFSR apart from SYSVOL, as the DFSR
configuration is deleted from every domain controller in phase 1.

## Overview

The SYSVOL restore process described in this document is divided into the following phases:

1. DFSR cleanup
2. SYSVOL restore
3. DC replication
4. DFSR reconfiguration

## Phase 1: DFSR cleanup

In this phase, we will stop and disable DFSR and delete the DFSR database. Deleting the DFSR database means we start
with a clean slate. Note that this is dangerous if your domain controller replicates other DFS folders as well.

Perform the following steps on *each* domain controller:

1. Stop the DFSR service by one of the following means:
    * In the *Services* management console, select the *DFS Replication* service and click on the stop button in the
      toolbar.
    * In an administrative command prompt, type `net stop DFSR` and press Enter.
    * In an administrative PowerShell, type `Stop-Service DFSR` and press Enter.
2. Disable the DFSR service by one of the following means:
    * In the *Services* management console, right-click the *DFS Replication* service, select *Properties* from the
      context menu, switch to the *General* tab and, next to *Startup type:*, choose *Disabled* from the combo box.
    * In an administrative command prompt, type `sc config DFSR start= disabled` and press Enter. (The space after the
      equals sign is deliberate!)
    * In an administrative PowerShell, type `Set-Service DFSR -StartupType Disabled` and press Enter.
3. Delete the DFSR database.
    1. Download `PsExec64.exe` from https://live.sysinternals.com/ ([direct
       link](https://live.sysinternals.com/PsExec64.exe)).
    2. Open an administrative command prompt or PowerShell.
    3. Using the `cd <Path>` command, switch to the directory into which you have downloaded `PsExec64.exe`. If you are
       using the command prompt, you will have to use `cd /d <Path>` if the downloaded file is on a different drive than
       the startup directory.
    4. Launch a PowerShell as the *SYSTEM* user by typing `.\PsExec64.exe -s powershell.exe`. Accept the EULA if
       necessary.
    5. In the *SYSTEM* PowerShell, switch to the system drive's *System Volume Information* directory:
       `cd "$env:SystemDrive\System Volume Information"`
    6. Delete the *DFSR* subdirectory: `Remove-Item DFSR -Recurse -Force`
    7. Exit the *SYSTEM* PowerShell: `exit`

## Phase 2: SYSVOL restore

Now, we will restore the SYSVOL contents from a backup. Since the DFSR configuration for SYSVOL is stored in Active
Directory and domain controllers cannot replicate if SYSVOL is broken, we will perform this “manual” restore on all
domain controllers.

I recommend to restore SYSVOL to all domain controllers from a backup of a single domain controller to ensure that the
data is consistent.

1. Choose a domain controller backup. In a healthily replicating domain, the recommendation is to choose the most
   recently backed up domain controller or the domain controller with the the *PDC Emulator* FSMO role.
2. Restore the contents of `%WinDir%\SYSVOL\domain` from this backup to each domain controller. Use the same backup for
   all domain controllers.
3. Reboot the domain controller to ensure all necessary services are restarted and functional.

## Phase 3: DC replication

DFSR requires that Active Directory replication between domain controllers is functional. Test this on each DC by
forcing replication using `repadmin /SyncAll /A` in an administrative command prompt or PowerShell and checking the
status using `repadmin /showrepl` and the Event Log.

## Phase 4: DFSR reconfiguration

Finally, we restore DFSR SYSVOL replication. Most of these steps are documented in
[KB2218556](https://support.microsoft.com/en-us/help/2218556/how-to-force-an-authoritative-and-non-authoritative-synchronization-fo).

1.  Choose a domain controller which will be authoritative for the restore. Since we have restored the same data to all
    domain controllers, the choice should not make a difference, but when in doubt, choose the one with the *PDC
    Emulator* FSMO role. The text in this phase will refer to the chosen domain controller as the *authoritative domain
    controller* and to the other domain controllers in the domain as *non-authoritative domain controllers*.
2.  Open *ADSI Edit* by launching *adsiedit.msc*. You may run ADSI Edit on any domain controller, or a Windows client
    computer with RSAT installed.
3.  Perform the following steps in ADSI Edit:
    1. Right-click *ADSI Edit* in the tree structure on the left, select *Connect to...*, select the well-known *Default
       naming context*, and ensure you are connecting to the authoritative domain controller (by typing its full name
       into the box labeled *Select or type a domain or server*, if need be).
    2. Open the default naming context in the tree structure, then open the domain object, the *Domain Controllers* OU,
       the object for the authoritative domain controller, the *DFSR-LocalSettings* object, the *Domain System Volume*
       object, and the *SYSVOL Subscription* object. The attribute editor should appear.
    3. Double-click the `msDFSR-Enabled` attribute and change its value to `False`. Confirm with *OK*.
    4. Double-click the `msDFSR-Options` attribute and change its value to `1`. This chooses that domain controller as
       the authoritative source for initial replication. Confirm with *OK*.
    5. Close the attribute editor with *OK*.
    6. Return to the *Domain Controllers* OU. For every other (i.e. non-authoritative) domain controller, do the
       following:
        1. Open the object for the non-authoritative domain controller, the *DFSR-LocalSettings* object, the *Domain
           System Volume* object, and the *SYSVOL Subscription* object. The attribute editor should appear.
        2. Double-click the `msDFSR-Enabled` attribute and change its value to `False`. Confirm with *OK*. (Do not change
           the `msDFSR-Options` attribute on the non-authoritative domain controllers!)
        3. Close the attribute editor with *OK*.
4.  Force domain controller replication from the authoritative domain controller to the non-authoritative domain
    controllers using `repadmin /SyncAll /A` in an administrative command prompt or PowerShell on the authoritative
    domain controller. Check that replication works.
    * You may want to add additional connections in ADSI Edit to the other domain controllers and check whether the new
      values of the attributes modified above have reached these domain controllers. For the sake of consistency, only
      modify the settings via the ADSI Edit connection to the authoritative domain controller.
5.  On the authoritative domain controller, take the DFSR service online again:
    1. Set the DFSR service to start automatically using one of the following methods:
        * In the *Services* management console, right-click the *DFS Replication* service, select *Properties* from the
          context menu, switch to the *General* tab and, next to *Startup type:*, choose *Automatic* from the combo box.
        * In an administrative command prompt, type `sc config DFSR start= auto` and press Enter. (As above, the space
          after the equals sign is deliberate!)
        * In an administrative PowerShell, type `Set-Service DFSR -StartupType Automatic` and press Enter.
    2. Start the DFSR service using one of the following methods:
        * In the *Services* management console, select the *DFS Replication* service and click on the play button in the
          toolbar.
        * In an administrative command prompt, type `net start DFSR` and press Enter.
        * In an administrative PowerShell, type `Start-Service DFSR` and press Enter.
6.  Check the *DFS Replication* event log (under *Applications and Services Logs*) on the authoritative domain
    controller. Event 4114, noting that the replicated folder at `%WinDir%\SYSVOL\domain` has been disabled, should have
    appeared. Do not worry about the warning event 6702; we have forced the creation of a new system configuration by
    deleting the existing one in phase 1.
7.  Perform the following steps in ADSI Edit to re-enable SYSVOL replication on the authoritative domain controller:
    1. Open the properties of the *SYSVOL Subscription* object of the authoritative domain controller, as described in
       step 3.ii.
    2. Change `msDFSR-Enabled` to `True`.
8.  Repeat step 4 to force and verify replication.
9.  In an administrative command prompt or PowerShell on the authoritative domain controller, force DFSR to update its
    configuration from Active Directory by executing `dfsrdiag PollAD`.
10. Check that event 4602, signifying successful initialization of SYSVOL replication, has been logged in the *DFS
    Replication* event log on the authoritative domain controller.
11. On every non-authoritative domain controller, take the DFSR service online again. The process is the same as for the
    authoritative domain controller, as outlined in step 5.
12. Every non-authoritative domain controller should have logged event 4114, noting that the replicated folder at
    `%WinDir%\SYSVOL\domain` has been disabled. (The warning event 6702 appears for the same reason as on the
    authoritative domain controller and is equally harmless.)
13. For every non-authoritative domain controller, perform the following steps in ADSI Edit (preferably through the
    connection to the authoritative domain controller!):
    1. Open the properties of the *SYSVOL Subscription* object of the non-authoritative domain controller, as described
       in step 3.vi.a.
    2. Change `msDFSR-Enabled` to `True`.
14. Repeat step 4 to force and verify replication.
15. On every non-authoritative domain controller, force DFSR to update its configuration from Active Directory by
    executing `dfsrdiag PollAD` from an administrative command prompt or PowerShell.
16. Check that event 4614, signifying that DFSR is ready to receive the new contents of SYSVOL from the authoritative
    domain controller, has been logged in the *DFS Replication* event log on each non-authoritative domain controller.
17. Eventually, every non-authoritative domain controller should log event 4604 in the *DFS Replication* event log,
    confirming that initial replication of SYSVOL is complete.
18. Test DFS replication:
    1. Create an empty text file under `%WinDir%\SYSVOL\domain\scripts` on one domain controller.
    2. Check that the empty text file has been replicated to the other domain controllers.
    3. Modify the contents of the text file on that domain controller.
    4. Verify that the modification has been successfully replicated to the other domain controllers.
    5. Repeat steps 18.iii and 18.iv for all the other domain controllers.
    6. Delete the text file on one of the domain controllers.
    7. Verify that the file has been deleted on the other domain controllers as well.

Congratulations; your SYSVOL should be replicating again!
