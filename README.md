# HTB writeups

Writeups (in Spanish) of retired Hack The Box machines, written while preparing for OSCP.
Each one covers enumeration, exploitation and privilege escalation. Flags and credentials are redacted.

| Machine | OS | Difficulty | Techniques |
|---|---|---|---|
| [Devvortex](devvortex/) | Linux | Easy | Subdomain enumeration, Joomla CVE-2023-23752 (information disclosure), webshell via template, hash cracking, sudo abuse (apport-cli) |
| [GoodGames](goodgames/) | Linux | Easy | SQL injection, hash cracking, SSTI (Jinja2) to RCE, Docker container escape, credential reuse |
| [Paper](paper/) | Linux | Easy | Virtual host discovery, WordPress CVE-2019-17671, chat bot abuse, Polkit CVE-2021-3560 |
| [Sau](sau/) | Linux | Easy | SSRF in Request Baskets (CVE-2023-27163), Maltrail command injection, sudo abuse (systemctl) |
| [Jerry](jerry/) | Windows | Easy | Default credentials brute force (hydra), malicious WAR upload on Tomcat |

Only retired machines are published, in line with the Hack The Box rules.
