# Transferring Active Directory DNS FSMO role

Warning: this is not official Microsoft documentation and some of these steps might not actually be supported.

This guide is provided "as is", without warranty of any kind, express or implied. In no event shall the authors or
copyright holders be liable for any claim, damages or other liability arising from, out of or in connection with
applying the steps outlined in this guide.

## Conventions in this document

<table>
  <tr>
    <th>Name of the domain:</th>
    <td><code>ad.example.com</code></td>
  </tr>
  <tr>
    <th>in LDAP syntax:</th>
    <td><code>DC=ad,DC=example,DC=com</code></td>
  </tr>
  <tr></tr>
  <tr>
    <th>DC being demoted:</th>
    <td><code>olddc.ad.example.com</code></td>
  </tr>
  <tr>
    <th>new DC replacing it:</th>
    <td><code>newdc.ad.example.com</code></td>
  </tr>
  <tr>
    <th>forcefully deleted DC:</th>
    <td><code>dcgrave.ad.example.com</code></td>
  </tr>
  <tr>
    <th>AD site name:</th>
    <td><code>MySite</code></td>
  </tr>
</table>

The GUIDs are placeholders as well.

In this document, the phrase *object name* refers to the *Distinguished Name* of an object in the LDAP directory.
Example of such an object name: `CN=Users,DC=ad,DC=example,DC=com`

## Symptoms

The domain controller `newdc.ad.example.com` is meant to take over from the domain controller `olddc.ad.example.com`. To
make this possible, the ancient domain controller `dcgrave.ad.example.com`, which has not been started for ages (long
past the tombstone age), had to be removed (using the `metadata cleanup` functionality of `ntdsutil`). However, the
demotion of `olddc.ad.example.com` fails with the following error message:

> The operation failed because:
>
> Active Directory Domain Services could not transfer the remaining data in directory partition
> DC=ForestDnsZones,DC=ad,DC=example,DC=com to Active Directory Domain Controller \\\\newdc.ad.example.com.
>
> "The directory service is missing mandatory configuration information, and is unable to determine the ownership of
> floating single-master operation roles."

Simultaneously, the following two messages are logged in the *Directory Service* event log:

<table>
  <tr>
    <th>Priority</th>
    <td>Warning</td>
  </tr>
  <tr>
    <th>Source</th>
    <td>ActiveDirectory_DomainService</td>
  </tr>
  <tr>
    <th>Event ID</th>
    <td>2091</td>
  </tr>
  <tr>
    <th>Task Category</th>
    <td>Replication</td>
  </tr>
  <tr>
    <th>Message</th>
    <td>
      Ownership of the following FSMO role is not set or could not be read.<br/>
      <br/>
      Operations which require contacting a FSMO operation master will fail until this condition is corrected.<br/>
      <br/>
      FSMO Role: CN=Infrastructure,DC=DomainDnsZones,DC=ad,DC=example,DC=com<br/>
      FSMO Server DN:
      CN=NTDS Settings\0ADEL:8d91e2ae-dada-4459-9ec7-c8e58b255cb7,CN=DCGRAVE\0ADEL:a6004b55-6988-4886-b027-31827f5f8f83,CN=Servers,CN=MySite,CN=Sites,CN=Configuration,DC=ad,DC=example,DC=com<br/>
      <br/>
      User Action:<br/>
      <br/>
      1. Determine which server should hold the role in question.<br/>
      2. Configuration view may be out of date. If the server in question has been promoted recently, verify that the
      Configuration partition has replicated from the new server recently.  If the server in question has been demoted
      recently and the role transferred, verify that this server has replicated the partition (containing the latest
      role ownership) lately.<br/>
      3. Determine whether the role is set properly on the FSMO role holder server. If the role is not set, utilize
      NTDSUTIL.EXE to transfer or seize the role. This may be done using the steps provided in KB articles 255504 and
      324801 on http://support.microsoft.com.<br/>
      4. Verify that replication of the FSMO partition between the FSMO role holder server and this server is occurring
      successfully.<br/>
      <br/>
      The following operations may be impacted:<br/>
      Schema: You will no longer be able to modify the schema for this forest.<br/>
      Domain Naming: You will no longer be able to add or remove domains from this forest.<br/>
      PDC: You will no longer be able to perform primary domain controller operations, such as Group Policy updates and
      password resets for non-Active Directory Domain Services accounts.<br/>
      RID: You will not be able to allocation new security identifiers for new user accounts, computer accounts or
      security groups.<br/>
      Infrastructure: Cross-domain name references, such as universal group memberships, will not be updated properly if
      their target object is moved or renamed.
    </td>
  </tr>
