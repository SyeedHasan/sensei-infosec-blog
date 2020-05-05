---
layout: post
title: "Windows DLLs: Attacks in a Nutshell"
date: 2020-05-05 11:00:00 +0500
categories: forensics dll-attacks
---

# What are DLLs? 

Dynamic-link Libraries (DLLs) are Microsoft's implementation of shared code on the Windows Operating System. By means of modularizing code into smaller segments and individual files, Windows applications can utilize this shared code. This allows them to avoid including the same piece of code, again and again.

Usually, the functions written in a DLL file are exportable. The **DllMain** function in a particular file carries out the basic tasks, whereas the individual functions can then be imported into code as well. For example, you can load the ws2_32.dll library by using the **LoadLibrary** API call and then, make use of the **GetProcAddress** to get the address to the particular function you're looking for (e.g. connect).

DLLs can be classified into two main types:
- System
- Application

System DLLs are available on your OS for healthy functioning and is code provided by Microsoft itself. For example, the ws2_32.dll is used to establish connections by means of sockets. Similarly, the application DLLs are created by individual developers to ship modular code or functionality.

# DLL Search Order

When you code your program to look for a particular DLL file and load it in, there's a particular order that it follows to look for that DLL file. This order is called the "DLL Search Order". By default, the order is specified as follows:
1. Search the directory from which the EXE image loads
2. Search the current directory 
3. Search the system directory (available to you by **GetSystemDirectory()**)
4. Search the Windows directory (available to you by **GetWindowsDirectory**)
5. Search the environment variables (PATH)

Fairly simple search order right? 

# Introducing 'DLL Hijacking'

What if I replace a legitimate DLL which comes up at the fourth point, with a malicious DLL at the first point? My program will look for the DLL but find my malicious DLL before the legitimate one. I can run whatever I want via this DLL file! 

![DLL Hijacking](/assets/dll-attacks/hijacking.png)

The attacker would usually identify an application which suffers from this vulnerability - DLL's are loaded without a fully qualified path. Then, incite the user to open an application which can trigger the loading of a malicious DLL, and the actor's goals being met.

How does one go about finding such vulnerable DLLs?

Missing DLL's are the worst thing. If you have a look at processes which are running but their DLL's aren't being found, that's one way to start your hijacking activity. Secondly, if you closely look at ProcMon (Process Monitor), provided by SysInternals, for the DLL's which are loaded into a process' address space, you'll find out the paths to those DLL's as well. If it isn't a system DLL which is necessary for it to operate, you can replace this DLL up the search order by placing your own DLL. 

Lastly, let's say your program **abc.exe** attempts to load a DLL from the path **'C:\Perl64** which is used by Perl. However, there's no matching DLL here. You've identified this to be the source for your DLL hijacking process. Mainly because of its placement in your file system, the folder will probably have higher privileges and anything running under it wiil have much more rights. If you do have Write permissions to this folder, you can place your DLL here, by replacing the original. This opens room for **Privilege Escalation!**

Generate a malicious DLL using Metasploit with a reverse-shell payload, place it at your Executable's directory, and let's execute it. If you've properly identified a vulnerable DLL, the shell session should now be established. 

# AppInit DLLs

Using AppInit DLLs, you can load custom DLLs to be loaded into the address space of every possible process. This is mostly used to hook into the system API and implement alternate functionality - but often used by malware to load malicious DLLs. 

If a DLL is loaded from here, it'll run under the context of that process and can be used to acquire persistence and privilege escalation. Though by default, the AppInit_DLLs are disabled on latest Windows to avoid this mal-behavior.

![DLL Injection](/assets/dll-attacks/injection.jpg)

The **LoadAppInit_DLLs** setting decides where AppInit_DLLs are utilized or not. A '0' by default suggests the feature is disabled. If enabled, you can set a custom DLL over at the AppInit_DLLs registry key and load your malicious DLL alongside every other user-mode process (since user32.dll will load these DLLs if the value is enabled).

You can check this setting out in the registry by going to this key: 

    Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows

So, steps being:
1. Generate a malicious DLL
2. Set the LoadAppInit_DLLs setting to 'True'
3. Set the 'AppInit_DLL's to point to the malicious DLL
4. Injection is ready!

Once you start running processes, the DLL will be injected into every process. This can create noise and several sessions on the attacker's machine, so the process isn't the most favorite but capable of compromising a machine.

# DLL Forwarding

Lastly, **DLL Forwarding or DLL Proxying** can be used to load a malicious DLL instead of a legitimate one but only for a few exportable functions. Here's how you do it. First, replace the legitimate DLL with a trojanized version, simply renaming your mal.dll to the original. Now, the trojanized DLL only implements a few functions, and if the rest are called from the original DLL, it passes them on to the original DLL (which you've renamed to something else). So, basically, your trojanized DLL acts as part of a proxy, forwarder, or intercepter which can then act as "Man in the Middle". This will ensure the actor's goal is met, and the functionality isn't reduced or broken.

Here's the step-by-step guide to this DLL attack:

1. Analyze a DLL and the functions it has
2. Identify the functions you'll be intercepting and the ones you'll be modifying
3. Implement those functions in the malicious DLL
4. Forward all remaining functions to the original DLL
5. Rename the original DLL to anything else
6. Forward the imports to this corrected file name
7. Replace the original DLL with your malicious DLL and rename it to what the original DLL's first name was

You can identify the exports of a file from its export address table and create a DEF (module definition) file for forwarding information.

# Mitigation of Attacks

As interesting as the attacks sound, they're just as easy to thwart for a defender as well. It requires you to setup proper mitigation techniques against loading of DLLs which are unsigned, malicious, or just appear out of the blue.

Here's a list of a few mitigations to avoid DLL-based attacks:

## Execution Prevention

By setting up appropriate whitelisting/blacklisting solutions, you can easily figure out the DLLs which are being loaded. This can also help you block DLLs which aren't normal for well-known processes. 

## Restrict Library Loading

Remote DLLs shouldn't be allowed in the system, in the first place. The **Safe DLL Search Mode** allows you to modify the DLL Search Order by going through the System directories well before the user's local directories for the DLL. This has proven to be a useful mitigation technique against DLL's aiming to bypass the search order.

The associated Windows Registry key for this is located at:

    HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\SafeDLLSearchMode

## Disable LoadAppInit_DLLs

By default, AppInit_DLLs are disabled. You should maintain this setting's initial value to avoid DLLs being injected into processes on your system. If you do disable this, the AppInit_DLL field gets removed or just its value is set to none to avoid a DLL being loaded, ever in the future.

# Conclusion

I've left out a few other DLL attack types, for example DLL Side-loading or Phantom DLL hijacking. We might probably raise these, in some future series. That's it for this article folks. Make sure you monitor your endpoints for malicious activities. SysInternals provides a good suite of tools which you can make use of as well. Autoruns is one such amazing tool to identify items which run during system startup or login.

Stay safe! 