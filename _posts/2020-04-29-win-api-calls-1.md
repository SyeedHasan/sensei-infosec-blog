---
layout: post
title: "Windows API Calls: The Malware Edition"
date: 2020-04-29 11:00:00 +0500
categories: forensics windows api-calls
---

Windows API, in short, the WinAPI, is a set of functions and procedures, which can abstract much of the tasks you perform everyday on the Windows OS. The Application Programming Interface (API) calls exposes these functions to programmers to make use of procedures when writing one of your own isn't the most effective. 

Although the API calls are a bit hard to work with, they can still help you achieve much of what you'd like, without further coding. For example, you can call the API call 'RegSetValueA' in order to set the value of a registry key (which should be in a text format).

Although their legitimate use is great, the exact opposite is possible as well. Malware writers make use of these API calls to interact with the OS and perform nefarious tasks. 

With this article, I'll help you analyze a particular malware sample, along with the identification of a few API calls, and see if we can further identify the behavior of that particular malware sample. Let's get to it!

# Fetch the Sample

First, let's head over to the amazing malware repository (amongst several others) at [theZoo]{:target="_blank"}, managed by ytisf. 

I've downloaded the malware sample for Kovter, a malware that initially started off as a police ransomware and then evolved into a click-fraud malware. 

Let's analyze the sample and see if we can pick apart the Windows API calls which are utilized by this malware sample.

# Static Analysis: Kovter

I'll skip past the usuals we perform for static analysis. I'll run it through **PEStudio**, the PE header analyze, that's capable of much more. Fortunately, it also provides us the **libraries and imports** which are used by the file.

Let's open it up and see it in detail. The file name is "PDFXCView.exe" which appears to be a PDF Viewer of some sorts, but we'll know for sure once we get started. Here's an overview of the libraries which are imported by this executable:

![Overview from PEStudio](/assets/winapicalls/overview.png)

Here's a list of a few libraries which are imported by this executable:

| ws2_32.dll | Windows Sockets 2 - Holds networking functions, spawns network connections, and manages connections |
| wininet.dll | Internet Extensions - Access Internet resources by means of interacting with several protocols e.g. HTTP, FTP  |
| version.dll | Version Checking and File Installation - Returns version information for specified files | 
| kernel32.dll | Windows NT Base API - Holds core functionality e.g. manipulation of files, memory, and hardware |
| user32.dll | Multi-user Windows User API - Holds user interfaces components | 
| advapi32.dll | Advanced Windows32 Base API - Holds advanced Windows components with access to the registry, security calls, system state, and services | 
| shell32.dll | Windows Shell - Holds functions related to the Windows Shell |

Before you move forward with it, pick apart the libraries which are going to be useful in the analysis and could actually be utilized by a malware. Some libraries will often be imported for the successful operation of the program in Windows (often useless for the analysis).

# Identified API Calls and Imports

Each library has several calls being imported. The import count for this executable is 367. The blacklisted API count is 114, as per PEStudio. Going through all of these is going to take loads of time.

Fortunately, PEStudio groups the API calls with which you can identify the usage of the call itself. But before we start with them, let's go over the conventions these API calls follow.

## API Call Convention

Windows API Calls which are involved with text manipulation are often appended with **'A' or 'W'**. The 'A' is used to identify functions which work with ANSI strings as input(s) and output(s). The 'W' is used to identify functions which work with Unicode strings as input(s) and output(s). By the way, Windows also has native support for Unicode strings, because they're widely supported and work well with several languages and character sets.

Similarly, the **'Ex'** at the end of several API calls standards for 'Extended' - as in an extension of functionality along with a more detailed interface. For example, the import 'CreateFile' works with 4 parameters, whereas the 'CreateFileEx' call works with 17 parameters, which much more detail as you can imagine based on the increase in parameters count.

## Continuining with the Calls

Here's a list of the most intriguing API calls, which aren't most frequently found in Windows programs either:

### Registry API Calls

The following pair of API calls can be used to manage the registry, modify the values, and spawn other keys as well. This behavior is particularly useful to identify persistence via the registry. Other than that, almost every configuration and setting is stored in the Registry. You're likely to find the registry keys for services or other paths used by the malware if you monitor these calls.

| RegSetValueEx | Set the data-type and value of a certain registry key  |
| RegCreateKeyEx | Create a registry key  |
| RegEnumKeyEx | Extract the data-type and value of a specific registry key  |
| RegQueryValueEx | Get the data-type and value of a specific registry key |
| RegOpenKeyEx | Open a specific key |
| RegCloseKey | Close the open handle to the specified registry key |
| RegEnumValue | Enumerate through the values from a given registry key path |

### Threads and Processes

Monioring processes and threads for unknown activity is also important. Often, malware spawn processes and threads to carry out several tasks without being noticed. 

