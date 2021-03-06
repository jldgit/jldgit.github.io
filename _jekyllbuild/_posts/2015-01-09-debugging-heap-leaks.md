---
layout: post
title: Debugging - Finding a native heap leak with WinDbg
tags:
- debugging
- windbg
---
While writing the [MySQL .NET UDF Plugin][mysqldotnet] I had to break away from my daily use of .NET. I have done a bit with Win32 COM before but not enough to know all of the ins and outs. But, as I plow through some books and bad code examples I introduce all kinds of bugs. Most notably memory leaks. I will show what leaks I found and how I fixed them using a couple of WinDbg commands as well as a few utilities.

## Leaks
I bet if you're here, you're guilty of introducing a memory leak once or twice. In the .NET world (where I hail from) these leaks were less common and not traditional in the sense of a true managed leak. In .NET you can't *leak* memory but you can *root* it in some structure that will never allow it to be reclaimed.

A key example of this is when you have a static list defined in your class and you continue to add items to this list but never remove them. Over time you will see an increase of consumed memory and eventually (on a 32-bit system) you could run out of memory.

However, on a native application, things get a little bit hairy when you start allocating memory without caution. Take for example a simple function that creates a character string. In normal operation your code may look something like this:

~~~C
void CreateString()
{
  char myBuffer[20] = {};
  char *initString = "Fully initialized string.";
  //do work
}
~~~

Defining a variable this way works because it only lives in the scope of the function. But, this requires you to know the constant size of your string before compile time. Not very useful when you're planning on taking in varying amounts of data varying amounts of time. So, the answer is to dynamically allocate new memory of varying size when you need it.

~~~C
void CreateDynamicString(int inputSize)
{
    char *initString = new char[inputSize];
    //do work
}
~~~

This nonsensical piece of code has now created a leak. Depending on frequency and intent, this could be a slow leak over time, or could be something drastic instantly. The correct way to handle this is to delete the data from the heap when you're done.

~~~C
void CreateDynamicString(int inputSize)
{
  char *initString = new char[inputSize];
  // do work
  delete initString;
}

char* ReturnDynamicString(int inputSize)
{
  char *initString = new char[inputSize];
  // do work
  return initString; // see-ya!
}
~~~

This may be fine for a trivial application, but what about something that has to manipulate and return a string. Who deletes the data? How do you know where it came from? Is it okay to delete?

## The Wrong Way
The code above used raw pointers to pass data around. In the MySQL extension we we're marshalling data to and from .NET and we will wrap our string in a [_bstr_t][bstr]. This is a reference counted wrapper for a [BSTR][bstrRaw] data type; which is a padded wchar_t array. It also provides some helper functions to get the length, counts, and char representations of the string.

If we were to pass in an array of strings to .NET, we use a SafeArray of BSTR, not `_bstr_t`; you will see why in a minute. So lets look at an example:

~~~Cpp
_bstr_t RunStrings(char** input, unsigned long *lengths, uint args)
{
  SAFEARRAYBOUND rgsabound[1];
  rgsabound[0].lLbound = 0;
  rgsabound[0].cElements = args - (codepageIndex - 1);
  SAFEARRAY* sa = SafeArrayCreate(VT_BSTR, 1, rgsabound);
  HRESULT hr = SafeArrayLock(sa);
    for (uint ix = codepageIndex; ix <= args; ix++)
    {
      int txtLen = lengths[ix] + 1;
      auto txt = new wchar_t[txtLen] {};
      MultiByteToWideChar(*codepage, NULL, input[ix],
        lengths[ix], txt, txtLen);
      ((_bstr_t*)sa->pvData)[ix - codepageIndex] = *(new _bstr_t(txt));
      delete txt;
    }
  hr = SafeArrayUnlock(sa);
  auto retstring = RunStringManged(sa);
  SafeArrayDestroy(sa);
  return retstring;
}
~~~

On line 14 of this code example you will see that I am newing up a new `_bstr_t` for each element of the array. On line 19 I am calling the proper SafeArrayDestroy which should mean that ***"...safe arrays of BSTR will have the SysFreeString function called on each element."*** Well if you have BSTRs, yes. It will. But, if you do it the wrong way (aka my way) it will free the underlying string, but it will not free the `_bstr_t` itself.