</table>

<table>
  <tr>
    <th>Priority</th>
    <td>Error</td>
  </tr>
  <tr>
    <th>Source</th>
    <td>ActiveDirectory_DomainService</td>
  </tr>
  <tr>
    <th>Event ID</th>
    <td>2022</td>
  </tr>
  <tr>
    <th>Task Category</th>
    <td>Replication</td>
  </tr>
  <tr>
    <th>Message</th>
    <td>
      The operations master roles held by this directory server could not transfer to the following remote directory
      server.<br/>
      <br/>
      Remote directory server:<br/>
      \\newdc.ad.example.com<br/>
      <br/>
      This is preventing removal of this directory server.<br/>
      <br/>
      User Action<br/>
      Investigate why the remote directory server might be unable to accept the operations master roles, or manually
      transfer all the roles that are held by this directory server to the remote directory server. Then, try to remove
      this directory server again.<br/>
      <br/>
      Additional Data<br/>
      Error value:<br/>
      5005 The directory service is missing mandatory configuration information, and is unable to determine the
      ownership of floating single-master operation roles.<br/>
      Extended error value:<br/>
      0<br/>
      Internal ID:<br/>
      52498735
    </td>
  </tr>
</table>

The substring `\0ADEL` in the name hints that the Active Directory object in question has been deleted.

## Reason

(Warning: this is based more on educated guesses than actual knowledge.)

To one’s surprise, the DNS zones stored in Active Directory also fall under the FSMO principle. Unfortunately,
transferring them is not as easy.

## Failed solution attempts

* `ntdsutil` does not have the relevant functionality.
* When editing the attribute using *ADSI Edit*, it tries to contact the registered (deleted) domain controller and
fails.

## Solution

The solution is to use Microsoft’s LDAP client, `ldp.exe`, to move the FSMO role from the forcefully removed domain
controller to the domain controller about to be demoted. (The FSMO role will then be transferred automatically to a
remaining domain controller during the demotion process.)

1.  Press the Windows key, type `ldp.exe` and press Enter. (The computer must have the Active Directory Domain Services
    Administration Tools installed.)
2.  Open the *Connection* menu and click *Connect...*
3.  Input `olddc.ad.example.com` as the *Server*, 389 as the *Port*, and uncheck *Connectionless* and *SSL*.
4.  Open the *Connection* menu and click *Bind...* (shortcut: Ctrl+B)
5.  If the current Windows user has the necessary privileges (Domain Admin and Enterprise Admin in the relevant forest),
    choose *Bind as currently logged on user*. Otherwise, choose *Bind with credentials* and fill out the *User*,
    *Password* and *Domain* fields accordingly. Next, click *OK* (in either case) to log in.
6.  Open the *View* menu and click *Tree*. (shortcut: Ctrl+T)
7.  As the *BaseDN*, choose the option `CN=Configuration,DC=ad,DC=example,DC=com`. Click *OK*.
8.  In the tree structure on the left, double-click and open the following objects (the object names become longer by
    one entry on the left; the following list only lists this new entry):
    * `CN=Configuration,DC=ad,DC=example,DC=com`
    * `CN=Sites,[...]`
    * `CN=MySite,[...]`
    * `CN=Servers,[...]`
    * `CN=OLDDC,[...]`
    * `CN=NTDS Settings,[...]`
