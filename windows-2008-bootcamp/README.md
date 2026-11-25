# Windows Server 2008 R2 — Bootcamp (lab)

| OS | Platform | Techniques |
|---|---|---|
| Windows Server 2008 R2 | Local lab VM | EternalBlue (MS17-010), Metasploit/Meterpreter, hash dumping, John the Ripper |

- **IP:** `10.0.2.26`
- **Access level:** `NT AUTHORITY\SYSTEM`
- **Status:** compromised

## Reconocimiento

Escaneo en profundidad con `nmap` usando `-sC -sV --script=vuln`. El resultado clave es que la máquina es vulnerable a **MS17-010** (EternalBlue).

## Explotación

Con Metasploit busco el módulo para esa vulnerabilidad, configuro el exploit y el payload, y lo lanzo. Obtengo una sesión de **Meterpreter** con el usuario de máximos privilegios (`SYSTEM`).

## Post-explotación

1. Con `hashdump` obtengo los hashes del sistema y los guardo en un fichero.
2. Uso **John the Ripper** con el formato `NT` (hashes NTLM de Windows) y el diccionario `rockyou.txt` en un ataque por diccionario.
3. Valido el acceso al sistema con las credenciales obtenidas usando un cliente SMB (credenciales omitidas).

**PWNED!**