## The Right Way
The wrong way was tricky only because I looked at the type and made an incorrect assumption about what I needed to pass into the SafeArray. The corrected code is only different on line 14. We only have to call SysAllocString to create the string on the heap. This is less overhead and it returns pretty quick. Plus it has the benefit of being cleaned up properly b the SafeArrayDestroy mentioned above.

~~~Cpp
_bstr_t RunStrings(char** input, unsigned long *lengths, uint args)
{
  SAFEARRAYBOUND rgsabound[1];
  rgsabound[0].lLbound = 0;
  rgsabound[0].cElements = args - (codepageIndex - 1);
  SAFEARRAY* sa = SafeArrayCreate(VT_BSTR, 1, rgsabound);
  HRESULT hr = SafeArrayLock(sa);
  for (uint ix = codepageIndex; ix <= args; ix++)
  {
    int txtLen = lengths[ix] + 1;
    auto txt = new wchar_t[txtLen] {};
      MultiByteToWideChar(*codepage, NULL, input[ix],
        lengths[ix], txt, txtLen);
        ((BSTR*)sa->pvData)[ix - codepageIndex] = SysAllocString(txt);
        delete txt;
  }
  hr = SafeArrayUnlock(sa);
  auto retstring = RunStringManged(sa);
  SafeArrayDestroy(sa);
  return retstring;
}
~~~

## Finding the leak
Here comes the fun part. In order to find the leak I needed to use gflags and turn on the user mode stack trace database. With this enabled I can get an accurate map of what was allocated, and we can see the call stack. It adds considerable overhead to the run time of the application. My tests show that with it enabled my queries went from 11 seconds to 33 seconds (and higher as the memory grew). So, do remember to disable the flags once its done.

### Step 1 - Enable GFlags
Inside of your WinDbg directory you should find [gflags.exe][gflags]. Execute the following enable command to turn on the user mode stack trace database. This allows you to find out what stack trace allocated a bit of memory.

~~~
ENABLE
=========================
gflags /i mysqld.exe +ust

DISABLE
=========================
gflags /i mysqld.exe -ust
~~~

>Once your settings are **enabled** all allocations are being tracked, including BSTR using SysStringAlloc.

### Step 2 - Execute mysqld.exe in WinDbg
A little caveat here is that I a version of mysql that I [compiled myself][p1]. This gives the added benefit of mysqld symbols, so in this case the stack trace doesn't look all wonky. I used the command line below because it allows me to execute the server process and it also skips past the initial break point.

~~~
windbg -g mysqld.exe
~~~

### Step 3 - Inspect the heap
Once the application is started and ready to accept connections, the first thing you should do is look at your heap usage with zero queries executed. This will give you the baseline you need to determine the heap growth. Any allocation using standard operators will use the default heap. There are other heaps assigned in the process, but all of our data is using the new operator.

~~~
0:019> !heap -s
NtGlobalFlag enables following debugging aids for new heaps:
stack back traces
LFH Key                   : 0x1a818dce
Termination on corruption : ENABLED
Heap     Flags   Reserv  Commit  Virt   Free  List   UCR  Virt  Lock  Fast
                  (k)     (k)    (k)     (k) length      blocks cont. heap
-----------------------------------------------------------------------------
Virtual block: 05c80000 - 05c80000 (size 00000000)
Virtual block: 058f0000 - 058f0000 (size 00000000)
Virtual block: 0eb40000 - 0eb40000 (size 00000000)
Virtual block: 0f810000 - 0f810000 (size 00000000)
Virtual block: 10e60000 - 10e60000 (size 00000000)
00690000 08000002   16384  11056  16384     14    41     5    5      0   LFH
004a0000 08001002    1088    312   1088     19     2     2    0      0   LFH
005a0000 08001002     256      4    256      2     1     1    0      0
00f10000 08001002    1088    264   1088      5     7     2    0      0   LFH
01040000 08001002    1088    272   1088      8     4     2    0      0   LFH
00fb0000 08001002     256     28    256      3    11     1    0      0
00240000 08001002      64      4     64      2     1     1    0      0
05a50000 08001002      64      4     64      2     1     1    0      0
05c40000 08011002     256      4    256      1     2     1    0      0
05c10000 08001002    1088    268   1088      9     3     2    0      0   LFH
05a10000 08001002     256      4    256      1     2     1    0      0
11150000 08001002      64     16     64      2     2     1    0      0
117d0000 08041002     256      4    256      2     1     1    0      0
-----------------------------------------------------------------------------
~~~

