---
layout: post
title: "Windows API Calls: Process Listing APIs"
date: 2020-04-30 10:00:00 +0500
categories: forensics windows api-calls
---

Let's continue our series on Windows API Calls and today, we'll be discussing some of the many methods, Windows provides to enumerate the processes on a system.

# Process Enumeration

Why is process enumeration even important for a Red-teamer or a Blue-teamer? The simplest reason, would be to see what's running on the system. In this serie, we can either assume that the system is under the post-exploitation phase or is under assessment by an analyst.

A malicious actor would normally go forward with this step to identify what's running and then act appropriately. Most AV's, IPS, IDS, and networking solutions follow the same naming convention across different devices, and thus, identification of them isn't a big deal. Or perhaps, the machine runs a server process - what should be the next step? That's what process enumeration can help us decide.

Once you've identified the process, you can pick your next step. Process injection, thread creation, memory dumps, and token impersonation are some of the most common attacks one can carry (though it's a non-exclusive list of attacks).

# Windows API Calls for Processes

To carry out the enumeration step, Windows provides the following four techniques in order to successfully find out the list of running processes:

1. Process Status API (PSAPI or the 'EnumProcesses' API call)
2. Tool Help Library ('Process32First' and 'Process32Next' API call)
3. Remote Desktop Services ('WTSEnumerateProcesses' API call)
4. WMI using the COM (Component Object Model)

# Remote Desktop Service: WTSEnumerateProcessesEx

The method we'll be discussing today is the enumeration of processes via the 'WTSEnumerateProcesses' API call which is provided by the RDS or Remote Desktop Service. The 'WTS' in the API call name is meant to be 'Windows Terminal Services', an older name for RDS.

The API call allows us to enumerate processes on a specified Remote Desktop Session Host (RDSH) Server. RDSH servers are a role in the RDS environment, where they expose several applications and desktops to remote users. Users can then connect to them and perform their tasks.

## A Look at the API Call

![API Call](/assets/wts-enum-proc/Screenshot_2.png)

If you take a look at the API call, it's structured like this:

    BOOL WTSEnumerateProcessesExA(
    HANDLE hServer,
    DWORD  *pLevel,
    DWORD  SessionId,
    LPSTR  *ppProcessInfo,
    DWORD  *pCount
    );

The first argument, **hServer** takes in the handle to the RDSH server. However, there's a catch here. If you provide the value 'WTS_CURRENT_SERVER_HANDLE' to the parameter, it'll assume the system you're running the application on to be the server. 

The second argument, **pLevel** takes in a pointer to a DWORD variable, which will suggest what type of information should the call return. You have two possible options here:
   
1. **WTS_PROCESS_INFO (0)** - Normal, lesser information is returned in this structure

![ProcessInfo](/assets/wts-enum-proc/wts-proc-info.png)

2. **WTS_PROCESS_INFO_EX (1)** - Extended, more information is returned in this structure

![ProcessInfo](/assets/wts-enum-proc/wts-proc-info-ex.png)

The third parameter is the **sessionId**, using which you can filter in on a particular session and exclude all other terminal services. If you wish to enumerate processes on all sessions, you can use the **WTS_ANY_SESSION** structure.

The fourth argument, **ppProcessInfo**, is a pointer to a variable which in turn receives a pointer to an array of structures, which contain our actual processes returned by the **WTS_PROCESS_INFO or *_EX**.

Finally, the **pCount** variable contains the actual number of processes (or process structures) returned by the ppProcessInfo parameter.

# Implementation

As for the implementation, I won't be referencing my own code for now. Rather, there's an amazing PoC available for use on CodeExpert.io which we can use to understand the entire method. I'll link them here, if you wish to go check their code out: _[code-expert]{:target="_blank"}_

    // Include the WTS Header Library
    #include <Wtsapi32.h>
    #pragma comment(lib, "Wtsapi32.lib")
    
    DWORD RDSAPI_EnumProcesses(CArray<WTS_PROCESS_INFO>& arrProcInfo)
    {
        // Clear the output array
        arrProcInfo.RemoveAll();

        // Set the parameters
        HANDLE hServer = WTS_CURRENT_SERVER_HANDLE; // local machine processes
        PWTS_PROCESS_INFO pProcessInfo = NULL;
        DWORD dwCount = 0;
        // enumerate processes
        if(!::WTSEnumerateProcesses(hServer, 0, 1, &pProcessInfo, &dwCount))
        {
            return ::GetLastError();
        }

        // fill output array
        arrProcInfo.SetSize(dwCount);
        for(DWORD dwIndex = 0; dwIndex < dwCount; dwIndex++)
        {
            arrProcInfo[dwIndex] = pProcessInfo[dwIndex];
        }

        // free the memory allocated in WTSEnumerateProcesses
        ::WTSFreeMemory(pProcessInfo);
        return NO_ERROR;
    }

