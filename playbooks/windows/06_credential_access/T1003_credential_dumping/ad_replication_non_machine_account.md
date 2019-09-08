# Active Directory Replication From Non-Domain-Controller Accounts

## Playbook Tags

**ID:** WINCRED1808152105

**Author:** Roberto Rodriguez [@Cyb3rWard0g](https://twitter.com/Cyb3rWard0g)

**References:** WINEXEC1904101010

## ATT&CK Tags

**Tactics:** Credential Access

**Techniques:** Credential Dumping (T1003)

## Applies To

## Technical Description

Active Directory replication is the process by which the changes that originate on one domain controller are automatically transferred to other domain controllers that store the same data.

Active Directory data takes the form of objects that have properties, or attributes. Each object is an instance of an object class, and object classes and their respective attributes are defined in the Active Directory schema. The values of the attributes define the object, and a change to a value of an attribute must be transferred from the domain controller on which it occurs to every other domain controller that stores a replica of that object.

An adversary can abuse this model and request information about a specific account via the replication request. This is done from an account with sufficient permissions (usually domain admin level) to perform that request. Usually the accounts performing replication operations in a domain are computer accounts (i.e dcaccount$). Therefore, it might be abnormal to see other non-dc-accounts doing it.

### Additional Reading

* [Active Directory Replication](https://github.com/Cyb3rWard0g/ThreatHunter-Playbook/tree/master/library/active_directory_replication.md)

## Permission Required

Domain Admin

## Hypothesis

Adversaries might attempt to pull the NTLM hash of a user via active directory replication apis from a non-domain-controller account with permissions to do so.

## Attack Simulation Dataset

| Environment| Name | Description |
|--------|---------|---------|
| [Shire](https://github.com/Cyb3rWard0g/mordor/tree/acf9f6be6a386783a20139ceb2faf8146378d603/environment/shire) | [empire_dcsync](https://github.com/Cyb3rWard0g/mordor/blob/master/small_datasets/windows/credential_access/credential_dumping_T1003/credentials_from_ad/empire_dcsync.md)  | A mordor dataset to simulate execution of DCSync from a non domain controller account |

## Recommended Data Sources

| Event ID | Event Name | Log Provider | Audit Category | Audit Sub-Category | ATT&CK Data Source |
|---------|---------|----------|----------|---------|---------|
| [4662](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/security/events/event-4662.md) | An operation was performed on an object | Microsoft-Windows-Security-Auditing | DS Access | Directory Service Access | Windows Event Logs |
| [4624](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/security/events/event-4624.md) | An account was successfully logged on | Microsoft-Windows-Security-Auditing | Audit Logon/Logoff | Audit Logon | Windows Event Logs |

## Data Analytics

| FP Rate | Source | Analytic Logic | Description |
|--------|---------|---------|---------|
| Low | Security | SELECT `@timestamp`, computer_name, SubjectUserName FROM mordor_file WHERE channel = "Security" AND event_id = 4662 AND AccessMask = "0x100" AND (Properties LIKE "%1131f6aa_9c07_11d1_f79f_00c04fc2dcd2%" OR Properties LIKE "%1131f6ad_9c07_11d1_f79f_00c04fc2dcd2%" OR Properties LIKE "%89e95b76_444d_4c62_991a_0facbeda640c%") AND NOT SubjectUserName LIKE "%$" | Monitoring for non-dc machine accounts accessing active directory objects on domain controllers with replication rights might be suspicious |
| Low | Security | SELECT o.`@timestamp`, o.computer_name, o.SubjectUserName, o.SubjectLogonId, a.IpAddress FROM mordor_file o INNER JOIN ( SELECT computer_name,TargetUserName,TargetLogonId,IpAddress FROM mordor_file WHERE channel = "Security" AND LogonType = 3 AND IpAddress is not null AND NOT TargetUserName LIKE "%\\$" ) a ON o.SubjectLogonId = a.TargetLogonId WHERE o.channel = "Security" AND o.event_id = 4662 AND o.AccessMask = "0x100" AND (o.Properties LIKE "%1131f6aa_9c07_11d1_f79f_00c04fc2dcd2%" OR o.Properties LIKE "%1131f6ad_9c07_11d1_f79f_00c04fc2dcd2%" OR o.Properties LIKE "%89e95b76_444d_4c62_991a_0facbeda640c%") AND o.computer_name = a.computer_name AND NOT o.SubjectUserName LIKE "%$" | You can use successful authentication events on the domain controller to get information about the source of the AD Replication Service request |

## False Positives

## Detection Blind Spots

* Adversaries could perform the replication request from a Domain Controller (DC) and with the DC machine account (*$) to make it look like usual/common replication activity. 

## Hunter Notes

* As stated before, when an adversary utilizes directory replication services to connect to a DC, a RPC Client call operation with the RPC Interface GUID of `E3514235-4B06-11D1-AB04-00C04FC2DCD2` is performed pointing to the targeted DC. This activity is recorded at the source endpoint, from where the replication request is being performed from, via the `Microsoft-Windows-RPC` ETW provider. Even though the `Microsoft-Windows-RPC` ETW provider provides great visibility for replication operations at the source endpoint level, it does not have an event tracing session writing events to an .evtx file which does not make it practical to use it at scale yet.
* You can collect information from the `Microsoft-Windows-RPC` ETW provider and filter on RPC Interface GUID `E3514235-4B06-11D1-AB04-00C04FC2DCD2` with SilkETW. One example here: https://twitter.com/FuzzySec/status/1127249052175872000
* You can correlate security events 4662 and 4624 (Logon Type 3) by their Logon ID on the Domain Controller (DC) that received the replication request. This will tell you where the AD replication request came from. This will also allow you to know if it came from another DC or not.

## Hunt Output

| Category | Type | Name |
|--------|---------|---------|
| Signature | Sigma Rule | [win_ad_replication_non_machine_account.yml](https://github.com/Cyb3rWard0g/ThreatHunter-Playbook/tree/master/signatures/sigma/win_ad_replication_non_machine_account.yml) |

## References

* https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1522b774-6464-41a3-87a5-1e5633c3fbbb
* https://docs.microsoft.com/en-us/windows/desktop/adschema/c-domain
* https://docs.microsoft.com/en-us/windows/desktop/adschema/c-domaindns
* http://www.harmj0y.net/blog/redteaming/a-guide-to-attacking-domain-trusts/
* https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc782376(v=ws.10)
* https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/f977faaa-673e-4f66-b9bf-48c640241d47