9.  With each double-click in the tree structure, the attributes of that object are appended to the text field on the
    right. The last object chosen was
    `CN=NTDS Settings,CN=OLDDC,CN=Servers,CN=MySite,CN=Sites,CN=Configuration,DC=ad,DC=example,DC=com`. Copy this full
    name (in bold text next to *Dn:*; Ctrl+C works as expected), without the *Dn:* or surrounding spaces, into Windows
    Notepad for later use.
10. Once again, open the *View* menu and click *Tree*. (shortcut: Ctrl+T)
11. Choose the naming context matching the object name in the warning message in the event log (e.g.
    `DC=DomainDnsZones,DC=ad,DC=example,DC=com`) and click *OK*.
    * The matching naming context is that with which the object name in the warning message in the event log ends.
    * If multiple naming contexts match (e.g. `DC=ad,DC=example,DC=com`, `CN=Configuration,DC=ad,DC=example,DC=com` and
      `CN=Schema,CN=Configuration,DC=ad,DC=example,DC=com`), always choose the longest matching one.
12. Walk through the tree structure as above until arriving at the object mentioned in the warning message in the event
    log (e.g. `CN=Infrastructure,DC=DomainDnsZones,DC=ad,DC=example,DC=com`).
13. Verify that the value of the attribute `fSMORoleOwner` matches the object name of the NTDS settings of the
    forcefully removed domain controller
    (`CN=NTDS Settings\0ADEL:8d91e2ae-dada-4459-9ec7-c8e58b255cb7,CN=DCGRAVE\0ADEL:a6004b55-6988-4886-b027-31827f5f8f83,CN=Servers,CN=MySite,CN=Sites,CN=Configuration,DC=ad,DC=example,DC=com`).
    * We will transfer the FSMO role by changing this attribute’s value to the object name of the NTDS settings of the
      domain controller that is about to be demoted.
14. Right-click the object mentioned in the warning message in the event log and click *Modify*. (This correctly
    pre-fills the *DN* field.)
15. In the *Edit Entry* section, as the *Attribute*, type `fSMORoleOwner`, and as the *Values*, paste in the object name
    copied into Notepad above – the object name of the NTDS settings of the domain controller that is about to be
    demoted (`CN=NTDS Settings,CN=OLDDC,CN=Servers,CN=MySite,CN=Sites,CN=Configuration,DC=ad,DC=example,DC=com`). Under
    *Operation*, choose *Replace*. Click *Enter* (the button, not the keyboard key!).
16. Verify that a matching entry has been added to the list box in the *Entry List* section.
17. Check the *Synchronous* check box and uncheck the *Extended* check box. Click the *Run* button.
18. Verify in the text field on the right that the modification has been successful
    (`Modified "CN=Infrastructure,DC=DomainDnsZones,DC=ad,DC=example,DC=com"`).
    * If the modification fails with `SvcErr: DSID-03152BF7`, it must be performed on a different domain controller. Try
      the domain controller holding the *Infrastructure Master* FSMO role for the afflicted domain, then all the others
      in the domain in turn. Open the *Connection* menu and click *Disconnect*, then follow steps 2 through 5, typing in
      the hostname of the domain controller you wish to contact. Finally, follow steps 10 through 18 again to reset the
      FSMO owner.
19. Double-click on the object in the tree strucutre once more. The attribute `fSMORoleOwner` should now have the new
    value (the object name of the NTDS settings of `OLDDC`).
20. Attempt to demote `olddc.ad.example.com` again. It should now see itself as the FSMO role owner of that partition
    and does not try to communicate about it with `dcgrave.ad.example.com`. The role is transferred to
    `newdc.ad.example.com` and the demotion should be successful.
