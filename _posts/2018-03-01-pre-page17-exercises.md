---
layout: post
title:  "Practical reverse engineering page17 exercises"
date:   2018-03-01
summary: Practical reverse engineering page17 exercises
---

**1.** Given what you learned about call and ret, explain how you would read the value of eip? why can you just do mov eax, eip?

There are no instructions that directly affect eip. However when making a call the instruction address after the call is pushed to the stack, when retn is executed it will be poped off the stack into eip. While in the function call we could read the stack eg ebp+4 will be the retn addr put into eip. 

**2.** Come up with at least two code sequences to set eip to 0xAABBCCDD.
{% highlight c %}
jmp 0xAABBCCDD

push 0xAABBCCDD
retn

call 0xAABBCCDD
{% endhighlight %}

**3.** In the example function, addme, what would happen if the stack pointer were not properly restored before executing ret?

retn would pop off the wrong value and eip would be set to somewhere else corrupting execution flow.

**4.** In all the calling conventions explained the return value is stored in eax. What happens when the return value does not fit in a 32bit register? Write a program to experiment and evaluate your answer. Does the mechanism change from compiler to compiler?

I usually program in Python and don't use C so I'm not entirely sure I know how to answer this correctly but will give it a go. I would imagine that it would return a pointer to the value rather than the value itself, or use multiple registers like `mul` does.

The main problem I had was actually getting a value larger than 32bit into the register, I had to look at the C data types and long long seemed to fit best as its a 64bit value. However I did have a few issues when experimenting and compiling `gcc: warning that the constant is too large for its type`.

Using the following C code and data type long long then disassembling shows that it puts the values into eax and edx then returns, it then moves these values onto the stack as a local variable.

[Link to godbolt for code](https://godbolt.org/g/4WjzvK)

<figure class="lineno-container">
{% highlight c linenos=table %}
long long testfunc(long long z){
  return z+z;
}

int main(){
  long long c;
  long long z = 2147483647;
  c = testfunc(z);
}
{% endhighlight %}
</figure>

<figure class="lineno-container">
{% highlight c linenos=table %}
testfunc(long long):
  ...snip
  mov     eax, DWORD PTR [ebp+8]
  mov     DWORD PTR [ebp-8], eax
  mov     eax, DWORD PTR [ebp+12]
  mov     DWORD PTR [ebp-4], eax
  mov     eax, DWORD PTR [ebp-8]
  mov     edx, DWORD PTR [ebp-4]
  add     eax, eax
  adc     edx, edx
  ...snip

main:
  ...snip
  mov     DWORD PTR [ebp-16], 2147483647
  mov     DWORD PTR [ebp-12], 0
  push    DWORD PTR [ebp-12]
  push    DWORD PTR [ebp-16]
  call    testfunc(long long)
  add     esp, 8
  mov     DWORD PTR [ebp-24], eax
  mov     DWORD PTR [ebp-20], edx
  ...snip
{% endhighlight %}
</figure>

[Godbolt](https://godbolt.org/) is an online compiler explorer were we can type in code and see what it disassembles to, there is also a drop down box to select different compilers. Testing this code with multiple compilers (gcc 5.3, clang 4.0 and msvc) produces the same results in that they all use two registers eax:edx.