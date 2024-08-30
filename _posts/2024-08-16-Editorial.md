---
title: Editorial
date: 2024-08-16
categories: [WRITEUPS, HackTheBox]
tags: [git, ssrf]  # TAG names should always be lowercase
image: /assets/images/hackthebox/htblogo.png
---

![picture](/assets/images/hackthebox/editorial1.png)

Buenas, en esta máquina explotaremos un **SSRF** para obtener acceso a la máquina, y para la escalada de privilegios tendremos que movernos por un repositorio local con Git para encontrar credenciales.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos será un escaneo con *NMAP* para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.11.20 -n -Pn -v
```

![picture](/assets/images/hackthebox/editorial2.png)

A continuación, añadiremos el dominio que nos sale en el navegador al */etc/hosts* para que nos resuelva el nombre de dominio, y una vez hecho nos iremos al navegador para ver de que se trata la página web.

![picture](/assets/images/hackthebox/editorial3.png)

![picture](/assets/images/hackthebox/editorial4.png)

Realizando **Fuzzing** no se ha encontrado nada, en el apartado de "*Publish with us*", se han estado probando varias cosas sin resultado, vamos a interceptar la petición con **Burpsuite** para ver como la envía, y a conmtinuación la mandamos al repeater.

![picture](/assets/images/hackthebox/editorial5.png)

![picture](/assets/images/hackthebox/editorial6.png)

Al enviar la petición, y ser un campo que pide **URLs**, la respuesta del servidor siempre es la misma, ponga "*prueba*" o cualquier otra *URL*. Eso puede darse porque no analiza las peticiones de las URLs solicitadas, o porque solo devuelve un contenido genérico.

![picture](/assets/images/hackthebox/editorial7.png)

---

## **Explotación**

Lo que podríamos probar es si es vulnerable a **SSRF** (*Server Side Request Forgery*) ya que al poder controlar la URL, si podemos conseguir que se envíe peticiones a su localhost (127.0.0.1), podríamos comprobar si hay algún servicio corriendo en algunos de sus puertos.

- PASOS
    - Creamos un archivo llamado **puertos.txt** con todos los puertos existentes.

        ![picture](/assets/images/hackthebox/editorial8.png)

    - Creamos un archivo llamado **requests.txt** con la petición copiada de **Burpsuite**, modificando la URL de la petición, añadiéndole ***:FUZZ***, que será donde se probarán todos los puertos, como se puede ver en la imagen.

        ![picture](/assets/images/hackthebox/editorial9.png)

    - Realizamos **Fuzzing** con **FFUF**, yo oculto las peticiones cuyo tamaño es 61. (Una vez lo ejecutas sin ocultar nada, saldrán las miles de peticiones de los puertos cerrados con el mismo tamaño, en mi caso el tamaño de las peticiones es de 61, por eso lo bloqueo.)

        ```bash
        ffuf -c -u http://editorial.htb/upload-cover -X POST -request request.txt -w puertos.txt:FUZZ -fs 61
        ```

        ![picture](/assets/images/hackthebox/editorial10.png)

Y obtenemos el puerto **5000** con un tamaño distinto al de los puertos cerrados, a continuación, enviaremos una petición al localhost añadiéndole el puerto desde **Burpsuite**, para comprobar si la respuesta cambia.

![picture](/assets/images/hackthebox/editorial11.png)

Vemos que cambia, la copiamos, y nos la descargamos con **wget**.

![picture](/assets/images/hackthebox/editorial12.png)

Vemos el contenido y es un **json**, a continuación, enviaremos otra petición con el *endpoint* de los *authors*.

![picture](/assets/images/hackthebox/editorial13.png){: w="600" h="300" }

Copiamos la respuesta, y nos la volvemos a descargar.

![picture](/assets/images/hackthebox/editorial14.png)

![picture](/assets/images/hackthebox/editorial15.png)

Viendo el contenido encontramos unas credenciales.

![picture](/assets/images/hackthebox/editorial16.png)

Probamos a entrar mediante **SSH**, y obtenemos resultado.

![picture](/assets/images/hackthebox/editorial17.png){: w="600" h="300" }

En el directorio personal encontramos la primera **flag** de usuario.

![picture](/assets/images/hackthebox/editorial18.png)

---

## **Escalada de privilegios**

Dentro del directorio **/home** encontramos otro usuario llamado **prod**.

![picture](/assets/images/hackthebox/editorial19.png)

Dentro del directorio personal del ususario **dev**, encontramos un repositorio local de ***git***.

![picture](/assets/images/hackthebox/editorial20.png)

Derntro del repositorio, si ejecutamos un **git log**, podemos ver todos los **commits** que se han hecho.

![picture](/assets/images/hackthebox/editorial21.png)

Mirando uno a uno he encontrado credenciales en el tercero.

![picture](/assets/images/hackthebox/editorial22.png)

Probando a migrar al ussuario **prod** con las credenciales encontradas, obtenemos acceso.

![picture](/assets/images/hackthebox/editorial23.png)

Listando los permisos de *SUDO* de **prod**, vemos que tenemos permisos en el siguiente script.

![picture](/assets/images/hackthebox/editorial25.png)

Mostramos el contenido del script.

![picture](/assets/images/hackthebox/editorial26.png)

Y buscando por internet he encontrado la siguiente vulnerabilidad por la cual podemos escalar los máximos privilegios [**CVE-2022-24439**](https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858).

Ejecutando el siguiente comando.

```bash
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py "ext::sh -c cat% /root/root.txt% >% /tmp/root"
```

![picture](/assets/images/hackthebox/editorial27.png)

![picture](/assets/images/hackthebox/editorial28.png){: w="600" h="300" }