>If we look at this listing we can see that the largest heap is 16MB (16384k = 16384 * 1024 = 16777216).


### Step 4 - Execute a few large queries
When running my test harness I used the [MySQL employee sample database][sampdb] for the backing data store. If you look at [post 7][p7] I mention it along with the queries I ran. My test harness spins up a few threads and executes the following query.

~~~SQL
SELECT  CAST(mysqldotnet_string("MySQLCustomClass.CustomMySQLClass",
"MULTI", first_name, first_name, first_name) AS char) FROM employees.employees
~~~

This query will effectively execute the `RunStrings()` we listed in the "Wrong Way" section of this post.

### Step 5 - Let the memory grow and inspect heap
Woah! We are now using over 453MB!! of the heap. In my case I only ran the queries for about 20 seconds. We would have easily caused an OOM for this application.

~~~
0:022> !heap -s
NtGlobalFlag enables following debugging aids for new heaps:
stack back traces
LFH Key                   : 0x1a818dce
Termination on corruption : ENABLED
Heap     Flags   Reserv  Commit  Virt   Free  List   UCR  Virt  Lock  Fast
                  (k)     (k)    (k)     (k) length      blocks cont. heap
-----------------------------------------------------------------------------
Virtual block: 05c80000 - 05c80000 (size 00000000)
Virtual block: 058f0000 - 058f0000 (size 00000000)
Virtual block: 0eb40000 - 0eb40000 (size 00000000)
Virtual block: 0f810000 - 0f810000 (size 00000000)
Virtual block: 10e60000 - 10e60000 (size 00000000)
00690000 08000002  453568 446212 453568   1526   233    32    5      2   LFH
004a0000 08001002    1088    312   1088     19     2     2    0      0   LFH
005a0000 08001002     256      4    256      2     1     1    0      0
00f10000 08001002    1088    264   1088      5     7     2    0      0   LFH
01040000 08001002    1088    292   1088     11     4     2    0      0   LFH
00fb0000 08001002     256     28    256      3    11     1    0      0
00240000 08001002      64      4     64      2     1     1    0      0
05a50000 08001002      64      4     64      2     1     1    0      0
05c40000 08011002     256      4    256      1     2     1    0      0
05c10000 08001002    1088    292   1088     16     3     2    0      0   LFH
05a10000 08001002     256      4    256      1     2     1    0      0
11150000 08001002    1088    292   1088     18     4     2    0      0   LFH
117d0000 08041002     256      4    256      2     1     1    0      0
12be0000 08001002     256    256    256      6     6     1    0      0   LFH
11c30000 08041002     256     20    256      0     1     1    0      0
005f0000 08001002      64     20     64      2     4     1    0      0
126f0000 08001002     256      4    256      0     1     1    0      0
15000000 08041002     256      4    256      2     1     1    0      0
-----------------------------------------------------------------------------

~~~

### Step 6 - Inspect the culprit heap
As we saw in the previous step, heap 00690000 is the culprit of our leak. Let's take a look at the stats of this heap to see what has taken place. This can take a minute since the !heap command has to walk all of the allocations.

~~~
0:022> !heap -stat -h 00690000
heap @ 00690000
group-by: TOTSIZE max-display: 20
size     #blocks     total     ( %) (percent of total busy bytes)
30 31520c - 93f6240  (49.26)
28 3150a7 - 7b49a18  (41.05)
800240 1 - 800240  (2.66)
701400 1 - 701400  (2.33)
38 d6f5 - 2f0598  (0.98)
21c494 1 - 21c494  (0.70)
204000 1 - 204000  (0.67)
20 de4f - 1bc9e0  (0.58)
fe968 1 - fe968  (0.33)
43a24 2 - 87448  (0.18)
3a9c0 2 - 75380  (0.15)
75300 1 - 75300  (0.15)
30d80 1 - 30d80  (0.06)
79e8 6 - 2db70  (0.06)
28a24 1 - 28a24  (0.05)
3e4 9d - 262d4  (0.05)
10fc4 2 - 21f88  (0.04)
3bc 5f - 162c4  (0.03)
ff8 12 - 11f70  (0.02)
8000 2 - 10000  (0.02)
~~~