| CreateProcess | Create a new process along with threads |
| GetCurrentProcess | Retrieve a pseudo-handle for a process |
| GetCurrentProessId | Retrieve the key process identifier |
| CreateThread | Create a thread within a process |
| SetThreadPriority/GetThreadPriority | Set and retrieve the priority for the thread within the context of a process |
| GetProcessTimes | Retrieves the timing information about the process |
| ExitProcess | Exit a process and the threads it spawned |

### System Information

API calls can also extract system information, which is of much use to a malware. This information particularly identifies a system. The actor can even exfiltrate this data out by means of C2 communication.

| GetFileVersionInfo | Retrieves the information about a particular file  |
| GetFileVersionInfoSize | Checks whether retrieval of information is possible and returs size, if true |
| GetSystemMetrics | Returns the informationn about a particular metric or configuration. Several values are available to the user, along with the option to pass it as a parameter |
| GetSystemInfo/GetNativeSystemInfo | Retrieves system information |
| IsDebuggerPresent | Identifies a user-mode debugger on the excecutable |
| QueryPerformanceCounter | Get the current value of the performance monitor | 

### Files 

File-based API calls accumulate to a large count. But, just going through the list, I can pick a few apart which are extremely sketchy (especially for an executable that pretends to be a PDF viewer).

DeleteFile | Delete an existing file |
GetFileType | Retrieve the type of the file |
MoveFile | Move the file from one location to another |
GetFileAttributes | Retrieve the attributes of the specified file |
CopyFile | Copy the file from one location to another |
FindFirstFileEx | Search a file with information|
GetFileSize | Retrieves the size of the specified file |
ReadFile | Read data from the I/O device or a file  |

### Keyboard, Mouses, and Input Devices

Quite obviously, its API calls which are used to send input data to the processor and so on. These API calls are also used by malware (especially keyloggers) with the intent to steal data from a computer and dispatch it away. Here's a list of API calls found in this sample:

| EnableWindow | Ability to enable or disable keyboard or mouse input to the said Window |
| GetAsyncKeyState | Identify the state of the keys (pressed or released) |

### Cryptography

There's also evidence of some cryptographic API calls being loaded with the executable. These can indeed encrypt the data and hide it from the user's eyes before sending it off to the HTTP server (yes, we'll go over the networking API calls next). Here's a list of those API calls: 

| CryptDeriveKey | Generates cryptographic session keys |
| CryptEncrypt | Encrypts data | 
| CryptDecrypt | Decrypts data which is encrypted by CryptEncrypt |
| CryptCreateHash | Obtain handle to the hash object for the encryption function (passed off as a parameter). Also initiates the hashing of a stream of data |
| CryptHashData | Adds the data to a specified hash object |

### Networking

Lastly, the Windows Sockets library imports several functions and API calls which are key to spawn connections and used by the malware for C2 communication. 

| HttpQueryInfo | Retrieve headers related to HTTP requests |
| HttpSendRequestEx | Send request to a web server |
| HttpEndRequest | Break communication with a web server |
| HttpOpenRequest  | Spawn an HTTP request handle | 
| InternetConnectA | Open an FTP or HTTP session for a website |
| InternetGetConnectedState | Retrieves the connection state for the system |
| InternetSetOptionA | Change Internet options |
| InternetWriteFile | Write data to an Internet file whose handle has already been acquired |
| InternetCrackUrlA | Divide the URL into parts |
| InternetSetStatusCallbackA | Functions made during a state change or progress during an operation | 

# Our Hypothesis

Listing down API calls by groups wasn't very useful on its own. But, this can help us identify the behavior of the malware by checking out the calls it makes or could make if an opportunity arises. 

Starting from system versioning and file version information using **version.dll**. The API calls clearly suggest the program is capable of retrieving system information, which includes the information on files, the system itself, and whether a debugger is present or not. The last one is perhaps the most important. If the executable is under debugging in user-mode, the API call will yield true and the program can behave differently based on that. This is sketchy behavior! 

Secondly, it has much control over the files in the system - the ability to move, create, delete, copy? This clearly suggests the malware would be capable of copying itself elsewhere for evasion and persist itself. But that wouldn't be on itself would it? It also has access to the registry keys. Say just the **Run** keys are modified. Just that and the malware could achieve persistence.

Thirdly, the input modifying API calls are capable of collecting keystrokes and information about the key state. This behavior is particularly suggestive of a keylogger. The data could be collected and encrypted using the cryptography module. Then, it could be shipped off by means of POST requests to the actor's setup web server. This initiates the C2 server communication. 

# Conclusion

In conclusion, API calls speak much about the malware. They're a good source to start at if you're statically analyzing a malware without digging deep. Though in case of packed malware, this approach could require a little digging and that's for another lecture.

Point being, you should start off with important factors like these calls to identify what you're dealing with. Till next time, ciao! 

[TheZoo]: https://github.com/ytisf/theZoo
