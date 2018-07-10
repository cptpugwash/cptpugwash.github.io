---
layout: post
title:  "Practical reverse engineering page11 exercises"
date:   2018-02-26
summary: Practical reverse engineering page11 exercises
---

1. Function uses a combination scas and stos to do its work. First explain what is the type of the [ebp+8] and [ebp+c] in line 1 and 8. Then explain what the code snippet does.

ebp+8 and ebp+c are both arguments on the stack passed to this function, the use of scasb suggests they are strings. ebp+c is moved into al so is only 1 byte long which means its 1 char.

<figure class="lineno-container">
{% highlight c linenos=table %}
8B 7D 08    mov edi, [ebp+8]		; moves string into edi
8B D7       mov edx, edi		; saves string in edx
33 C0       xor eax, eax		; xoring a reg with its self will null it out eg make it 0
83 C9 FF    or ecx, 0FFFFFFFFh		; makes ecx -1, ecx used as counter for repne
F2 AE       repne scasb			; scans edi and compares it to eax which is set to 0
83 C1 02    add ecx, 2			; adds 2 to ecx
F7 D9       neg ecx			; twos complement of ecx, makes it a positive number
8A 45 0C    mov al, [ebp+0Ch]		; moves char into al
8B AA       mov edi, edx		; moves the string back into edi
F3 AA       rep stosb			; stores the value of al into edi ecx times
8B C2       mov eax, edx		; move the string into eax
{% endhighlight %}
</figure>

The snippet scans a string till it finds 0 (null byte/end of string). Then copies a char into the string ecx times. Basically it searches the string [ebp+8] for the end of the string (null byte) while counting using ecx. Then it copies the second argument (a single char) over the first string passed, effectively overwriting it with the single char passed. Essentially counting chars in string, then overwriting with a char.

To test this out the snippet needs some extra code. Two arguments need to be passed, a string and a char to overwrite it with. This code can now be assembled, debugged and stepped through to see what's actually happening.

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string db  'Hello world', 0
char    db  'A',0

section .text
global CMAIN
CMAIN:
    mov ebp, esp; for correct debugging
    push string
    push char
    call testfunc
    add esp, 8
    ret
    
testfunc:
    mov ebp, esp
    mov edi, [ebp+8]		
    mov edx, edi			
    xor eax, eax			
    or ecx, 0FFFFFFFFh		
    repne scasb				
    add ecx, 2				
    neg ecx					
    mov al, [ebp+0Ch]		
    mov edi, edx			
    rep stosb				
    mov eax, edx
    ret	
{% endhighlight %}
</figure>