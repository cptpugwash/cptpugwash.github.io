---
layout: post
title:  "Practical reverse engineering page35 exercises part 2"
date:   2018-04-02
summary: Practical reverse engineering page35 exercises part 2
---

**4.** Implement the following functions in x86 assembly: strlen, strchr, memcpy, memset, strcmp, strset.

**strlen** - is straight forward, we just use repne scasb with eax set to the null byte, it will then scan bytes in edi until it matches, counting is done with ecx

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db  'Hello world', 0

section .text
global CMAIN
CMAIN:
    mov ebp, esp    ; for correct debugging
    push string     ; passing string as arg
    call strlen     ; strlen call
    add esp, 4      ; stack cleanup
    ret
    
; gets the length of a string passed
strlen:
    push ebp            ; push old ebp to stack to save it
    mov ebp, esp        ; new stack frame
    mov edi, [ebp+8]    ; put pointer to string in edi
    xor eax, eax        ; zero eax
    or ecx, 0FFFFFFFFh  ; -1 in ecx
    repne scasb         ; scan string bytes, scan for eax in edi
    not ecx             ; negate to positive number
    dec ecx             ; scas counts null byte, decrement it by 1
    mov eax, ecx        ; move result into return register
    mov esp, ebp        ; epilogue
    pop ebp             ; pop ebp to restore stack frame
    ret                 ; return
{% endhighlight %}
</figure>

**strchr** - can be done with a simple loop using 3 registers or with scasb, either way we are reading a byte and comparing it to another byte, if that byte matches we return else if that byte matches 0 we exit

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db  'Hello world', 0
char: db 'w', 0

section .text
global CMAIN
CMAIN:
    mov ebp, esp    ; for correct debugging
    push char       ; push char as second arg+c
    push string     ; push as first arg+8
    call strchr     ; call function strchr(string, char)
    sub esp, 8
    mov esp, ebp
    pop ebp
    ret

; get pointer to first occurence of char in string
strchr:
    push ebp
    mov ebp, esp
    mov eax, [ebp+8]    ; string
    mov ecx, [ebp+12]   ; char
    mov cl, [ecx]       ; dereference to get char byte

    sloop:
    mov dl, [eax]   ; get first byte of string
    cmp cl, dl      ; compare first byte to char arg
    je returnzero   ; if equal return, eax will already be set to pointer
    inc eax         ; if not inc pointer to next string
    cmp dl, 0h      ; check if string char is null byte
    jnz sloop       ; if not start loop
    xor eax, eax    ; else end of string, char not found return zero

    returnzero:
    mov esp, ebp
    pop ebp
    ret
{% endhighlight %}
</figure>

**memcpy** - uses movsb which uses esi, edi and ecx which is perfect for copying data from src to dst and ecx as the count

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db  'Hello world', 0
buffer: times 11 db 'A'

section .text
global CMAIN
CMAIN:
    mov ebp, esp    ; for correct debugging
    push 11         ; byte count
    push string     ; src
    push buffer     ; dst
    call memcpy     ; call memcpy(dst, src, n bytes)
    sub esp, 16
    mov esp, ebp
    pop ebp
    ret
    
memcpy:
    push ebp
    mov ebp, esp
    mov edi, [ebp+8]    ; dst
    mov esi, [ebp+12]   ; src
    mov ecx, [ebp+16]   ; byte count
    mov eax, edi        ; put dst location in eax for return
    rep movsb           ; copy single byte from esi to edi, ecx times
    mov esp, ebp
    pop ebp
    ret
{% endhighlight %}
</figure>

**memset** - very similar to memcpy but uses stosb, stores a byte from eax into edi, ecx times

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db  'Hello world', 0
buffer: times 11 db 0

section .text
global CMAIN
CMAIN:
    mov ebp, esp    ; for correct debugging
    push 11         ; count
    push 'A'        ; char
    push buffer     ; dst
    call memset     ; call memset(dst, char, count)
    sub esp, 16
    mov esp, ebp
    pop ebp
    ret
    
memset:
    push ebp
    mov ebp, esp
    mov edi, [ebp+8]  ; dst
    mov eax, [ebp+12] ; char
    mov ecx, [ebp+16] ; count
    push edi          ; push pointer to dst
    rep stosb         ; store byte from eax in edi, ecx times
    pop eax           ; pop pointer to dst into return
    mov esp, ebp
    pop ebp
    ret
{% endhighlight %}
</figure>

**strcmp** - can be implemented with a similar loop as strchr to compare each byte in a loop, however this method will return 0 (true) if the first string ends and the second string is only a partial match ie hasn't ended. Can also be implemented with rep cmpsb but requires the string lengths to be known beforehand.

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db  'Hello', 0
string2: db 'Hello', 0

section .text
global CMAIN
CMAIN:
    mov ebp, esp ; for correct debugging
    push string2 ; str 2
    push string  ; str 1
    call strcmp  ; call strcmp(str1, str2)
    sub esp, 12
    mov esp, ebp
    pop ebp
    ret
    
strcmp:
    push ebp
    mov ebp, esp
    mov esi, [ebp+8]  ; str 1
    mov edi, [ebp+12] ; str 2
    xor eax, eax
    
    sloop:
    ; get bytes
    mov al, [esi]
    mov bl, [edi]
    cmp al, bl    ; check they match
    jne exit      ; if they dont match exit, eax will hold a value
    
    cmp bl, 0h    ; checking for null byte
    je exit       ; eax will hold null byte if end of char as well
    
    ; increment pointer to strings, and loop again
    inc edi
    inc esi
    jmp sloop
    
    ; eax will contain null byte if it matches
    exit:         ; else it will contain value of string
    mov esp, ebp
    pop ebp
    ret
{% endhighlight %}
</figure>

**strset** - again we can use a call to strlen and use rep stosb to replace all the bytes or use another simple loop replacing chars till null byte

<figure class="lineno-container">
{% highlight c linenos=table %}
%include "io.inc"
section .data
string: db "Hello world", 0

section .text
global CMAIN
CMAIN:
    mov ebp, esp    ; for correct debugging
    push 'A'        ; char
    push string     ; string
    call strset     ; calls strset(str, char)
    sub esp, 12
    mov esp, ebp
    pop ebp
    ret

; set all chars in string to char passed, return pointer to string
strset:
    push ebp
    mov ebp, esp
    mov edi, [ebp+8]  ; string
    mov eax, [ebp+12] ; char
    push edi          ; save pointer to string
    
    sloop:
    mov cl, [edi]     ; get byte
    cmp cl, 0h        ; check if null byte
    je exit
    mov [edi], al
    inc edi
    jmp sloop
    
    exit:
    pop eax           ; get pointer to string
    mov esp, ebp
    pop ebp
    ret
{% endhighlight %}
</figure>