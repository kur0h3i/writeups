# Hack The Box — Paper

| OS | Difficulty | Techniques |
|---|---|---|
| Linux | Easy | Virtual host discovery, WordPress CVE-2019-17671, chat bot abuse, Polkit CVE-2021-3560 |

## Enumeracion
Tiramos un nmap basico para poder observar que puertos hay abiertos, de ahi
descubrimos el puerto 22 (ssh), 80 (http) y 443 (https)

Una vez sabemos cuales son los posibles puertos vamos a indagar en sus
versiones y usar los scripts default.
localhost.localdomain   en un CN
Indica un certificado auto-firmado con la configuración por defecto de la distro.
No aporta información sobre el hostname real del servidor.

 En la pagina principal al entrar tanto por http o https vemos la Test Page de
Apache/CentOS
Es la página por defecto que sirve Apache cuando se instala pero no se ha
configurado contenido para el sitio principal. Significa que en la raíz del
servidor no hay una web "de verdad" — solo el placeholder de bienvenida.

 Hacemos un curl -I a la IP para que nos devuelva las cabeceras HTTP sin el
cuerpo de la respuesta. En este caso descubrimos la cabecera X-Backend-Server:
office.paper , que nos revela un hostname interno que Apache está usando por

detrás.

Añado este vhost a la lista de hosts

Encontramos una pagina nueva en http://office.paper/

## Pre-Explotacion
He encontrado un panel de admin de wordpress bsucando por la web asique
vamos a buscar que version de wordpress corre

Vamos a usar el CVE-2019-17671 que nos va a permitir leer drafts que no se
muestra a simple vista en los mensajes del blog gracias a abusar del static.
http://office.paper/?static=1

 Si leemos con atencion nos dice que hay otro vhost que seria el
chat.office.paper

Agregamos dentro de etc/hosts

Me creo una cuenta y entro al chat

Dento del chat leo que hay un bot con el que se puede interactuar llamado
recyclops

## Explotacion
Usando le bot y moviendome a traves de los directrorios encuentro donde se
encuentra los archivos del bot

Saco el .env que es donde esta la información sensible

 Y logro un acceso por ssh dentro de la maquina y obtengo la primera flag

## Post-Explotacion
He buscado el polkit

He buscado un CVE asociado al polkit que en este caso es el CVE-2021-3560

 Lo explotmos usando este poc : https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation

Usamos el usuario al que hemos cambiado para poder entrar como sudo

Obtenemos root

Obtenemos la flag

 PWNED!
