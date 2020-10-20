---
layout: post
title: "Command-line Auditing on Windows: Why You Need It!"
date: 2020-10-16 04:00:00 +0500
categories: forensics windows command-prompt
---

It’s unfortunate that the Windows Command Prompt, the descendant of the prehistoric command.com from MS-DOS, has no persistent storage of command execution. It does, however, support temporary storage of commands executed in an active session. So, if an attacker proceeds to enumerate other hosts or ex-filtrate data using a console window, does a defender have no means of identifying the execution of a binary? Well, it’s not entirely true.

## Auditing Process Creations

Windows does support command-line auditing or process creation auditing to some extent. The event ID, 4688, is widely recognized on Windows operating systems for “Process Creation”. Although the auditing for process creation is disabled by default, it can be easily enabled through the Local Security Policy (including a few other means). Along with the creation of processes, these events can also be tweaked to include command line arguments. These events are great to identify the source of hideous command prompt launches, identify executables run on an endpoint, or track an adversary’s activities.

Here’s how you can enable ‘process creation’ auditing using the Local Security Policy. The settings are under “Advanced Audit Policy Configuration”, followed by “Detailed Tracking”, and then under “Audit Process Creation”. Based on your system’s log storage capabilities, enable the “Success” and “Failure” of process creation logs.

![process-creation](/assets/win-cmd-history/process-creation.png)

This isn’t all. Process creation is not enough on its own to decide the usage of an executable. It’s why you should also log its command line usage and arguments. Though it should be noted that the logging can enable any and all sorts of data including passwords or personal information. To proceed with command line logging in process creations: Open up “Local Computer Policy” or the “Group Policy Editor”, head to “Computer Configuration, then to “Administrative Templates”, then to “System”, and finally under “Audit Process Creation”, double click “Include command line in process creation events”. Enable the setting and press “OK” to continue.

Now that you’re done enabling process auditing for your system — it’s time to monitor what sort of data it logs. Here’s an example log:

![process-creation](/assets/win-cmd-history/cmdline-proc-creation.png)


The “Process Command Line” attribute has the data you required initially. This can be incredibly helpful for defenders to monitor and identify possibly malicious activity. Secondly, process creation logging can help them identify the execution of binaries which shouldn’t exist on a particular system. For example, most modern ransomware initially remove Shadow copies using the tool ‘vssadmin.exe’. If you forward your logs to a SIEM or ingest it in a data search engine like ElasticSearch, you can easily look for vssadmin — a tool a normal user would seldom use in day-to-day activities.

![process-creation](/assets/win-cmd-history/vssadmin.png)

Here’s a generic command (PowerShell) to display all spawned processes along with their command line arguments:

    Get-WinEvent -FilterHashtable @{
        LogName = 'Security'
        ID = 4688
    } | Select-Object TimeCreated,@{name='NewProcessName';expression={ $_.Properties[5].Value }}, @{name='CommandLine';expression={ $_.Properties[8].Value }}

## Commands Mostly Abused by Attackers

Based on a report released by Shusei Tomonaga from the Analysis Center at JPCERT, attackers commonly utilize a specific set of commands from the initial stages of recon to the final stages of impact.

### Initial Investigation
Firstly, an attacker usually goes through the ‘sniffing’ phase. This is where the attacker would query the system for more information, seek information about processes, services, time, IP configurations, etc. Here’s a list of commands most commonly utilized in the Initial Investigation phase:

![process-creation](/assets/win-cmd-history/top10_1.png)

### Reconnaissance
Next, the attackers would begin snooping on other machines and resources on the network, seek confidential information, and more information relevant to the compromises host and its neighbors. Here’s a list of commands most commonly utilized during the Recon phase:

![process-creation](/assets/win-cmd-history/top10_2.png)

### Spread of Infection
This is where the attacker would decide to use the collected information and begin spreading malware within the network. For that purpose, the following commands are used most commonly:

![process-creation](/assets/win-cmd-history/top10_3.png)

Using commands like ‘at’ and ‘wmic’, attackers can do much more than local execution. With proper access rights, the adversary can easily execute anything on remote hosts.

## Restricting Execution of Blacklisted Commands
If you take a closer look at the list above, these are highly technical commands. It’s obvious an oblivious end-user would never make use of them. It’s fairly easy to restrict these applications on endpoints to avoid mass-scale damage. AppLocker configurations can work perfectly in such cases.

### AppLocker Configuration

AppLocker can help allow or deny the execution of executables based on defined rules. Using these simple rules, we can do two things — deny the execution of malware and record the failure of execution for commands in the event log (AppLocker has its own channel which we’ll explore later). To start configuring AppLocker rules, visit the Local Security Policy again. Under Application Control Policies, head to AppLocker. Under Executable Rules, let’s create a new rule to deny the usage of a sample command — ‘whoami’ (completely harmless).

![process-creation](/assets/win-cmd-history/pathbased.png)

Before the selection of the executable, you can also select the entities you wish to apply this rule to. Also, apart from path-based execution blockage, you can also apply it based on hashes or the sign status of the executable. An even more strategic move would be to block the command prompt — even though this will only partially contribute to halting a malware’s execution. Secondly, a few casual users are also Windows-savvy and can make use of the console to perform an operation or two. Should you really block the command prompt? That’s up to you. How to do it? Here’s that:
- Open up the Group Policy Editor again
- Shift to User Configuration
- Then to Administrative Templates
- Finally, under ‘System’, look for the “Prevent access to the command prompt” setting. Enable or disable it based on your need.

## Conclusion
That’s it for this article. Process creation and command-line auditing is definitely a mandatory for all environments. Though the decision to do so comes with its own issues — the most significant being the size and nature of these logs (abundant in nature). If it’s viable for you to do so, the decision will be fruitful in case of incidents or investigations.