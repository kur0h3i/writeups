# Active Directory lab — de la DMZ a Domain Admin

| Entorno | Dificultad | Técnicas |
|---|---|---|
| Laboratorio AD simulado (`utad.local`, DC con Windows Server 2012 R2) | Medium | Subida de ficheros con bypass, pivoting SOCKS (NeoReGeorg), SUID, Pass-the-Hash, Kerberoasting, DCSync, RDP con hash |

> Laboratorio de la universidad, ficticio y ya inexistente. Los secretos (hashes y contraseñas) aparecen como marcadores.

## Escenario

- Dominio objetivo: `utad.local`, controlador de dominio **DC01** (`10.10.10.10`).
- Una máquina **DMZ** con doble interfaz (`192.168.100.4` y `10.10.10.20`) que hace de puente hacia la red interna `10.10.10.0/24`.

## 1. Reconocimiento

Barrido de `192.168.100.0/24` con `nmap -sn` y después `nmap -sV --script=vuln` sobre los hosts vivos. Hay cinco activos; dos IPs comparten claves SSH, así que son la misma máquina con doble interfaz (la DMZ).

El script `vuln` revela dos directorios web sin autenticación: `/backup/` (configuraciones sensibles) y `/uploads/` (subida de ficheros con listado de directorio).

## 2. Credenciales de dominio sin privilegios

En `/backup/config.txt` aparecen credenciales de dominio y un hash NTLM:

| Campo | Valor |
|---|---|
| Usuario | `UTAD.LOCAL\jsmith` |
| Hash NTLM | `<HASH_JSMITH>` |
| Dominio | `utad.local` |

Intento crackear el hash con `john` y `hashcat` contra `rockyou.txt` sin éxito. Lo reservo para **Pass-the-Hash**.

## 3. RCE en la DMZ: bypass de subida de ficheros

El panel `upload.php` solo parece aceptar JPG. Pruebo varias vías de bypass: doble extensión, *magic bytes* (`FFD8FF`) y extensiones alternativas (`.phtml`, `.php5`, `.phar`). La que funciona es **falsificar el `Content-Type` a `image/jpeg`** al subir un `.php`.

El servidor lo guarda en `/uploads/shell.php` y confirmo ejecución de comandos como `www-data` (uid=33).

## 4. Pivoting con NeoReGeorg

Para llegar a la red interna monto un túnel **SOCKS5** con NeoReGeorg: genero `tunnel.php`, lo subo renombrado a `.jpg` y levanto el cliente. Configuro `proxychains` con `socks5 127.0.0.1 8888`. Desde aquí, todo el tráfico hacia `10.10.10.0/24` pasa por la DMZ comprometida.

## 5. Escalada local en la DMZ

Enumero binarios SUID con `find / -perm -4000 2>/dev/null`. `/usr/bin/find` lo tiene, así que lo uso (GTFOBins) para lanzar una bash con `-p` y quedar como **root** en la DMZ.

## 6. Reconocimiento del DC

Con el túnel, escaneo el DC con `proxychains` y `nmap`. Con `netexec` sobre SMB y el hash de `jsmith` (**Pass-the-Hash**) obtengo: DC01, Windows Server 2012 R2, SMBv1 activo y *null authentication*.

## 7. EternalBlue (descartado)

El escaneo marca el DC como vulnerable a **MS17-010**. Lo intento con Metasploit (`ms17_010_eternalblue` con `bind_tcp` para pasar por el SOCKS), pero la sesión se abre y muere al instante, con varios BSOD y reinicios del DC. Descarto este vector por inestable y destructivo.

## 8. Kerberoasting

Con `jsmith` por Pass-the-Hash enumero SPNs con `impacket` (`GetUserSPNs`). Obtengo el TGS (`krb5tgs`) de la cuenta de servicio `svc_sql`, que está en **Domain Admins**. Lo crackeo con `john` y una wordlist con patrón de año: la contraseña es `<PASS_SVC_SQL>`.

## 9. Domain Admin: DCSync

Con `svc_sql` vuelco los hashes del dominio con `impacket` (`secretsdump`, **DCSync**). Obtengo el hash NTLM del Administrador (`<HASH_ADMIN>`) y confirmo que pertenece a Domain Admins y Enterprise Admins.

## 10. RDP al DC

Con el hash del Administrador abro una sesión RDP por **Pass-the-Hash** con `xfreerdp` (`/pth:`). PowerShell confirma `utad\administrator`: compromiso total del dominio.

## Mitigaciones

- Quitar los ficheros de configuración sensibles del servidor web y no guardar credenciales en claro.
- Validar el tipo real del fichero subido en el servidor (no solo el `Content-Type`) y no ejecutar PHP en `/uploads/`.
- Eliminar el bit SUID de binarios como `find`.
- Deshabilitar SMBv1 y parchear MS17-010.
- Contraseñas largas para cuentas de servicio, y no meter cuentas de servicio con SPN en Domain Admins (usar gMSA).
- Monitorizar replicación DCSync y peticiones TGS anómalas.