The `!heap -stat -h [HEAP]` command outputs the contents of the heap and orders by what has the most busy bytes; the busy bytes indicate that something was malloc'd or new'd up but not deleted.


### Step 7 - Inspect the culprit items
As we saw in the previous step, heap 00690000 is the culprit of our leak. Let's take a look at the stats of this heap to see what has taken place. This can take a minute since the !heap command has to walk all of the allocations.

Now we need to take a look at what is allocating all of these 48-byte (0x30 byte) heap blocks.

~~~
0:022> !heap -flt s 30
_HEAP @ 690000
HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
00694ce0 0009 0000  [00]   00694cf8    00030 - (busy)
006958d0 0009 0009  [00]   006958e8    00030 - (busy)
006dc718 0009 0009  [00]   006dc730    00030 - (busy)
006e2db8 0009 0009  [00]   006e2dd0    00030 - (busy)
006e4f80 0009 0009  [00]   006e4f98    00030 - (busy)
006ed198 0009 0009  [00]   006ed1b0    00030 - (busy)
mswsock!WinsockTlsDataList
006ed1e0 0009 0009  [00]   006ed1f8    00030 - (busy)
006ed228 0009 0009  [00]   006ed240    00030 - (busy)
006ed390 0009 0009  [00]   006ed3a8    00030 - (busy)
006ed4f8 0009 0009  [00]   006ed510    00030 - (busy)
006fce98 0007 0009  [00]   006fcea0    00030 - (free)
006fdc68 0007 0007  [00]   006fdc70    00030 - (free)
...
1b33ee98 0009 0009  [00]   1b33eeb0    00030 - (busy)
1b33eee0 0009 0009  [00]   1b33eef8    00030 - (busy)
1b33ef28 0009 0009  [00]   1b33ef40    00030 - (busy)
1b33ef70 0009 0009  [00]   1b33ef88    00030 - (busy)
1b33efb8 0009 0009  [00]   1b33efd0    00030 - (busy)
1b33f000 0009 0009  [00]   1b33f018    00030 - (busy)
1b33f048 0009 0009  [00]   1b33f060    00030 - (busy)
user interrupted
_HEAP @ 4a0000
~~~

I let the heap allocation list spin for a minute (it will fill your console buffer) and just stop it at some random place. From the list I pick a random item. It is possible that you won't see a stack trace for the item you pick. In that instance, just keep trying from your list or let a few thousand more roll off the screen.

In the instance below I call `!heap -p -a [UsrPtr]` which displays the call stack when the allocation was made.

~~~
0:022> !heap -p -a 1b33f060
address 1b33f060 found in
_HEAP @ 690000
HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
1b33f048 0009 0000  [00]   1b33f060    00030 - (busy)
77d8dff2 ntdll!RtlAllocateHeap+0x00000274
5b7fc6e1 MSVCR120D!_heap_alloc_base+0x00000051
5b80d72f MSVCR120D!_free_dbg_nolock+0x000006ff
5b80dbcd MSVCR120D!_nh_malloc_dbg+0x0000007d
5b80db7a MSVCR120D!_nh_malloc_dbg+0x0000002a
5b80e5a9 MSVCR120D!malloc+0x00000019
5b7fc26f MSVCR120D!operator new+0x0000000f
6114f34c MySQLDotNet!_bstr_t::Data_t::operator new+0x0000000c
6114e2a0 MySQLDotNet!_bstr_t::_bstr_t+0x00000040
61144e62 MySQLDotNet!RunStrings+0x00000262
611465d7 MySQLDotNet!mysqldotnet_string+0x00000227
10ea180 mysqld!udf_handler::val_str+0x00000070
10ea058 mysqld!Item_func_udf_str::val_str+0x00000018
11b955c mysqld!Item_char_typecast::val_str+0x000000ec
10d74b3 mysqld!Item::send+0x000001b3
10c95ff mysqld!Protocol::send_result_set_row+0x0000006f
110a61d mysqld!select_send::send_data+0x0000005d
11ca09b mysqld!end_send+0x0000009b
11caec0 mysqld!evaluate_join_record+0x00000150
11d9d01 mysqld!sub_select+0x00000081
11c9c8c mysqld!do_select+0x000001bc
11cc04d mysqld!JOIN::exec+0x0000100d
11d208b mysqld!mysql_select+0x000001bb
11cd7b3 mysqld!handle_select+0x000000b3
1121a1b mysqld!execute_sqlcom_select+0x000001db
112392d mysqld!mysql_execute_command+0x000004cd
112709a mysqld!mysql_parse+0x000000ca
1120a26 mysqld!dispatch_command+0x00000616
11214f1 mysqld!do_command+0x000000d1
1146c30 mysqld!do_handle_one_connection+0x00000110
11477c5 mysqld!handle_one_connection+0x00000035
124e476 mysqld!pthread_start+0x00000016
~~~

