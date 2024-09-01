---
title: Papafrita
date: 2024-09-01
categories: [WRITEUPS, TheHackersLabs]
tags: []  # TAG names should always be lowercase
image: /assets/images/thehackerslabs/thllogo.png

---

En este ***Writeup***, encontraremos una cadena codificada en **Brainfuck**, obtendremos acceso a la máquina por el texto encontrado, y escalaremos privilegios por permisos de **SUDO** en el binario **node**. (Muy sencilla)

---

En este ***Writeup*** se engloban las siguientes fases:
- **[Reconocimiento](#reconocimiento)**
- **[Explotación](#explotación)**
- **[Escalada de privilegios](#escalada-de-privilegios)**

---

## **Reconocimiento**

Una vez conozcamos la dirección **IP** de la máquina, realizaremos un escaneo de puertos con **NMAP**.

```bash
nmap -p- --open -T5 192.168.1.163 -n -Pn -v
```

![picture](/assets/images/thehackerslabs/papa1.png){: w="600" h="300" }

A continuación, nos dirigiremos al navegador para ver el contenido de la página web asociada a la dirección **IP**.

Encontramos una página por defecto de **Apache2**.

![picture](/assets/images/thehackerslabs/papa2.png)

Si vemos la fuente de la página, encontramos unos caracteres de lo que parece ser código **Brainfuck**.

![picture](/assets/images/thehackerslabs/papa3.png)

Si nos vamos a  [**Brainfuck - Translator**](https://md5decrypt.net/en/Brainfuck-translator/), podemos observar que nos devuelve texto claro.

![picture](/assets/images/thehackerslabs/papa4.png)

---

## **Explotación**

Probando a realizar **Fuzzing** no he encontrado nada, he probado a conectarme con el usuario **abuela**, el cual es una parte del código descifrado de **Brainfuck**, y de contraseña el texto entero, y ha funcionado.

![picture](/assets/images/thehackerslabs/papa5.png){: w="600" h="300" }

Dentro del directorio del usuario **abuela**, se encuentra la **flag** de usuario, pero no tenemos permisos de lectura ya que pertenece a **root**.

---

## **Escalada de privilegios

Listando los permisos de **SUDO** del usuario, no necesitamos contraseña para el binario **node**.

![picture](/assets/images/thehackerslabs/papa6.png)

Si nos vamos a la página de [**GTFOBins - node**](https://gtfobins.github.io/gtfobins/node/), nos dice como escalar privilegios.

```bash
sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

Lo ejecutamos, y obtenemos la shell como **root**.

![picture](/assets/images/thehackerslabs/papa7.png)

Y ya podemos leer la **flag** de usuario, en el directorio de **abuela**, y la **flag** de **root**.

![picture](/assets/images/thehackerslabs/papa8.png)

