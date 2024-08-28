---
title: Writeup de la máquina BoardLight
date: 2024-08-03
categories: [WRITEUPS, HackTheBox]
tags: [dolibar]  # TAG names should always be lowercase
---

![picture](/assets/images/hackthebox/board1.png)

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo de puertos con **NMAP**, para comprobar los  puertos abiertos de la máquina, y los servicios que corren por los puertos.

```bash
nmap -p- --open -T5 10.10.11.11 -n -Pn -v
```

![picture](/assets/images/hackthebox/board2.png)

Introducimos el dominio *board.htb* al **/etc/hosts**, para que nos resuelva el nombre de dominio.

En busca de posible información filtrada en la página web asociada, no encontramos nada.

![picture](/assets/images/hackthebox/board3.png)

Probamos a hacer **Fuzzing** con **Gobuster**, pero tampoco encontramos nada interesante, ya que a la mayoría de rutas no tenemos acceso, y los archivos **PHP** listados, los podemos encontrar clicando dentro de la web...

```bash
gobuster dir -u http://board.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .txt,.php,.html,.sh,.png,.jpg,.jpeg -b 402,404,403,500,503 
```

![picture](/assets/images/hackthebox/board4.png)

A continuación, probamos a buscar subdominios con la herramienta **FFUF**.

```bash
ffuf -c -u hhtp://board.htb/ -H "Host: FUZZ.board.htb" -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -fw 6243
```

![picture](/assets/images/hackthebox/board5.png)

Encontramos el subdominio *CRM*, así que lo añadimos al **/etc/hosts**.

Y una vez dentro del subdominio, encontramos un panel de autenticación con ***Dolibar***, versión ***17.0.0***.

![picture](/assets/images/hackthebox/board6.png)

---

## **Explotación**

Probando a entrar con usuarios y contraseñas comunes, obtenemos acceso con *USERNAME*: **admin**, y *PASSWORD*: **admin**.

![picture](/assets/images/hackthebox/board7.png)

Buscando en internet, encontramos la vulnerabilidad **CVE-2023-30253**, para versiones **17.0.0** e inferiores, con la que podremos obtener una **Reverse Shell**. [Enlace de GitHub](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253)

Nos descargamos el script escrito en **Pyhton**, y una vez lo ejecutemos, nos indica como tenemos que usarlo.

![picture](/assets/images/hackthebox/board8.png)

Añadimos los datos que nos pide, y antes de ejecutarlo, debemos ponernos en escucha con un **Listener** por el puerto que queremos que nos envíe la **Reverse Shell**, una vez echo esto, ejecutamos el script.

```bash
nc -nlvp 4444
```
En la imagen:
1. *Ejecución del script*
2. *Listener*

![picture](/assets/images/hackthebox/board9.png)

Y obtenemos acceso a la máquina con el ususario **www-data**.

----

## **Escalada de privilegios**

Una vez dentro de la máquina, después de buscar archivos que nos puedan ayudar a escalar privilegios, encontramos en el archivo de configuración "conf.php" una contraseña.

![picture](/assets/images/hackthebox/board10.png)

Viendo el archivo **/etc/passwd**, comprobamos que hay un usuario llamado "*larissa*", al haber encontrado antes el puerto SSH (22) abierto, vamos a intentar entrar con el usuario y la contraseña encontrada.

Obtenemos acceso.

![picture](/assets/images/hackthebox/board11.png)

En la carpeta personal de *larissa*, encontramos la primera **flag**.

![picture](/assets/images/hackthebox/board12.png)

Realizando una búsqueda de binarios que contengan el bit de **SUID**, nos encontramos con ***Enlightenment***, que es un entorno de escritorio, y gestor de ventanas, conocido por su alto nivel de personalización y sus efectos visuales avanzados.

![picture](/assets/images/hackthebox/board13.png)

Comprobamos la versión, para ver si es vulnerable.

![picture](/assets/images/hackthebox/board14.png)

Y en internet encontramos **[CVE-2022-37706](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit)**.

![picture](/assets/images/hackthebox/board15.png)

Nos descargamos el script, le damos *permisos de ejecución*, y una vez lo ejecutemos, obtendremos una *shell* con el ususario **root**, en su carpeta personal encontraremos la segunda **flag**.

![picture](/assets/images/hackthebox/board16.png)

![picture](/assets/images/hackthebox/board17.png)
