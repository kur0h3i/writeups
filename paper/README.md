# Hack The Box — Paper

| OS | Difficulty | Techniques |
|---|---|---|
| Linux (CentOS) | Easy | Virtual host discovery, WordPress CVE-2019-17671, chat bot abuse, Polkit CVE-2021-3560 |

## Enumeración

Un escaneo básico con `nmap` muestra los puertos **22** (SSH), **80** (HTTP) y **443** (HTTPS). Después lanzo el escaneo de versiones con los scripts por defecto.

- El certificado HTTPS es autofirmado y tiene `localhost.localdomain` como CN. Es la configuración por defecto de la distribución y no revela el hostname real.
- Tanto por HTTP como por HTTPS aparece la *Test Page* de Apache/CentOS: el servidor no tiene una web real en la raíz, solo la página de bienvenida.

Con `curl -I` sobre la IP leo las cabeceras HTTP sin el cuerpo. Aparece la cabecera:

```
X-Backend-Server: office.paper
```

Es un hostname interno que Apache usa por detrás. Lo añado a `/etc/hosts` y encuentro una web nueva en `http://office.paper/`.

## Pre-explotación

Buscando por la web encuentro el panel de administración de WordPress, así que averiguo qué versión corre.

Esa versión es vulnerable al **CVE-2019-17671**, que permite leer *drafts* que no se muestran en el blog abusando del parámetro `static`:

```
http://office.paper/?static=1
```

Leyendo los borradores con atención, uno menciona otro virtual host: `chat.office.paper`. Lo añado también a `/etc/hosts`, me creo una cuenta y entro al chat.

Dentro del chat hay un bot, **recyclops**, con el que se puede interactuar.

## Explotación

Con el bot puedo moverme por los directorios y localizar los ficheros del propio bot. En su `.env`, que es donde guarda la información sensible, obtengo credenciales que me dan acceso por **SSH** a la máquina. Ahí consigo la primera flag.

## Post-explotación

Busco versiones de `polkit` y encuentro un CVE asociado: **CVE-2021-3560** (escalada de privilegios en Polkit). Lo explotamos con este PoC:

- <https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation>

Con el usuario creado por el exploit puedo entrar como `sudo`, obtengo **root** y leo la flag.

**PWNED!**
