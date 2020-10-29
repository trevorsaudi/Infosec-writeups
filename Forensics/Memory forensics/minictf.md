##miniCTF memory forensics


We start off by identifying the profile in which volatility will run in for this 


```
volatility -f memdump.mem imageinfo
Volatility Foundation Volatility Framework 2.6.1
INFO    : volatility.debug    : Determining profile based on KDBG search...
          Suggested Profile(s) : Win10x64_17134, Win10x64_14393, Win10x64_10586, Win10x64_16299, Win2016x64_14393, Win10x64_17763, Win10x64_15063 (Instantiated with Win10x64_15063)
                     AS Layer1 : SkipDuplicatesAMD64PagedMemory (Kernel AS)
                     AS Layer2 : FileAddressSpace (/home/saudi/Desktop/Projects/Personal Research/Forensics/Memory Corruption forensics/memdump.mem)
                      PAE type : No PAE
                           DTB : 0x1ad002L
                          KDBG : 0xf800ca7bd520L
          Number of Processors : 1
     Image Type (Service Pack) : 0
                KPCR for CPU 0 : 0xfffff800c9714000L
             KUSER_SHARED_DATA : 0xfffff78000000000L
           Image date and time : 2018-08-06 18:13:42 UTC+0000
     Image local date and time : 2018-08-06 14:13:42 -0400

```

#Question #1
Find the running rogue (malicious) process. The flag is the MD5 hash of its PID.


select the profile and run pstree to view the processes

```

volatility -f memdump.mem --profile=Win10x64_17134 pstree


. 0xffffc20c6be63040:MemCompression                  1096      4     34      0 2018-08-01 19:20:31 UTC+0000
. 0xffffc20c6a212040:smss.exe                         500      4      2      0 2018-08-01 19:20:20 UTC+0000
 0xffffc20c6b8ed580:csrss.exe                         680    672     14      0 2018-08-01 19:20:25 UTC+0000
 0xffffc20c6b910080:winlogon.exe                      732    672      6      0 2018-08-01 19:20:25 UTC+0000
. 0xffffc20c69d00580:userinit.exe                    4756    732      0 ------ 2018-08-01 19:20:57 UTC+0000
.. 0xffffc20c69cfe580:explorer.exe                   4824   4756    125      0 2018-08-01 19:20:58 UTC+0000
... 0xffffc20c6ddad580:svchost.exe                   8560   4824     10      0 2018-08-01 20:13:10 UTC+0000
... 0xffffc20c6e495080:cmd.exe                       8868   4824      0 ------ 2018-08-01 19:40:14 UTC+0000
... 0xffffc20c6cdf4580:scvhost.exe                    360   4824      0 ------ 2018-08-01 19:56:45 UTC+0000
... 0xffffc20c6ab70080:svchost.exe                   8852   4824      0 ------ 2018-08-01 19:59:49 UTC+0000
... 0xffffc20c6c095580:MSASCuiL.exe                  6268   4824      3      0 2018-08-01 19:21:56 UTC+0000
... 0xffffc20c6d5ac340:svchost.exe.ex                5528   4824      0 ------ 2018-08-01 19:52:20 UTC+0000
... 0xffffc20c6ab2b580:svchost.exe.ex                6176   4824      0 ------ 2018-08-01 19:52:19 UTC+0000
... 0xffffc20c6d6fc580:svchost.exe                  10012   4824      0 ------ 2018-08-01 19:49:19 UTC+0000
... 0xffffc20c6d4d2080:dxdiag.exe                    6324   4824      0 ------ 2018-08-01 19:51:18 UTC+0000
... 0xffffc20c6daf9580:notepad.exe                   7968   4824      0 ------ 2018-08-01 19:57:10 UTC+0000
... 0xffffc20c6d789580:Bubbles.scr                   6948   4824      0 ------ 2018-08-01 19:50:30 UTC+0000
... 0xffffc20c6e24f580:xwizard.exe                    252   4824      0 ------ 2018-08-01 19:51:52 UTC+0000
.
 0xffffc20c6d756580:   
```
After observing the running processes you can see that scvhost.exe is executed when explorer is run, so let's grep for svc and it's parent ID


volatility -f memdump.mem --profile=Win10x64_17134 pstree | grep -E "^Name|svchost.exe" | grep -E "^Name|4824"

