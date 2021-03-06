---
layout: post
title: Debugging - I break things. Visual Studio Crash
tags:
- debugging
- visual studio
- stack overflow
---
Sometimes I abuse Visual Studio without trying. I use Visual Studio to create web performance test scripts and I usually use Fiddler to record the scripts. After recording a fairly trivial script I exported the Fiddler session and imported it into Visual Studio. Once I was done mucking with the Web Test layout I converted it to a coded web test. As soon as the coded test was done generating, Visual Studio crashed. So, I tried again, same thing. Nope, I don't like it---let's find out what's going on.

##TL;DR
You should see Visual Studio just disappear (crash) and not have anything in the `%APPDATA%\Microsoft\VisualStudio\<<Version>>\ActivityLog.xml.` In this case it was a stack overflow caused by too many lines for a string concatenation---this eventually lead to an access violation. You can correct this by reducing the number of lines below 1800. However, this amount of lines will slow down the UI considerably. You should really consider using a reference file.

## The Problem - Visual Studio
Visual Studio crashed. No errors, no warnings, no pop-up boxes. Nada. I looked around in a few places to see if anything was logged and I couldn't see a problem. This, of course, isn't expected behavior. As mentioned in the opening paragraph, this occurred when I converted my Fiddler recorded session into a coded web test.

I've done this same action thousands of times before. It's really not that involved and all it's doing is converting XML into code. I noticed that the code is indeed generated but VS crashes upon displaying the resulting code. This leads me to believe it's an issue with Visual Studio and not the way I recorded the tests or how Fiddler generated the code.

## The Analysis
I used WinDbg to open the devenv executable and see what was happening. Since the process was terminating I knew I'd see what was causing it. So I did my steps to reproduce the error and quickly found this:

~~~
(299c.1c08): Stack overflow - code c00000fd (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=1bbf93bc ebx=00000001 ecx=1bcef0fc edx=0000000d esi=1bcef0fc edi=20687454
eip=1b6ed1b7 esp=1bbf8f6c ebp=1bbf918c iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010206
cslangsvc!ExpressionBinder::BindExpr+0x1a:
1b6ed1b7 53              push    ebx
~~~

Yikes, it's not often you see one of these in a "fully productionalized" application (:grin:)... Anyway, looking at the call stack I could see something that was quite obvious. We're basically executing a bunch of these `cslangsvc!ExpressionBinder::BindExpr`. While I don't know exactly what this does, I can make a few educated guesses. Let's take a look at the stack to see what's up.

~~~
0:028> kb
 # ChildEBP RetAddr  Args to Child
00 1bbf918c 1b75cb4b 20687420 00000001 1d1155e1 cslangsvc!ExpressionBinder::BindExpr+0x1a
01 1bbf93cc 1b75cb4b 20687454 00000001 1d1157a1 cslangsvc!ExpressionBinder::BindExpr+0x1211
02 1bbf960c 1b75cb4b 20687488 00000001 1d115261 cslangsvc!ExpressionBinder::BindExpr+0x1211
03 1bbf984c 1b75cb4b 206874bc 00000001 1d115c21 cslangsvc!ExpressionBinder::BindExpr+0x1211
04 1bbf9a8c 1b75cb4b 206874f0 00000001 1d115ee1 cslangsvc!ExpressionBinder::BindExpr+0x1211
05 1bbf9ccc 1b75cb4b 20687524 00000001 1d1158a1 cslangsvc!ExpressionBinder::BindExpr+0x1211
<<< REMOVED FOR CLARITY >>
~~~

>The default number of frames shown is 20. To see more use kb 5000---or use the  .kframes command

~~~
0:028> kb 5000
 # ChildEBP RetAddr  Args to Child
00 1bbf918c 1b75cb4b 20687420 00000001 1d1155e1 cslangsvc!ExpressionBinder::BindExpr+0x1a
01 1bbf93cc 1b75cb4b 20687454 00000001 1d1157a1 cslangsvc!ExpressionBinder::BindExpr+0x1211
02 1bbf960c 1b75cb4b 20687488 00000001 1d115261 cslangsvc!ExpressionBinder::BindExpr+0x1211
03 1bbf984c 1b75cb4b 206874bc 00000001 1d115c21 cslangsvc!ExpressionBinder::BindExpr+0x1211
 <<< REMOVED About 1600 lines for clarity! >>>
6d2 1bceea0c 1b75cb4b 1fffda38 00000001 1d602e61 cslangsvc!ExpressionBinder::BindExpr+0x1211
6d3 1bceec4c 1b77cc01 1fffda6c 00000021 1d602821 cslangsvc!ExpressionBinder::BindExpr+0x1211
6d4 1bceee8c 1b6ed7e2 1fffda88 00000021 1d602ae1 cslangsvc!ExpressionBinder::BindExpr+0x14e3
6d5 1bcef0c8 1b6ee3b2 2066fb40 00000001 1d603501 cslangsvc!ExpressionBinder::BindExpr+0x645
6d6 1bcef140 1b75dcdd 2066fb40 1bcef15c 2001557c cslangsvc!StatementBinder::bindStatementWorker+0xdc
6d7 1bcef164 1b785f0b 20650010 00000000 1d603539 cslangsvc!StatementBinder::bindBlockInner+0x4d
6d8 1bcef1f8 1b786668 20650010 1d603649 206596d0 cslangsvc!StatementBinder::BindMethOrPropBody+0x1e6
6d9 1bcef304 1b7880f5 1bcef320 1d6036f1 00000001 cslangsvc!FUNCBRECCS::CompileMethod+0x6eb
6da 1bcef4bc 1b787d80 1bcef50c 1bcef5f4 00000001 cslangsvc!CIDEExpressionNodeBinder::BindMethod+0x187
6db 1bcef4fc 1b9d78fb 1bcef574 1f0ddd88 00000001 cslangsvc!CIDELanguageAnalysisEngine::BindMethod+0x8e
6dc 1bcef530 1b889039 1bcef574 1bcef5ec 1bcef5d4 cslangsvc!CIDELanguageAnalysisEngine::BindMethodForErrors+0x26
6dd 1bcef598 1babd87a 1bcef5fc 1bcef5ec 1bcef5d4 cslangsvc!CCompilerBinder::GetMethodErrors+0x3b
6de 1bcef634 1b884680 1f0c12b4 1fd3b644 1d603209 cslangsvc!__isa_available_init+0x830f5
6df 1bcef690 1b8843dc 1d60328d 76b91420 12cc5d10 cslangsvc!CSemanticBindRequest::ExecuteOneSlice+0xa8
6e0 1bcef6cc 1b69d871 1d603341 76b91420 12cc5d10 cslangsvc!CSemanticBindRequest::ExecuteWorker+0x118
6e1 1bcef700 1b69dbf4 1bcef734 1d603309 12cc5d9c cslangsvc!CBackgroundQueue::ExecuteRequestHelper+0x31
6e2 1bcef748 1b69db59 1bcef730 12cc5d10 4b651533 cslangsvc!CBackgroundQueue::ExecuteOneRequestWithoutAcquiringLock+0x1dd
6e3 1bcef768 1b69dbae 093f8a90 1d6033d5 12cc5d9c cslangsvc!CBackgroundQueue::ExecuteOneRequest+0x45
6e4 1bcef794 1b6e6a58 1d6033e1 00000000 00000000 cslangsvc!CBackgroundQueue::ExecuteRequests+0x3c
6e5 1bcef7cc 1b7b096b 1d6033b5 00000000 00000000 cslangsvc!CBackgroundQueue::ThreadEntry+0x8f
6e6 1bcef7f4 76b9336a 12cc5d10 1bcef840 770c92b2 cslangsvc!CBackgroundQueue::StaticThreadEntry+0x2b
6e7 1bcef800 770c92b2 12cc5d10 740deae1 00000000 kernel32!BaseThreadInitThunk+0xe
6e8 1bcef840 770c9285 1b7b0940 12cc5d10 00000000 ntdll!__RtlUserThreadStart+0x70
6e9 1bcef858 00000000 1b7b0940 12cc5d10 00000000 ntdll!_RtlUserThreadStart+0x1b
~~~

So, as I said before, we can makes some educated guesses and see what's going on here. It looks like when it's trying to compile the code and generate all of the tokens it pukes. Let's zoom in on frame 0x6d5. This is where we enter the method. We can see that it looks like a recursive function.

Let's inspect the stack to look for any clues as to what is going on. We should just grab a couple of frames worth of stack data. Since we're in a stack overflow we're possibly looking at 1MB of data to spin through. I'll pick 10 frames starting from 0x6d5 and stopping at 0x6df; use the ChildEBP column to get the addresses.

~~~
0:028> ?1bcef0c8-1bcedc8c
Evaluate expression: 5180 = 0000143c

0:028> dc 1bcedc8c 1bcef0c8
1bcedc8c  1bcedecc 1b75cb4b 1fffd900 00000001  ....K.u.........
1bcedc9c  1d6018e1 1fffd968 1bcef0fc 00000001  ..`.h...........
1bcedcac  20652928 180d8d00 00000000 00000200  ()e ............
1bcedcbc  1b75d077 1bcef0fc 00000004 0000000d  w.u.............
1bcedccc  180d8d00 2001f334 1b6f698c 50000163  ....4.. .io.c..P
1bcedcdc  00000000 2001f334 20652928 180d8d00  ....4.. ()e ....
1bcedcec  180d894c 1d6018b5 1bcedd18 1b773c2e  L.....`......<w.
1bcedcfc  2001f334 1fd66e38 1fd64db0 180d8d00  4.. 8n...M......
1bcedd0c  20652928 2001f334 180d8d00 1bcedd3c  ()e 4.. ....<...
1bcedd1c  1b772755 2001f334 1b772766 2001f334  U'w.4.. f'w.4..
1bcedd2c  1bcedd8c 20652928 180d8d00 1d60197d  ....()e ....}.`.
1bcedd3c  00000000 00000000 20652928 2001f334  ........()e 4..
1bcedd4c  1bcedd5e 1bcef0fc 1d601919 1bcef0fc  ^.........`.....
1bcedd5c  1ee82d30 20021114 1bceddc4 1ee82d30  0-..... ....0-..
1bcedd6c  180d8d01 1fd4e040 19746f60 1fd64db0  ....@...`ot..M..
1bcedd7c  2001f334 1bceddb4 1b72c0f7 200205fc  4.. ......r....
1bcedd8c  1bcef0fc 20668ef4 206e4ab0 20652928  ......f .Jn ()e
<<< NOTHING USEFUL IN HERE >>>
~~~

Woah! 5kb in just 10 frames! That is one hefty ass set of stack variables. Let's see what we have in that 5k. I only displayed the first couple hundred bytes to show the output of the command. When looking at the rest of the output it looks the same. While it doesn't look like much, I can see a lot of references to other places on the stack and out in the heap.

Let's use some pointer display commands to view what's in the heap out there. I'll start by using `dpu` this command will get the address pointed by the address on the stack and attempt to cast it to a set of unicode (wchar_t) strings.

~~~
0:028> dpu 1bcedc8c 1bcef0c8
<<< REMOVED FOR CLARITY >>>
1bcee99c  20706590 "華.華.2"
1bcee9a0  0000000b
1bcee9a4  1bcee9e8 "栰.....ﭠ.....ᚨ...贀...쭋...."
1bcee9a8  00000000
1bcee9ac  1bcee9c8 "죠.核...."
1bcee9b0  1b77c86a "䎉褄謞忆孞.쉝...অﬅ臿.í"
1bcee9b4  00000016
1bcee9b8  1bceea04 "..贀...쭋...."
1bcee9bc  1bf9f114 "__VIEWSTATE"
1bcee9c0  1bf9f114 "__VIEWSTATE"
1bcee9c4  1bceea04 "..贀...쭋...."
1bcee9c8  1b77c8e0 "쒃謌忇嵞..ၵ䒍ᠤ.ﾲ..ډ욋譞工.렀."
1bcee9cc  20706838 "__VIEWSTATE"
1bcee9d0  1bf9f114 "__VIEWSTATE"
1bcee9d4  00000016
1bcee9d8  2066fb60 "勰ኃ..."
1bcee9dc  1bceea7c "栰.("
1bcee9e0  1bceea08 "贀...쭋...."
1bcee9e4  1b77c8f8 ".ډ욋譞工.렀."
1bcee9e8  20706830 "."
1bcee9ec  1bf9f10c "."
1bcee9f0  1bf9f10c "."
<<< REMOVED FOR CLARITY >>>
~~~

I skimmed over a lot of the data to show you the only real readable references in there. **__VIEWSTATE**. Hmm, looks like it was possibly processing the VIEWSTATE in this coded webtest. I opened the output of the coded webtest in an external text editor and I could see a few places where I should start. The first one that caught my eye was this:


~~~Csharp
request22Body.FormPostParameters.Add("__VIEWSTATE",
"/wEPDwUJMzE5NDE5MjIzD2QWAgIDD2QWCAICDw8WAh4HVmlzaWJsZWhkZAIFDxQrAAUPFgQeFE1lbnVIa" +
"WdobGlnaHRUb3BNZW51Zx8AaGRkZBQrAAQQFgoeBkl0ZW1JRAURY3RsMDUtbWVudUl0ZW0wMDEeCEl0Z" +
"W1UZXh0BRBFdmVudCBNYW5hZ2VtZW50HhBNZW51SXRlbUNzc0NsYXNzBQdhcHBtZW51HhVJdGVtTW91c" +
"2VPdmVyQ3NzQ2xhc3MFDGFwcG1lbnVob3Zlch4OTWVudUl0ZW1IZWlnaHQbAAAAAAAACEABAAAAFCsAA" +
"hAWCh8CBSVjdGwwNS1tZW51SXRlbTAwMS1zdWJNZW51LW1lbnVJdGVtMDAwHwMFEU1hbmFnZSBFdmVud" +
"CBUeXBlHgdJdGVtVVJMBUsvRmNzdFRvb2xLaXRXZWIvRXZlbnRNYW5hZ2VtZW50L01hbmFnZUV2ZW50L" + "....";

// This goes on for about 4000 lines
~~~

**FOUR THOUSAND LINES!!** Knowing that I have such a large amount of strings that have to be concatenated/tokenized, and knowing that we had a stack overflow, *AND* knowing the VIEWSTATE is in the stack, I jumped to the conclusion that this was dependent on the amount of lines or the amount of data. Jump to the repro section for more info. **Spoiler alert, it doesn't matter about the amount of characters.**


## Why so much stack?

Now, about that 5k of data in just 10 frames. What's up with that? Here is some review for all of you who once knew about ABI and calling conventions...

When a new call is created the compiler does a few things so your application doesn't lose it's place when calling into another location. It does things such as storing the last stack location, setting the new stack location and setting up the stack space for it's method (the stack frame), and sometimes it will clear the address space. This is a general statement and there are other things that can and do go on inside of the frame setup.

To validate the function that is being called I used the RetAddr that is being repeated all over the stack trace `0x1b75cb4b`. If I unassemble the instructions before this return address I should see the `CALL` that invoked this method. You can use the `ub` command to do this.

Yep, we see a call into `cslangsvc!ExpressionBinder::BindExpr`. In order to see what was going on I unassembled the function like normal.

~~~
0:028> ub 1b75cb4b
cslangsvc!DefaultAttrBind::CompileEarly+0x6b:
1b75cb30 8bf8            mov     edi,eax
1b75cb32 85ff            test    edi,edi
1b75cb34 0f84b81dfeff    je      cslangsvc!DefaultAttrBind::CompileEarly+0x5c (1b73e8f2)
1b75cb3a e9ad211100      jmp     cslangsvc!DefaultAttrBind::CompileEarly+0x71 (1b86ecec)
1b75cb3f 6a01            push    1
1b75cb41 ff7714          push    dword ptr [edi+14h]
1b75cb44 8bce            mov     ecx,esi
1b75cb46 e85206f9ff      call    cslangsvc!ExpressionBinder::BindExpr (1b6ed19d)

0:028> u cslangsvc!ExpressionBinder::BindExpr
cslangsvc!ExpressionBinder::BindExpr:
1b6ed19d 55              push    ebp
1b6ed19e 8bec            mov     ebp,esp
1b6ed1a0 83e4f8          and     esp,0FFFFFFF8h
1b6ed1a3 6aff            push    0FFFFFFFFh
1b6ed1a5 68a816a91b      push    offset cslangsvc!__isa_available_init+0x5509e (1ba916a8)
1b6ed1aa 64a100000000    mov     eax,dword ptr fs:[00000000h]
1b6ed1b0 50              push    eax
1b6ed1b1 81ec10020000    sub     esp,210h
~~~

The assembly command `sub esp,210h` let's us know that the compiler has decided to reserve 528 bytes of space for this method---16bytes for what's pushed before the alloc, plus 512bytes. While that's not a lot of space, it certainly can add up since this is a (possible infinitley) recursive function.

So if we do the math, I have about 4000 lines of text to tokenize so it would take about 2MB of stack space. This won't work since the standard stack space is 1MB. At the top of my stack we can see my address is close to the stack limit.

~~~
0:028> !teb
TEB at fff02000
    ExceptionList:        1bbf93bc
    StackBase:            1bcf0000
    StackLimit:           1bbf1000
    SubSystemTib:         00000000
    FiberData:            00001e00
    ArbitraryUserPointer: 00000000
    Self:                 fff02000
    EnvironmentPointer:   00000000
    ClientId:             0000299c . 00001c08
    RpcHandle:            00000000
    Tls Storage:          1255b700
    PEB Address:          fffde000
    LastErrorValue:       0
    LastStatusValue:      c000007c
    Count Owned Locks:    0
    HardErrorMode:        0
~~~


But in reality, there's 32k left---we could squeeze another 64 lines out of that. So, why did we trigger a stack overflow when we have that much space left? The short answer is Windows loves you and does not want to let you fail. If you look at the amount of stack space that is allocated we see  there is a total of 1,044,480 (0xFF000), which is exactly 255 4kb (0x1000) pages.

As your stack grows you will cross into a new page boundary and the OS will need to allocate that space for you. In this instance you're pretty much at the limit so Windows tells you, "Hey, we've reached a reasonable amount of pages used, you should stop."

If you don't heed this warning, the stack overflow exception won't stop your execution (unless you have an exception handler which this does not). Instead it will allow it to continue on until you have a hard fault. If you were to let this application continue you will will eventually get an access violation.

~~~
0:028> g
(299c.1c08): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=1bbf137c ebx=00000001 ecx=1bcef0fc edx=0000000d esi=1bcef0fc edi=20686898
eip=1b6ed1b7 esp=1bbf0f2c ebp=1bbf114c iopl=0         nv up ei pl nz na po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010202
cslangsvc!ExpressionBinder::BindExpr+0x1a:
1b6ed1b7 53              push    ebx
~~~

The instruction that makes it fail is `push ebx` it fails because it's trying to access `1bbf0f2c`. If we inspect the memory here we will see invalid marks. So the unhandled access violation is the problem here.

~~~
0:028> dp @esp L 1
1bbf0f2c  ????????
~~~

## Side Note About Symbols
*Wait! I noticed you used the raw addresses a lot, why didn't you just disassemble the symbol `cslangsvc!ExpressionBinder::BindExpr+0x1211`?*  Well, the symbol is resolved to the best of WinDbg's ability. If you use this symbol across different functions you will see different behavior. As a matter of fact if you try to unassemble it you get some invalid assembly.

~~~
0:028> ? cslangsvc!ExpressionBinder::BindExpr+0x1211
Evaluate expression: 460252078 = 1b6ee3ae

0:028> u cslangsvc!ExpressionBinder::BindExpr+0x1211
cslangsvc!StatementBinder::bindStatementWorker+0xd8:
1b6ee3ae ebed            jmp     cslangsvc!StatementBinder::bindStatementWorker+0xc7 (1b6ee39d)
1b6ee3b0 ff              ???
1b6ee3b1 ff8bf8c645fc    dec     dword ptr [ebx-3BA3908h]
1b6ee3b7 018b4b046a01    add     dword ptr [ebx+16A044Bh],ecx
1b6ee3bd 6a00            push    0
1b6ee3bf 6a00            push    0
1b6ee3c1 6a01            push    1
1b6ee3c3 83c114          add     ecx,14h

0:028> ln cslangsvc!ExpressionBinder::BindExpr+0x1211
(1b6ee2d6)   cslangsvc!StatementBinder::bindStatementWorker+0xd8   |  (1b6ee42d)   cslangsvc!StatementBinder::BindStatemet

0:028> ln 1b75cb4b
(1b6ed19d)   cslangsvc!ExpressionBinder::BindExpr+0x1211   |  (1b6edf47)   cslangsvc!BindingContext::BindingContext
~~~

## Reproduction and Solutions

The simple fix was to remove the line breaks using an external editor. However, there is no real solution that can be applied with out changing the code in Visual Studio or by changing the way the code is generated, again by a Microsoft product.

If you want to see this behavior for yourself. Create a blank console application and create a bunch of string concats. Notice that the problem isn't dependent on how much data is inside of your methods.

~~~Csharp
using System;

namespace WillBreakIDE
{
  class Program
  {
    string willbreakide = " " +
                          " " +
                          " " +
                          " " +
                          // Repeat this patern until you have about 2000 lines.
                          " ";
    static void Main(string[] args)
    {
    }
  }
}
~~~
j
