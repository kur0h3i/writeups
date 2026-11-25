# HTB writeups

Writeups (in Spanish) of retired Hack The Box machines and lab VMs, written while preparing for OSCP.
Each one covers enumeration, exploitation and privilege escalation. Flags and credentials are redacted.

### Hack The Box

| Machine | OS | Difficulty | Techniques |
|---|---|---|---|
| [Devvortex](devvortex/) | Linux | Easy | Subdomain enumeration, Joomla CVE-2023-23752 (information disclosure), webshell via template, hash cracking, sudo abuse (apport-cli) |
| [GoodGames](goodgames/) | Linux | Easy | SQL injection, hash cracking, SSTI (Jinja2) to RCE, Docker container escape, credential reuse |
| [Paper](paper/) | Linux | Easy | Virtual host discovery, WordPress CVE-2019-17671, chat bot abuse, Polkit CVE-2021-3560 |
| [Sau](sau/) | Linux | Easy | SSRF in Request Baskets (CVE-2023-27163), Maltrail command injection, sudo abuse (systemctl) |
| [Jerry](jerry/) | Windows | Easy | Default credentials brute force (hydra), malicious WAR upload on Tomcat |

### Other labs

| Lab | OS | Difficulty | Techniques |
|---|---|---|---|
| [Active Directory lab](active-directory-lab/) | Windows (AD) | Medium | File-upload RCE, SOCKS pivoting, Pass-the-Hash, Kerberoasting, DCSync, Domain Admin |
| [Windows Server 2008 R2 Bootcamp](windows-2008-bootcamp/) | Windows | — | EternalBlue (MS17-010), Meterpreter, hash dumping, John the Ripper |

Only retired machines are published, in line with the Hack The Box rules.
