# LSASS Access from Non System Account

## Playbook ID

WINCRED1701052210

## Author

Roberto Rodriguez [@Cyb3rWard0g](https://twitter.com/Cyb3rWard0g)

## Tactic Name(s)

Credential Access

## Technique ID(s)

T1003

## Applies To

## Playbook References

## Technical Description

After a user logs on, a variety of credentials are generated and stored in the Local Security Authority Subsystem Service (LSASS) process in memory. This is meant to facilitate single sign-on (SSO) ensuring a user isn’t prompted each time resource access is requested. The credential data may include Kerberos tickets, NTLM password hashes, LM password hashes (if the password is <15 characters, depending on Windows OS version and patch level), and even clear-text passwords (to support WDigest and SSP authentication among others.

Adversaries look to get access to the credential data and do it so by finding a way to access the contents of memory of the LSASS process. For example, tools like Mimikatz get credential data by listing all available provider credentials with its `SEKURLSA::LogonPasswords` module. The module uses a Kernel32 function called OpenProcess to get a handle to lsass to then access LSASS and dump password data for currently logged on (or recently logged on) accounts as well as services running under the context of user credentials.

Even though most adversaries might inject into a System process to blend in with most applications accessing LSASS, there are ocassions where adversaries do not elevate to System and use the available administrator rights from the user since that is the minimum requirement to access LSASS.

## Permission Required

Administrator

## Hypothesis

Adversaries might be using a non system account to access LSASS and extract credentials from memory.

## Attack Simulation Dataset

| Environment| Name | Description |
|--------|---------|---------|
| [Shire](https://github.com/Cyb3rWard0g/mordor/tree/acf9f6be6a386783a20139ceb2faf8146378d603/environment/shire) | [empire_mimikatz_logonpasswords](https://github.com/Cyb3rWard0g/mordor/blob/master/small_datasets/windows/credential_access/credential_dumping_T1003/credentials_from_memory/empire_mimikatz_logonpasswords.md)  | A mordor dataset to simulate execution of Mimikatz module LogonnPasswords from non-system account |

## Recommended Data Sources

| Event ID | Event Name | Log Provider | Audit Category | Audit Sub-Category | ATT&CK Data Source |
|---------|---------|----------|----------|---------|---------|
| [4663](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/security/events/event-4663.md) | An attempt was made to access an object | Microsoft-Windows-Security-Auditing | Object Access | Kernel Object | Windows Event Logs |
| [4656](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/security/events/event-4656.md) | A handle to an object was requested | Microsoft-Windows-Security-Auditing | Object Access | Kernel Object | Windows Event Logs |
| [10](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/sysmon/event-10.md) | Process Access | Microsoft-Windows-Sysmon | | | Process Monitoring |
| [7](https://github.com/Cyb3rWard0g/OSSEM/blob/master/data_dictionaries/windows/sysmon/event-7.md) | Module Loaded | Microsoft-Windows-Sysmon | | | Loaded DLLs |

## Data Analytics

| FP Rate | Source | Analytic Logic | Description |
|--------|---------|---------|---------|
| Medium | Security | SELECT `@timestamp`, computer_name, SubjectUserName, ProcessName, ObjectName, AccessMask, event_id FROM mordor_file WHERE channel = "Security" AND (event_id = 4663 OR event_id = 4656) AND ObjectName LIKE "%lsass.exe" AND NOT SubjectUserName LIKE "%$" | Look for non-system accounts getting a handle and access lsass |
| Medium | Sysmon | SELECT `@timestamp`, computer_name, SourceImage, TargetImage, GrantedAccess FROM mordor_file WHERE channel = "Microsoft-Windows-Sysmon/Operational" AND event_id = 10 AND TargetImage LIKE "%lsass.exe" AND CallTrace LIKE "%UNKNOWN%" | Processes opening handles and accessing Lsass with potential dlls in memory (i.e UNKNOWN in CallTrace) |
| Medium | Sysmon | SELECT ProcessGuid,Image, COUNT(DISTINCT ImageLoaded) AS hits FROM mordor_file WHERE channel = channel = "Microsoft-Windows-Sysmon/Operational" AND event_id = 7 AND ( ImageLoaded LIKE "%samlib.dll" OR ImageLoaded LIKE "%vaultcli.dll" OR ImageLoaded LIKE "%hid.dll" OR ImageLoaded LIKE "%winscard.dll" OR ImageLoaded LIKE "%cryptdll.dll") AND `@timestamp` BETWEEN "2019-03-00 00:00:00.000" AND "2019-06-20 00:00:00.000" GROUP BY ProcessGuid,Image ORDER BY hits DESC LIMIT 10 | Look for processes loading a few known DLLs loaded by tools like Mimikatz to interact with credentials |
| Medium | Sysmon | SELECT a.`@timestamp`, a.computer_name, m.Image, a.SourceProcessGUID FROM mordor_file a INNER JOIN ( SELECT ProcessGuid,Image, COUNT(DISTINCT ImageLoaded) AS hits FROM mordor_file WHERE channel = "Microsoft-Windows-Sysmon/Operational" AND event_id = 7 AND (ImageLoaded LIKE "%samlib.dll" OR ImageLoaded LIKE "%vaultcli.dll" OR ImageLoaded LIKE "%hid.dll" OR ImageLoaded LIKE "%winscard.dll" OR ImageLoaded LIKE "%cryptdll.dll") AND `@timestamp` BETWEEN "2019-03-00 00:00:00.000" AND "2019-06-20 00:00:00.000" GROUP BY ProcessGuid,Image ORDER BY hits DESC LIMIT 10) m ON a.SourceProcessGUID = m.ProcessGuid WHERE a.channel = "Microsoft-Windows-Sysmon/Operational" AND a.event_id = 10 AND a.TargetImage LIKE "%lsass.exe" AND a.CallTrace LIKE "%UNKNOWN%" AND m.hits >= 3 | Join processes opening a handle and accessing LSASS with potential DLLs loaded in memory and processes loading a few known DLLs loaded by tools like Mimikatz to interact with credentials |
| Medium | Sysmon | SELECT p.`@timestamp`, p.computer_name, p.Image, p.User FROM mordor_file p INNER JOIN potential_mimikatz m ON p.ProcessGuid = m.SourceProcessGUID WHERE p.channel = "Microsoft-Windows-Sysmon/Operational" AND p.event_id = 1 AND NOT p.LogonId = "0x3e7 | Join non system accounts creating processes that open handles and access LSASS with potential DLLs loaded in memory and load a few known DLLs loaded by tools like Mimikatz to interact with credentials on ProcessGuid values|

## False Positives

## Detection Blind Spots

* The Microsoft Monitoring Agent binary pmfexe.exe is one of the most common ones that accesses Lsass.exe with at least 0x10 permissions as System. That could be useful to blend in through the noise.

## Hunter Notes

* Looking for processes accessing LSASS with the 0x10(VmRead) rights from a non-system account is very suspicious and not as common as you might think.
* GrantedAccess code 0x1010 is the new permission Mimikatz v.20170327 uses for command "sekurlsa::logonpasswords". You can specifically look for that from processes like PowerShell to create a basic signature.
  * 0x00000010 = VMRead
  * 0x00001000 = QueryLimitedInfo
* GrantedAccess code 0x1010 is less common than 0x1410 in large environment.
* Out of all the Modules that Mimikatz needs to function, there are 5 modules that when loaded all together by the same process is very suspicious.
    * samlib.dll, vaultcli.dll, hid.dll, winscard.dll, cryptdll.dll
* For signatures purposes, look for processes accessing Lsass.exe with a potential CallTrace Pattern: /C:\\Windows\\SYSTEM32\\ntdll.dll\+[a-zA-Z0-9]{1,}\|C:\\Windows\\System32\\KERNELBASE.dll\+[a-zA-Z0-9]{1,}\|UNKNOWN.*/
* You could use a stack counting technique to stack all the values of the permissions invoked by processes accessing Lsass.exe. You will have to do some filtering to reduce the number of false positives. You could then group the results with other events such as modules being loaded (EID 7). A time window of 1-2 seconds could help to get to a reasonable number of events for analysis.

## Hunt Output

## Referennces

* https://tyranidslair.blogspot.com/2017/10/bypassing-sacl-auditing-on-lsass.htmls
* https://adsecurity.org/?page_id=1821#SEKURLSALogonPasswords
* https://github.com/PowerShellMafia/PowerSploit/tree/dev
* http://clymb3r.wordpress.com/2013/04/09/modifying-mimikatz-to-be-loaded-using-invoke-reflectivedllinjection-ps1/