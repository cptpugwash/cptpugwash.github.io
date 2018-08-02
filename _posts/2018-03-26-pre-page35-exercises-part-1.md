---
layout: post
title:  "Practical reverse engineering page35 exercises part 1"
date:   2018-03-26
summary: Practical reverse engineering page35 exercises part 1
---

**1.** Repeat the walk through by yourself. Draw the stack layout, including parameters and local variables.

| Stack 1     |
| ----------- |
| EDI         |           |
| …           | [EBP-130] |
| IDT         | [EBP-8]   |
| IDT base    | [EBP-6]   |
| Old ebp     | [EBP]     |
| Return addr | [EBP+4]   |

In the first frame it subs 130h from esp creating local variable space on the stack then sidt writes the IDT to ebp-8. Ebp-6 is then moved into eax and compared, if bellow or equal to 8003F400h jump and if not bellow 80047400h jump. If neither jump is taken it will clean up the stack and return. 

Note 80047400h - 8003F400h is the base of IDT for windows xp, looks like it is testing if the machine is win xp.

| Stack 2     |
| ----------- |
| 2           |           |
| 0           |           |
| EDI         |           |
| Init memory | [EBP-300] |
| 0           | [EBP-130] |
| IDT         | [EBP-8]   |
| IDT base    | [EBP-6]   |
| Old ebp     | [EBP]     |
| Return addr | [EBP+4]   |

Eax is zeroed and 49h moved into ecx, then [ebp-12ch] moved into edi and 0 into [ebp-130]. 0 and 2 are pushed onto the stack and rep stosd is executed. This will copy 0 into the location pointed to by edi 49h times, effectively zeroing out memory. Then createtoolhelp32snapshot(2, 0) is called.

From the MSDN documentation it looks like this function will take a snapshot of all the processes, used to enumerate them, returns a handle to the snapshot. The return value is then moved into edi and compared to -1. 

| Stack 3                  |
| ------------------------ |
| EDI (handle to snapshot) |           |
| EAX (EBP-130)            |           |
| ESI (1)                  |           |
| EDI                      |           |
| Struct                   | [EBP-300] |
| 128h                     | [EBP-130] |
| IDT                      | [EBP-8]   |
| IDT base                 | [EBP-6]   |
| Old ebp                  | [EBP]     |
| Return addr              | [EBP+4]   |

ebp-130 is then moved into eax and esi (1), eax (ebp-130) and edi (handle) are pushed to the stack and 128h is moved into ebp-130. Then process32first(handle, pointer to processentry32 struct) is called. This will return true if the first entry has been copied into the buffer (ebp-300), the result is tested if not zero it will continue. If it is it will jump to other code.

| Stack 4               |           || Stack 5               |
| --------------------- |           || --------------------- |
| Offset "explorer.exe" |           ||                       |
| ECX (string)          |           ||                       |
| ESI (1)               |           || ESI (1)               |           |
| EDI                   |           || EDI                   |           |
| Struct                | [EBP-300] || Struct                | [EBP-300] |
| 128h                  | [EBP-130] || 128h                  | [EBP-130] |
| IDT                   | [EBP-8]   || IDT                   | [EBP-8]   |
| IDT base              | [EBP-6]   || IDT base              | [EBP-6]   |
| Old ebp               | [EBP]     || Old ebp               | [EBP]     |
| Return addr           | [EBP+4]   || Return addr           | [EBP+4]   |

Note that in stdcall the stack is cleaned up by the callee and the 2 parameters for the previous function have been removed. _stricmp is then moved into edi, cdecl calling convention is used here as the caller cleans the stack. Ebp-268 is then loaded into ecx and an offset string and ecx are pushed onto the stack before calling esi _stricmp("explorer.exe", ecx). The stack is then cleaned up and the result tested. If its zero (a match) then it jumps to other code else it continues.

| Stack 6               |           || Stack 7               |
| --------------------- |           || --------------------- |
| EDI (handle)          |           || Offset "explorer.exe" |           |
| EDX (EBP-130)         |           || EAX (string)          |           |
| ESI (1)               |           || ESI (1)               |           |
| EDI                   |           || EDI                   |           |
| Struct                | [EBP-300] || Struct                | [EBP-300] |
| 128h                  | [EBP-130] || 128h                  | [EBP-130] |
| IDT                   | [EBP-8]   || IDT                   | [EBP-8]   |
| IDT base              | [EBP-6]   || IDT base              | [EBP-6]   |
| Old ebp               | [EBP]     || Old ebp               | [EBP]     |
| Return addr           | [EBP+4]   || Return addr           | [EBP+4]   |