```
Name                                                  Pid   PPid   Thds   Hnds Time
... 0xffffc20c6ddad580:svchost.exe                   8560   4824     10      0 2018-08-01 20:13:10 UTC+0000
... 0xffffc20c6ab70080:svchost.exe                   8852   4824      0 ------ 2018-08-01 19:59:49 UTC+0000
... 0xffffc20c6d5ac340:svchost.exe.ex                5528   4824      0 ------ 2018-08-01 19:52:20 UTC+0000
... 0xffffc20c6ab2b580:svchost.exe.ex                6176   4824      0 ------ 2018-08-01 19:52:19 UTC+0000
... 0xffffc20c6d6fc580:svchost.exe                  10012   4824      0 ------ 2018-08-01 19:49:19 UTC+0000
... 0xffffc20c6dbc5340:svchost.exe                   7852   4824      0 ------ 2018-08-01 19:49:21 UTC+0000
... 0xffffc20c6d82e080:svchost.exe                   1404   4824      0 ------ 2018-08-01 19:54:55 UTC+0000
... 0xffffc20c6d99b580:svchost.exe.ex                8140   4824      0 ------ 2018-08-01 19:52:16 UTC+0000


```

Ok, so from the above information PID 8560 has threads running which tells that it is a running process.
We can confirm by running psscan. PID 8560 has no time exited hence being the rogue pricess


```
volatility -f memdump.mem --profile=Win10x64_17134 psscan | grep -E "^Offset|svchost.exe" | grep -E "^Offset|4824"

Offset(P)          Name                PID   PPID PDB                Time created                   Time exited                   
0x0000c20c6ab2b580 svchost.exe.ex     6176   4824 0x000000004d100002 2018-08-01 19:52:19 UTC+0000   2018-08-01 19:52:19 UTC+0000  
0x0000c20c6ab70080 svchost.exe        8852   4824 0x0000000096f00002 2018-08-01 19:59:49 UTC+0000   2018-08-01 20:00:08 UTC+0000  
0x0000c20c6d5ac340 svchost.exe.ex     5528   4824 0x0000000119400002 2018-08-01 19:52:20 UTC+0000   2018-08-01 19:52:20 UTC+0000  
0x0000c20c6d6fc580 svchost.exe       10012   4824 0x0000000136200002 2018-08-01 19:49:19 UTC+0000   2018-08-01 19:49:19 UTC+0000  
0x0000c20c6d82e080 svchost.exe        1404   4824 0x00000000a0f00002 2018-08-01 19:54:55 UTC+0000   2018-08-01 19:56:35 UTC+0000  
0x0000c20c6d99b580 svchost.exe.ex     8140   4824 0x00000000b8600002 2018-08-01 19:52:16 UTC+0000   2018-08-01 19:52:16 UTC+0000  
0x0000c20c6dbc5340 svchost.exe        7852   4824 0x000000003ff00002 2018-08-01 19:49:21 UTC+0000   2018-08-01 19:49:22 UTC+0000  
0x0000c20c6ddad580 svchost.exe        8560   4824 0x00000000b2200002 2018-08-01 20:13:10 UTC+0000  
```
```
echo -n 8560 | md5sum #echo -n removes the trailing characters that the echo command appends at the end of the output
bc05ca60f2f0d67d0525f41d1d8f8717
```
#Question #2
Find the running rogue (malicious) process and dump its memory to disk. You'll find the 32-character flag within that process's memory.

-- we start by dumping the memory of the rogue process to disk and extracting the human-readable strings

```
volatility -f memdump.mem --profile=Win10x64_17134 memdump -p 8560 -D . strings memdump.mem > 8560.dmp.strings

-D supplies the output directory
-p specifies the pricess id
```

The file contains so many so we filter out using strings and narrow to 32 bytes
```
strings -n 32 8560.dmp.strings | sort | uniq > 8560.dmp.strings.32
cat 850.dmp.32 | grep "fl"


  "contents": "da391kdasdaadsssssss    t.h.e. fl.ag.is. M2ExOTY5N2YyOTA5NWJjMjg5YTk2ZTQ1MDQ2Nzk2ODA=",
  decodes to 3a19697f29095bc289a96e4504679680



```

#Question #3
What is the MAC address of this machine's default gateway? The flag is the MD5 hash of that MAC address in uppercase with dashes (-) as delimiters. Example: 01-00-A4-FB-AF-C2.

#Question #4
Find the full path of the browser cache created when an analyst visited "www.13cubed.com." The path will begin with "Users\." Convert the path to uppercase. The flag is the MD5 hash of that string.