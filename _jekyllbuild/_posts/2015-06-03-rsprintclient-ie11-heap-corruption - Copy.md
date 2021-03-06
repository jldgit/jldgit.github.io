---
layout: post
title: Debugging - SSRS Report Viewer and IE11
tags:
- debugging
- ssrs
- ie11
---
Recently we upgraded IE in order to keep up with the EOL of Internet Explorer---for most of our enterprise that means we're going to IE11. This started a while ago and it took some time for it to trickle down to a majority of the machines. In fact, some standard user machines were upgraded before the developer's machines were. This lead to a number of broken pages and functionality. Aside from broken HTML pages was the SSRS Report Viewer control---after the upgrade it started crashing IE. You may have run into something like this after your upgrades and you don't have a lot of options. Be prepared to read the words unsupported and not recommended ... ***A LOT***.

##TL;DR
So, here is a disclaimer that you should read and take seriously. The patching I'm providing is not meant to be a long term fix. Hell, it might not even be something you'd even consider. So, in short I'd stay away from them unless you don't have much of a choice. Also, to keep errant versions of the DLL from getting out in the wild, I will NOT provide the code. You will need to patch it yourself.

###Not Recommended
1. Patch the RSClientPrint.dll
 - There is a 7 byte patch that will jump over the offending code. But, this would not be supported by Microsoft and could cause undefined behavior. [Jump to the patch section](#the-patch).

###Recommended
1. Upgrade to the latest and greatest version of the Report Viewer control
2. Write your own (if you can)

## The Problem - IE11 crashes when printing a report
The crash would happen when attempting to print from inside IE11 using the print button on the report viewer control. The version information is below. When the crash would occur it there was no message, no warning, just a hard crash of the IE process. Depending on your organizations policies or even some old lurking fixes and workarounds, you may only be executing your browser sessions in one process ([TabProcGrowth][tpgrow]) if this was the case everything died and you could lose a lot of work progress.

In our case, reporting is part of a critical work flow and it definitely has an impact to the end users.

~~~
Name:                   RSClientPrint 2005 Class
Publisher:              Microsoft Corporation
Type:                   ActiveX Control
Architecture:           32-bit
Version:                9.00.4053.00
File date:              ‎Wednesday, ‎May ‎27, ‎2009, ‏‎3:26 AM
Date last accessed:     ‎Today, ‎June ‎02, ‎2015, ‏‎1 hour ago
Class ID:               {0D221D00-A6ED-477C-8A91-41F3B660A832}
Use count:              26
Block count:            0
File:                   RSClientPrint.dll
Folder:                 C:\Windows\Downloaded Program Files
~~~

>Before we start down the path of who's at fault, it should be noted that this version is unsupported by MS in our configuration. The lowest supported version is 2008 CU1 for IE11.

## The Analysis
In the wild, we were getting reports of IE crashing randomly. It was crashing more times than it was not, but there were instances where the user was able to complete the workflow, which seems odd. If it crashes once, it should crash all of the time. Turns out it was during one specific action. Trying to print reports.

On the workstations we set up an ADPlus script to capture the unhandled exceptions. There was one exception in particular that was causing the crash. Let's look at the stack trace below.

~~~
0:020> k
 # ChildEBP RetAddr  
00 0ae096e8 76eaf96a RSClientPrint!CPrintDlg::PrintHookProc+0xb7
01 0ae09710 76eaf64f comdlg32!PrintInitPrintDlg+0x422
02 0ae0980c 7708c4e7 comdlg32!PrintDlgProc+0xc57
03 0ae09838 770a5855 user32!InternalCallWinProc+0x23
04 0ae098b4 770a59f3 user32!UserCallDlgProcCheckWow+0xd6
05 0ae098fc 770a5be3 user32!DefDlgProcWorker+0xa8
06 0ae09918 7708c4e7 user32!DefDlgProcW+0x22
07 0ae09944 7708c5e7 user32!InternalCallWinProc+0x23
08 0ae099bc 77085294 user32!UserCallWinProcCheckWow+0x14b
09 0ae099fc 770a4f6c user32!SendMessageWorker+0x4d0
0a 0ae09ab8 770a3b16 user32!InternalCreateDialog+0xb0d
0b 0ae09ae8 770a3b76 user32!InternalDialogBox+0xa7
0c 0ae09b08 76eaec0f user32!DialogBoxIndirectParamAorW+0x37
0d 0ae09b3c 76eaf13e comdlg32!PrintDisplayPrintDlg+0xec
0e 0ae09f90 76eae95d comdlg32!PrintDlgX+0x575
0f 0ae0a448 63963a8b comdlg32!PrintDlgA+0x5e
10 0ae0a464 0cd87d04 IEShims!NS_UISuppression::APIHook_PrintDlgA+0x7d
11 0ae0a4f0 0cd87dab RSClientPrint!CPrintDlg::Print+0x102
12 0ae0a598 0cd7ee73 RSClientPrint!CPrintDlg::PrintDlgA+0x4d
13 0ae0b27c 774e3e75 RSClientPrint!CRSClientPrint::Print+0x6b
~~~

Pretty much it says that the RSClientPrint application is faulting at this instruction. And as it would be the context in this stack indicates it's when we open the print dialog. We had two dumps that looked the exact same so I'm certain it's going to occur here. As it would be, nothing has changed from the application perspective or the machine, so it's definitely IE.

Let's look at the instruction that caused the fault.

~~~
0:020> r
eax=0000006c ebx=0bf66770 ecx=00005511 edx=773770f4 esi=0bf67300 edi=0ae09fa4
eip=0cd873ee esp=0ae096e0 ebp=0ae096e8 iopl=0         nv up ei pl nz na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010206
RSClientPrint!CPrintDlg::PrintHookProc+0xb7:
0cd873ee 8b4804          mov     ecx,dword ptr [eax+4] ds:0023:00000070=????????
~~~

So, we're definitely trying to access some invalid memory, address 0x70 is certainly not something we can get to directly in user mode. And something we shouldn't be fiddling with in kernel mode either.

If we look back up the instruction stack we can see that the EAX register is populated with ESI+0x44. That instruction is at 0x0cd873e7. Looks simple enough, but what is so important that it needs to get this data?

~~~
0:020> ub @eip
RSClientPrint!CPrintDlg::PrintHookProc+0xa1:
0cd873d8 83664400        and     dword ptr [esi+44h],0
0cd873dc 59              pop     ecx
0cd873dd eb48            jmp     RSClientPrint!CPrintDlg::PrintHookProc+0xf0 (0cd87427)
0cd873df ff7508          push    dword ptr [ebp+8]
0cd873e2 e83bf5ffff      call    RSClientPrint!CPrintDlg::SetDialogResources (0cd86922)
0cd873e7 8b4644          mov     eax,dword ptr [esi+44h]
0cd873ea 85c0            test    eax,eax
0cd873ec 741e            je      RSClientPrint!CPrintDlg::PrintHookProc+0xd5 (0cd8740c)
~~~

~~~
0:020> u @eip L 10
RSClientPrint!CPrintDlg::PrintHookProc+0xb7:
0cd873ee 8b4804          mov     ecx,dword ptr [eax+4]
0cd873f1 8b780c          mov     edi,dword ptr [eax+0Ch]
0cd873f4 8b10            mov     edx,dword ptr [eax]
0cd873f6 8b4008          mov     eax,dword ptr [eax+8]
0cd873f9 6a01            push    1
0cd873fb 2bf9            sub     edi,ecx
0cd873fd 57              push    edi
0cd873fe 2bc2            sub     eax,edx
0cd87400 50              push    eax
0cd87401 51              push    ecx
0cd87402 52              push    edx
0cd87403 ff7508          push    dword ptr [ebp+8]
0cd87406 ff157c13d70c    call    dword ptr [RSClientPrint!_imp__MoveWindow (0cd7137c)]
<<< REMOVED >>>
~~~

Ah, so it's calling `MoveWindow()` and setting the window position of the dialog and forcing a redraw. Look at the instruction at 0x0cd873ea in the first disassembly---`test eax,eax`. This is checking for 0 (or nullptr) at that memory address. If it's not 0 it will continue down the execution path. So, if we were to put a little thought into the entire disassembly of the `RSClientPrint!CPrintDlg::PrintHookProc` code, it might look like this:

~~~Cpp
INT_PTR CALLBACK DialogProc(
HWND   hwndDlg,
UINT   uMsg,
WPARAM wParam,
LPARAM lParam
)
{
  if (uMsg == WM_INITDIALOG)
  {
    SetDialogResources(hwndDlg);
    if (CPrintDlg::m_WindRect != NULL)
      {
        MoveWindow(hwndDlg, CPrintDlg::m_WindRect->top,
          CPrintDlg::m_WindRect->left,
          (CPrintDlg::m_WindRect->bottom - CPrintDlg::m_WindRect->top),
          (CPrintDlg::m_WindRect->right - CPrintDlg::m_WindRect->left),
          TRUE);
      }
    return 0;
  }
  else if (uMsg == WM_COMMAND)
  {
    if (wParam == CMD_PREVIEW) //0x2B - Open Preview
    {
      // Do some checks

      CPrintDlg::g_WindRectFlag = true;
      if (CPrintDlg::m_WindRect != NULL)
      {
        delete CPrintDlg::m_WindRect;
      }

      g_WindRect = new RECT();

      if (g_WindRect != NULL)
      {
        ZeroMemory(CPrintDlg::m_WindRect, sizeof(RECT));
        GetWindowRect(hwndDlg, CPrintDlg::m_WindRect);
        MoveWindow(hwndDlg, g_WindRect->top,
          CPrintDlg::m_WindRect->left,
        	(CPrintDlg::m_WindRect->bottom - CPrintDlg::m_WindRect->top),
        	(CPrintDlg::m_WindRect->right - CPrintDlg::m_WindRect->left),
        	TRUE);
        // Start the preview Window
        SendMessageA(hwndDlg, WM_COMMAND, MAKEWPARAM(1, 0),0);
        return 0;
      }
    }

    if (wParam < CMD_UNK1) //0x02 - Close Preview
    {
      if (CPrintDlg::m_WindRect != NULL && CPrintDlg::g_WindRectFlag != false)
      {
        delete CPrintDlg::m_WindRect;
        CPrintDlg::m_WindRect = NULL;
      }
      return 0;
    }
      if (wParam < CMD_UNK2) //0x22B - Exit Dialog
      {
        CPrintDlg::m_WindRectFlag = false;
        return 1;
      }
  }
  return 0;
}
~~~

>The code is what I reversed from the assembly for the DialogProc. I didn't go full out on it and recreate the class. But you will see a couple of names like `CPrintDlg::m_WindRect` which are just made up.

The code is pretty solid (save for any errors I may have added). But there is one problem. In the `WM_INITDIALOG` if statement at the top there is a potential for us to use uninitialized or once-initialized memory. In the case of the exception we were using address space `0x70` which, as you (should) know, is invalid most of the time.

## The Patch

In order to perform this I used [HxD][hxd]. Any good hex editor will do.

Here are the quick steps for this:

1. Open %WINDIR%\Downloaded Program Files\RSClientPrint.dll
2. Search for `e83bf5ffff` (call to SetDialogResources)
3. Directly after that enter `C6 46 44 00 EB 3C 90` starting at offset 0x167E7
4. Save file.


You can also check out my post on [Fixing a deadlock][deadlockpost] if you'd like to know how to do it all in WinDbg.

> **NOTE** This has only been tested on version 9.00.4053.00, if you use any other version you will have to do the analysis to find the code and offest.

Here is what we added. Pay special attention to the jmp instruction `eb 3c`. This is a short jump to an offset 0x3c (60) bytes from this location. If you have a different file version there is no guarantee this will work for you.

~~~
c6464400        mov     [esi+44h],0
eb3c            jmp     RSClientPrint!CPrintDlg::PrintHookProc+0xf0
90              nop
~~~

The key bits of code are right near the top of the reversed C++ code in the previous section. If we can somehow avoid the else statement we shouldn't fault when we initialize the application.

~~~Cpp
// Snippet from previous section.

if (uMsg == WM_INITDIALOG)
{
  SetDialogResources(hwndDlg);
  if (CPrintDlg::m_WindRect != NULL)
    {
      MoveWindow(hwndDlg, CPrintDlg::m_WindRect->top,
        CPrintDlg::m_WindRect->left,
        (CPrintDlg::m_WindRect->bottom - CPrintDlg::m_WindRect->top),
        (CPrintDlg::m_WindRect->right - CPrintDlg::m_WindRect->left),
        TRUE);
    }
  return 0;
}
~~~

And the assembly.

~~~
RSClientPrint!CPrintDlg::PrintHookProc+0xa1:
0cd873d8 83664400        and     dword ptr [esi+44h],0
0cd873dc 59              pop     ecx
0cd873dd eb48            jmp     RSClientPrint!CPrintDlg::PrintHookProc+0xf0 (0cd87427)
0cd873df ff7508          push    dword ptr [ebp+8]
0cd873e2 e83bf5ffff      call    RSClientPrint!CPrintDlg::SetDialogResources (0cd86922)
0cd873e7 8b4644          mov     eax,dword ptr [esi+44h]
0cd873ea 85c0            test    eax,eax
0cd873ec 741e            je      RSClientPrint!CPrintDlg::PrintHookProc+0xd5 (0cd8740c)
~~~

If you look at line `0cd873e7` you see where we are moving the pointer into the EAX register and testing it for NULL. If we look at the code we have 7 bytes to play with and there is a set of code that happens to fit in this spot.

~~~Cpp
// Snippet from previous section.

if (uMsg == WM_INITDIALOG)
{
  SetDialogResources(hwndDlg);
  CPrintDlg::m_WindRect = NULL;
  return 0;
}
~~~

And the assembly.

~~~
RSClientPrint!CPrintDlg::PrintHookProc+0xa1:
0cd873d8 83664400        and     dword ptr [esi+44h],0
0cd873dc 59              pop     ecx
0cd873dd eb48            jmp     RSClientPrint!CPrintDlg::PrintHookProc+0xf0 (0cd87427)
0cd873df ff7508          push    dword ptr [ebp+8]
0cd873e2 e83bf5ffff      call    RSClientPrint!CPrintDlg::SetDialogResources (0cd86922)
0cd873e7 c6464400        mov     [esi+44h],0
0cd873ec eb3c            jmp     RSClientPrint!CPrintDlg::PrintHookProc+0xf0 (45637427)  Branch
0cd873ef 90              nop
~~~

With this "simple" patch it effectively sets g_WindRect to 0 and jumps over the code that tries to set the Window position. I could have also just set a bunch of `nop` instructions and jumped to `RSClientPrint!CPrintDlg::PrintHookProc+0xf0` which is the start of the `return 0` portion of the disassembly. However, that would leave a dangling pointer.

Of course it goes without saying that this is completely unsupported and it brings about some other issues. Namely we haven't taken care of `[esi+0x48] (CPrintDlg::m_WindRectFlag)` who knows what state this is in and if it's getting overwritten.


## Final Notes
All of the ribs about IE11 breaking compatibility and the lack of preparation aside; IE11 is way better for security and "future proofing" your web applications. Plus as you might already know you only have until Feb 2016 to upgrade. This doesn't leave a lot of time. Depending on your application, your vendor, or your budget you may not be in a position to upgrade.

The real fix here is to upgrade. If you can't upgrade you can try these solutions for yourself, but they come with a cost of possible instability or security errors. So, of course, proceed with caution.

[deadlockpost]: {% post_url 2014-09-24-critical-section-deadlock %}
[tpgrow]: http://blogs.msdn.com/b/askie/archive/2009/03/09/opening-a-new-tab-may-launch-a-new-process-with-internet-explorer-8-0.aspx
[ericlaw]: https://twitter.com/ericlaw
[jonsamp]: https://twitter.com/jonathansampson
[hxd]: http://mh-nexus.de/en/hxd/
[art]: http://securityintelligence.com/understanding-ies-new-exploit-mitigations-the-memory-protector-and-the-isolated-heap/#.VW5Uyc9VhBd