It loads ebp-130 into edx and edx and edi are pushed onto the stack before calling process32next(handle, struct). This gets the next process item and the result is tested. If it gets an item then it enters a loop that executes stricmp("explorer.exe", string), if it gets a match it jumps to other code else loop back and the next process item is got, process32next(handle, struct). If that doesn’t get an item it will jump to other code. This will continue until the string matches "explorer.exe" or runs out of items.

| Stack 8     |           || Stack 9     |
| ----------- |           || ----------- |
| ESI (1)     |           ||             |
| EDI         |           || EDI         |           |
| Struct      | [EBP-300] || Struct      | [EBP-300] |
| 128h        | [EBP-130] || 128h        | [EBP-130] |
| IDT         | [EBP-8]   || IDT         | [EBP-8]   |
| IDT base    | [EBP-6]   || IDT base    | [EBP-6]   |
| Old ebp     | [EBP]     || Old ebp     | [EBP]     |
| Return addr | [EBP+4]   || Return addr | [EBP+4]   |

If the strings match, ebp-280 is moved into eax and ebp-296 into ecx, these are then compared. If they don’t match it returns else it continues. If it runs out of process items it moves ebp+fdwreason into eax/ecx, compares and returns.

| Stack 10       |
| -------------- |
| 0              |           |
| 0              |           |
| Offset address |           |
| 0              |           |
| 0              |           |
| 0              |           |
| EDI            |           |
| Struct         | [EBP-300] |
| 128h           | [EBP-130] |
| IDT            | [EBP-8]   |
| IDT base       | [EBP-6]   |
| Old ebp        | [EBP]     |
| Return addr    | [EBP+4]   |

Note that fdwreason indicates why the dll entry point was called, there are values in the documentation (1, 0, 2 and 3). Fdwreason is then moved into eax and decremented by 1 (note that if 0 it will be FFFFFFFF). If the result is not zero then 1 is moved into eax and returns. Else it creates a thread createthread(0, 0, offset addr, 0, 0 ,0). Once created it moves 1 into eax, cleans up the stack and returns.

**2.** In the example walk through, we did a nearly one to one translation of the assembly code to c. as an exercise re-decompile this whole function so that it looks more natural. what can you say about the developers skill level/experience? explain your reasons. Can you do a better job?

<figure class="lineno-container">
{% highlight c linenos=table %}
/* pseudo c for sample j dll main */

typedef struct _IDTR(){
  DWORD base;
  SHORT limit;
} IDTR, *PIDTR;

BOOL __stdcall DLLmain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)

{
  IDTR idtr;
  __sidt(&idtr);
  if (idtr.base > 0x8003F400 && idtr.base < 0x80047400){
    return FALSE;
  }

  // zero out memory
  PROCESSENTRY32 pe;
  memset(&pe, 0, 0x49);

  hsnapshot = CreateToolhelp32Snapshot(2, 0);
  if (!hsnapshot){
    return FALSE;
  }

  pe.dwSize = sizeof(PROCESSENTRY32);

  if (Process32First(hsnapshot, &pe) != FALSE){
      do {
        if (_stricmp(pe.szExeFile, "explorer.exe") == 0){
          if (pe.th32ParentProcessID == pe.th32ProcessID){
            break;
          }
        }
      } while (Process32Next(hsnapshot, &pe) != FALSE);
      if (fdwReason == 0){
        return FALSE;
      }
  } else if (fdwReason == 0){
    return FALSE;
  }

  if (fdwReason == 0 || fdwReason == 1){
    return TRUE;
  } else {
    CreateThread(0,0,0, 0x100032d0, 0,0)
    return TRUE;
  }
}
{% endhighlight %}
</figure>

The developer uses a method of testing the base address of the IDT to check if its win xp. This is hardcoded and easily spotted by an analyst. Also its not full proof as the address can change if running on another core that isnt core 0. The code also lacks some error checking for memset and createthread and not closing the snapshot handle. Coming from a Python background not having done much C coding the logic seams a bit odd eg the main do while loop and the checking of fdwreason.

**3.** In some of the assembly listings, the function name has a @ prefix followed by a number. explain when and why this decoration exists.

Both stdcall and fastcall use the @ name decoration or prefix followed by a number (@n or @12). This is used to represent the number of bytes passed to the function, note that the return instruction then has an optional argument of the same number of bytes to pop off the stack when it returns. The number will always be stack aligned eg 4, 8, 12 etc.

_DllMain@12   // will call dll main passing 12 bytes  
retn 0ch      // will pop off 12 bytes when returning to clean up the stack