If you've understood the parameters, the implementation of the API call was fairly simple. The parameters are initialized before hand and then passed off to our WTSEnumerateProcesses call. Next, the output array's size is allocated to be the size of the ppProcessInfo array. Once done, the dwCount variable (the pCount) is used to loop over the list of processes. Once all are assigned to our own variable, the memory is freed for the API call.  

# Extracting the User SID

As we can see, the program makes use of the WTS_PROCESS_INFO_EX structure by stating the pLevel parameter to be '1'. Under this structure, we also have several the field 'pUserSid' - which returns the user security identifier responsible for the process. We can apply filters based on this SID or the user to identify the user's processes alone.

When a user signs in, the SID is placed in the access token of the user which is then associated with all activities which the user performs. Similarly, the access token also contains the security context along with the privileges assigned to a particular user. So, basically, all processes created by a user under a given session have the same access token.

We can extract the user SID from the token, as well as the process just returned to us. We'll go with the latter for now. Once picked, the SID has to be converted into a String. We can use the **ConvertSidtoStringSid** in order to receive a string-based SID. 

![ConvertSid](/assets/wts-enum-proc/convert-sid.png)

Furthermore, we can use the **LookupAccountSid** API call to extract the username and domain of a user from the SID we just extracted. This should give you more information about the user who's behind a particular process.

# Lack of Privileges

If you run the program (which I haven't included here), and attempt to identify the users and domains under normal execution, not all processes are going to return a result. Why? If you take a look at the official documentation for [WTSEnumerateProcesses], the remarks section suggests "The caller must be a member of the Administrators group to enumerate processes that are running under another user session.".

![ProcessInfo](/assets/wts-enum-proc/list-proc-no-admin.png)

If we re-run the program as administrator, you'll see way more usernames and domain names pop up in the results. 

![ProcessInfo](/assets/wts-enum-proc/list-proc-adm.png)

So, what changed? Privileges. As an administrator, you have much more privileges, especially including the '**SeDebugPrivilege**', which allows you to debug programs. By default, this privilege is disabled when you run as an administrator. Programmatically, we can enable this setting as well. This would allow you program to shift its privileges during run-time and not wait for external action.

# Privilege Escalation via AdjustTokenPrivileges

Let's first make a mind-map of the steps we'll follow to escalate our privileges. Here's how we do this:

1. First, we identify the availability of the 'SeDebugPrivilege' by using the **LookupPrivilegeValue** API call, which retrieves the LUID (Locally Unique Identifier)
2. Second, we make use of the **OpenProcessToken** API call to open up the token for a certain process. Which process should that be? The calling process. We can use the **GetCurrentProcess** API call to get a handle to the current process. Now, we have a handle to the access token of the calling function
3. Here, we can see the privileges which the user has and use the **AdjustTokenPrivileges** API call to escalate the privileges or enable the SeDebugPrivilege, if that's all we need from it.

There's a shortcoming with the AdjustTokenPrivilege API call. If you use a privilege which isn't available in the access token with this call, the call **won't fail.** Instead, it succeeds and thus, needs more error-chekcing. Rather than blindly trusting the token to have a particular privilege, what we can do is, we can enumerate the list of privileges for the user by calling the **GetTokenInformation** API call and then enumerating over the list of privileges. 

# Requesting Elevation 

Most programs do show up the UAC prompt in order to request elevation and add on more privileges than what the normal access token assigns. These settings are usually defined in the Manifest file, which is an XML-based metadata file for other files in your program. 

Other than this, you can make use of the **ShellExecuteEx** API call to perform an action on a particular file. To perform it on the executable, we first need it's proper path. Use the **GetModuleFileName** API call to retrieve the path and then, let's pass it on to the ShellExecuteEx call. The call takes in a structure of the **SHELLEXECUTEINFOA** type, which has several information about the file and operation to perform. 

![ProcessInfo](/assets/wts-enum-proc/shell-exec.png)

Here, the **lpVerb** has to be **runas** in order for your program to request elevation. Similarly, the **lpFile** has to be the result of your ModuleFileName API.

# Conclusion

That's it for today folks. It was interesting how the Remote Desktop Service was able to fetch processes for the loal system. We'll discuss the rest of the process enumeration techniques tomorrow. Till then, take care. 


[code-expert]: http://codexpert.ro/blog/2013/12/01/listing-processes-part-4-using-remote-desktop-services-api/
[WTSEnumerateProcesses]: https://docs.microsoft.com/en-us/windows/win32/api/wtsapi32/nf-wtsapi32-wtsenumerateprocessesexa#remarks