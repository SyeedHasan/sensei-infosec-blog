---
layout: post
title: "Windows API Calls: More on Processes and Memory"
date: 2020-05-04 11:15:00 +0500
categories: forensics windows api-calls
---

In the previous article, I talked about the WTSEnumerateProcessesEx API call to enumerate processes on a local machine via the Windows Terminal (RDS) service. Today, we'll go over a few other API calls which can be used to enumerate processes, along with a short discussion on memory-intensive API calls.

# More Process Listing APIs

Let's discuss two other options which we can use to get a list of running processes on a system: 

## CreateToolHelp32Snapshot

As part of the ToolHelp library (tlhelp32.h), the **CreateToolHelp32Snapshot** offers a chance to take a snapshot of the processes, their heaps, threads, and modules (which are utilized by these processes). You can use the API for querying information about the processes on a minimal scale (just the ID's) and on a much more detailed scale (where you can request the heap, for example).

    HANDLE CreateToolhelp32Snapshot(
    DWORD dwFlags,
    DWORD th32ProcessID
    );

The API call is fairly simple. It takes in a DWORD **flags** field, which present the options you'd like to use during your query. And secondly, the **th32ProcessID** takes in a process ID, for which, you'd like to take a snapshot. Though you can use **0** in order to take a snapshot of the current process. Also, this field usually defaults to _all processes_, and a value is only required for a few special flag field values.

Mind you - as the API call suggests, the function only returns a handle to the snapshot. You'll be given read-only access and can further use the same command to target the heap, modules, or threads using the flags. Here's a list of the most important flag field values: (Taken from the official MSDN documentations)

| TH32CS_SNAPALL | Includes all processes and threads in the system, plus the heaps and modules of the process specified in th32ProcessID.|
| TH32CS_SNAPHEAPLIST | Includes all heaps of the process specified in th32ProcessID in the snapshot. To enumerate the heaps, see Heap32ListFirst. |
| TH32CS_SNAPMODULE | Includes all modules of the process specified in th32ProcessID in the snapshot. To enumerate the modules, see Module32First.|
| TH32CS_SNAPPROCESS | Includes all processes in the system in the snapshot. To enumerate the processes, see Process32First. |
| TH32CS_SNAPTHREAD | Includes all threads in the system in the snapshot. To enumerate the threads, see Thread32First. To identify the threads that belong to a specific process, compare its process identifier to the th32OwnerProcessID member of the THREADENTRY32 structure when enumerating the threads. |

Here's the pseudo-code to how you can extract the processes and then use the Process32First to loop over a list of the processes.

1. Get the processes using the **CreateToolHelp32Snapshot** API call with the '**SNAPPROCESS**' flag enabled
2. Using the **Process32First** API Call, get access to a single process. 
   1. Specify the **PROCESSENTRY32** structure for the output, and use a pointer to it in your Process32First call. This will also be used in subsequent calls.
3. Loop over the list of available processes using the handle to your snapshot and the PROCESSENTRY32 structure you've previously created, in the **Process32Next** API call.
4. Once done, you can destroy this snapshot by closing the handle and passing it on to the **CloseHandle** API call.

### Remarks

As a blue-teamer, one concern with this API call is the fact that it doesn't require privileges at all. You can easily query the system for the processes without acquiring the **SeDebugPrivilege**. This also makes it harder to detect or raise IDS suspicion alerts. 

Though there's the issue of the snapshot being a preserved state (not a live image of the system), it really isn't an issue. Since most AV, IDS, IPS, and EDR solutions make use of well-known or similar names across devices. The attacker only needs to know the name and can then send in suited bypass for the particular vendor-based security solution. Again, even this is possible, using Kernel mode drivers (but that's a different story for now).

The API call is commonly utilized by many malware families. The most prominent of them being the Zeus banking trojan. This is done to inject into a particular process, install import hooks, and identifying running threads. 

## Process Status API

The last API call that can be used to enumerate the processes on a system is the **EnumProcess** API call, which is provided as part of the PSAPI (Process Status API). However, this API call only returns the process ID's and not the process handles itself - though we can quickly open those up using other API calls (which we'll discuss).

Here's the syntax for the API call:

    BOOL EnumProcesses(
    DWORD   *lpidProcess,
    DWORD   cb,
    LPDWORD lpcbNeeded
    );

The **lpIdProcess** sets in a pointer to an array, in which the processes will be stored/retrieved. As a general rule of thumb, you should set this to a bigger size to accomodate to all processes on a system (since you're unaware of how much space it should require at the start). The **cb** field requires the size of the pProcessIds array, whereas the **pcbNeeded** returns the number of bytes actually returned in the pProcessIds array.

To identify the actual number of processes, you can divide the returned number of bytes by the size of DWORD. However, if the initial size was too low, the array would fill up, and leave no choice but fill it with as many processes as it could (possibly leaving a few). You should re-run your call with a more leniently sized array to bypass this issue. 

Here's a sample pseudocode you can utilize to query information about these processes:

1. Use the **EnumProcess** API call to get the Process IDs
2. Use the **OpenProcess** API call to get a handle to the actual process
3. Use the handle in other API calls like **OpenProcessToken** to get the access token for that particular process.

### Remarks

Privileges are required for the EnumProcess call to get access to all processes. You might also want to opt for API obfuscation to avoid this API call being easily available under static analysis. In comparison to the CreateToolHelp32Snapshot, this method is fairly noisy and is definitely capable of generating alarms on a well-protected system.

# Process Memory Operations

If you wish to peek at the memory or address space of a process, access its content, or even modify it, you can do so by utilizing memory-based API calls. 

## ToolHelp32ReadProcessMemory

Our first mention, **ToolHelp32ReadProcessMemory** is from the same library as the CreateToolHelp32Snapshot, tlhelp32.h. This API call can copy the requested segment of the memory into a buffer, that's supplied by the programmer. The general syntax for the call is:

    BOOL Toolhelp32ReadProcessMemory(
    DWORD   th32ProcessID,
    LPCVOID lpBaseAddress,
    LPVOID  lpBuffer,
    SIZE_T  cbRead,
    SIZE_T  *lpNumberOfBytesRead
    );

The first field is the **ProcessID** which is the process identifier for which the memory is being read (0 for the current process). Then, the **BaseAddress** field is the base address in the physical memory from which the read operation will begin. Mind you, you require read privileges otherwise the function call will fail. Then, you can specify the **buffer** in which the memory content will be stored and the **bRead** field, which will identify the number of bytes to read. Finally, you can use the **numberOfBytesRead** to identify the bytes read by the call. 

You should try to run this call with privileges and with the same user who spawned the process. Though running an escalated shell can bypass all such requirements. This can also help you identify strings in memory for a given process. 

## ReadProcessMemory

Similar to the ToolHelp32ReadProcessMemory call, the **ReadProcessMemory** (memoryapi.h) call can extract the contents of a processes' memory. The syntax for the call is as follows:

    BOOL ReadProcessMemory(
    HANDLE  hProcess,
    LPCVOID lpBaseAddress,
    LPVOID  lpBuffer,
    SIZE_T  nSize,
    SIZE_T  *lpNumberOfBytesRead
    );

Most of the parameters are the same if you've been reading from the top. However, the one difference is the handle to the process. Unlike our previous choice, which requested the PID to capture content, this API call doesn't require that and would work with the handle, which we can acquire using the **OpenProcess** API call along with the 'PROCESS_VM_READ' access to it. 

## WriteProcessMemory

The **WriteProcessMemory** (memoryapi.h) task does what its name suggests. It's going to write over a specified segment of the memory, but require privileges to do so. The syntax structure is the same as read memory - the difference being a write operation instead of a read operation. 

For the content of the buffer to actually be copied to the destination in the memory, you'd require a handle to the process. Acquire it using the OpenProcess API call along with PROCESS_VM_WRITE and PROCESS_VM_OPERATION access to the process.  

# Conclusion

That's it for today folks. I've brushed up on some of the most prominent API calls for memory and process listing. Practice these, and make sure you understand their use-cases for when we finally head to malware analysis. 