Yup. It's my method. Interestingly enough we can see that the new operator calls malloc which calls RtlAllocateHeap. You can also see that we're using the debug heap.

### Step 8 - Find the code
If you look at the line `61144e62 MySQLDotNet!RunStrings+0x00000262` you can use the address (or the symbol + offeset) to give you the actual line of code. Which unsurprisingly is `((_bstr_t*)sa->pvData)[ix - codepageIndex] = *(new _bstr_t(txt));`

Now that you found your code it's easy to fix. Something to note is there are a few operations that take place on this line of code. Also, if you look into the stack a bit more you can see that it's not just the action of the new operator. It's the new operator that's nested inside of the `_bstr_t` constructor.

~~~
0:043> u MySQLDotNet!RunStrings+0x00000262
MySQLDotNet!RunStrings+0x262 [c:\users\james\source\repos\mysql_udf\mysql_udf.c @ 141]:
61144e62 894580          mov     dword ptr [ebp-80h],eax
61144e65 eb07            jmp     MySQLDotNet!RunStrings+0x26e (61144e6e)
61144e67 c7458000000000  mov     dword ptr [ebp-80h],0
61144e6e 8b4d80          mov     ecx,dword ptr [ebp-80h]
61144e71 894da0          mov     dword ptr [ebp-60h],ecx
61144e74 c745fcffffffff  mov     dword ptr [ebp-4],0FFFFFFFFh
61144e7b 8b4da0          mov     ecx,dword ptr [ebp-60h]
61144e7e e83acdffff      call    MySQLDotNet!ILT+3000(??B_bstr_tQBEPA_WXZ) (61141bbd)
~~~

~~~
0:043> u 6114e2a0
MySQLDotNet!_bstr_t::_bstr_t+0x40
[c:\program files (x86)\microsoft visual studio 12.0\vc\include\comutil.h @ 322]:
~~~

~~~Cpp
// Construct a _bstr_t from a const whar_t*
//
inline _bstr_t::_bstr_t(const wchar_t* s)
: m_Data(new Data_t(s))
{
  if (m_Data == NULL) {
    _com_issue_error(E_OUTOFMEMORY);
  }
}
~~~

## Conclusion
A few misplaced allocations can cause a lot of problems down the road. But, a little bit of WinDbg magic and you can find your problems extremely fast. Love your debugger, it will save your many more times than your IDE.




[mysqldotnet]: https://github.com/jldgit/mysql_udf_dotnet
[bstr]: http://msdn.microsoft.com/en-us/library/zthfhkd6.aspx
[gflags]:http://msdn.microsoft.com/en-us/library/windows/hardware/ff549557(v=vs.85).aspx
[flagtbl]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff549596(v=vs.85).aspx
[bstrRaw]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms221069%28v=vs.85%29.aspx
[p1]: {% post_url 2014-11-11-extending-mysql-server-part1 %}
[p7]: {% post_url 2014-12-15-extending-mysql-server-part7 %}
[sampdb]: https://dev.mysql.com/doc/employee/en/
