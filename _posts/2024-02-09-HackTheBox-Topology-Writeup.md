---
title: Topology - Hack The Box
description: >-
  Topology se trata de una máquina basada en el Sistema Operativo Linux, donde a través de Local File Inclusion (LFI) via LaTeX Injection se obtiene un archivo que contiene credenciales que luego se utilizan para entablar conexión por SSH. Finalmente, para la escalada de privilegios, se debe crear un archivo con extensión .plt para el programa gnuplot, con el fin de convertir la BASH en permisos SUID.
author: Mariano Alfonso
date: 2024-02-09
categories: [HackTheBox, Easy, CFTs]
tags: [Pentesting, Linux, LaTeX, LaTeX Injection, Local FIle Inclusion, LFI, SSH]
image:
  path: /assets/img/HTB/writeup-topology/topology.png
  alt: Máquina Topology de la Plataforma Hack The Box.
---

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Topology.pdf)

---
## Introducción 📄
En el presente Write Up explicare los pasos para resolver la máquina <a href="https://app.hackthebox.com/machines/Topology" style="color:#2AAFE8"><strong>**Topology**</strong></a> de la plataforma [HackTheBox](https://hackthebox.com).

## Alcance 🎯
El alcance de esta máquina fue definida como la siguiente.

<div style="width: 100%;">
  <table style="width: 100%; table-layout: fixed; border-collapse: collapse;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 1</span>: Scope.
    </caption>
    <tr>
      <th style="text-align: center; white-space: nowrap; padding: 10px; width: 50%;">Servidor Web / Direcciónes IPs / Hosts / URLs</th>
      <th style="text-align: center; padding: 10px; width: 35%;">Descripción</th>
      <th style="text-align: center; padding: 10px; width: 20%;">Subdominios</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">10.129.16.121</td>
      <td style="text-align: left; word-wrap: break-word; padding: 10px;">Dirección IP de la máquina Topology</td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Todos</td>
    </tr>
  </table>
</div>

## Metodologia Aplicada 📦​​📚
* Enfoque de prueba: En el proceso de pruebas de seguridad, se optó por un enfoque gray-box, lo que significó que se tenía un nivel de acceso parcial a la infraestructura y el sistema objetivo.
* Las etapas aplicadas para esta auditoria fueron las siguientes:

![](/assets/img/HTB/writeup-topology/Etapas-pentest.png) _**Figure 1**: Etapas del Pentest._

## Reconocimiento - Enumeración 🔍
### Uso de la Herramienta Nmap 👁️
Primeramente realizamos un escaneo con ayuda de la herramienta nmap en búsqueda de puertos abiertos.

```bash
# Primer escaneo.
--------------------------------------------------------------
nmap -p- --open -sV --min-rate 5000 10.129.16.121
```

<div style="width: 100%;">
  <table style="width: 100%; table-layout: fixed; border-collapse: collapse;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 2</span>: Definición de Parámetros de Nmap Utilizados en el Primer Escaneo.
    </caption>
    <tr>
      <th style="text-align: center; white-space: nowrap; padding: 10px; width: 20%;">Parámetro</th>
      <th style="text-align: center; padding: 10px; width: 80%;">Descripción</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-p-</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Escanea los 65535 puertos.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>--open</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Muestra solo los puertos abiertos.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-sV</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Determina las versiones de los servicios que se ejecutan en los puertos encontrados.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>--min-rate 5000</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Establece la velocidad mínima de envío de paquetes a 5000 paquetes por segundo.</td>
    </tr>
  </table>
</div>

El resultado que nos arrojó este primer escaneo fue que la máquina tiene el puerto **22** que pertenece al servicio *SSH* y el puerto **80** que pertenece al protocolo *HTTP*.

![](/assets/img/HTB/writeup-topology/nmap_1.png) _**Figure 2**: Resultado del Primer Escaneo con Nmap._

Se procedió a realizar otro escaneo con los scripts default de nmap, también especificando la versión nuevamente.

```bash
# Segundo escaneo.
--------------------------------------------------------------
nmap -sC -sV -p22,80 10.129.16.121
```

<div style="width: 100%;">
  <table style="width: 100%; table-layout: fixed; border-collapse: collapse;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 3</span>: Definición de Parámetros de Nmap Utilizados en el Segundo Escaneo.
    </caption>
    <tr>
      <th style="text-align: center; white-space: nowrap; padding: 10px; width: 20%;">Parámetro</th>
      <th style="text-align: center; padding: 10px; width: 80%;">Descripción</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-sC</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Realiza un escaneo con los scripts por defecto.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-sV</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Determina las versiones de los servicios que se ejecutan en los puertos encontrados.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-p</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Especifica los puertos que se escanearán.</td>
    </tr>
  </table>
</div>


Lo único interesante que obtenemos es el título de la página web **Miskatonic University \| Topology Group**.

![](/assets/img/HTB/writeup-topology/nmap_2.png) _**Figure 3**: Resultado del Segundo Escaneo con Nmap._

### Aplicación Web 🌐​
Luego del segundo escaneo, se ingresó a la aplicación web, donde en el código fuente se encontró un subdominio **http://latex.topology.htb/equation.php**

![](/assets/img/HTB/writeup-topology/código-fuente.png) _**Figure 4**: Observando el Código Fuente._

Antes de ingresar al subdominio, se agregó el mismo al archivo **/etc/hosts**

```bash
# Agregando subdominio al archivo /etc/hosts.
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.16.121 latex.topology.htb
```
{: file='/etc/hosts'}

## Análisis de Vulnerabilidades ​🔬
Al ingresar al subdominio, vemos que cosiste en una generador de ecuaciones mediante sintaxis en LaTeX.

![](/assets/img/HTB/writeup-topology/Latex-equation.png) _**Figure 5**: Identificando Aplicativo._

Realizamos un **PoC**, para ver más en detalle la funcionalidad de la aplicación web.

![](/assets/img/HTB/writeup-topology/PoC_1.png) _**Figure 6**: Analizando el Corpotamiento del Aplicativo._

Una vez que enviamos el comando de LaTeX para generar la ecuación, nos envia a una ruta con la ecuación generada en formato de imagen.

![](/assets/img/HTB/writeup-topology/PoC_resultado.png) _**Figure 7**: Ecuación Generada._

Muy interesante el funcionamiento de la aplicación y de como se genera la ecuación.

### Local File Inlcusion (LFI) via LaTeX Injection 📄💉​
Al entender como funciona la aplicación, se me ocurrió realizar una prueba que consiste en incluir el archivo **/etc/passwd**, ingresando código de LaTeX arbitrario.

Para realizar esta prueba se utilizó en siguiente recurso [https://book.hacktricks.xyz/v/es/pentesting-web/formula-csv-doc-latex-ghostscript-injection](https://book.hacktricks.xyz/v/es/pentesting-web/formula-csv-doc-latex-ghostscript-injection) de [HackTricks](https://book.hacktricks.xyz)

Realicé una prueba para incluir el archivo **/etc/passwd** con el siguiente comando.

```latex
# Comando utilizado.
--------------------------------------------------------------
$input/etc/passwd$
```

![](/assets/img/HTB/writeup-topology/Latex-Injection.png) _**Figure 8**: Comando Ingresado._

Como resultado la apliación detecta que se están ingresando comandos arbitrarios.

![](/assets/img/HTB/writeup-topology/Latex-Injection-Resultado.png) _**Figure 9**: Imagen Generada._

Después de varios intentos, el comando que me funcionó para obtener lectura del archivo **/etc/passwd**, fue el siguiente:

![](/assets/img/HTB/writeup-topology/Latex-Injection_2.png) _**Figure 10**: Comando Útil._

Como resultado genera la imagen del archivo **/etc/passwd**

![](/assets/img/HTB/writeup-topology/Latex-Injection_2_Resultado.png) _**Figure 11**: Archivo /etc/passwd_

## Explotación 💣​
### Enumeración de Archivos del Sistema 📌
Al obtener lectura del archivo, se confirma que la aplicación web es vulnerable a **Local File Inclusion** via **LaTeX Injection**.

Utilizaremos esta vulnerabilidad para conseguir posibles datos que nos interesen o sean útiles. 

Como la app web tiene un servidor web apache, procedemos a leer el archivo de configuración predeterinado del servidor web, que se encuentra en esta ruta **/etc/apache2/sites-available/000-default.conf**

![](/assets/img/HTB/writeup-topology/comando-archivo-config.png) _**Figure 12**: Comando para Obtener Lectura del Archivo de Configuración._

Obtenemos el resultado, donde se aprecia la ruta de la landing page de la universidad y las demas rutas de los aplicativos.

![](/assets/img/HTB/writeup-topology/archivo-cofig_1.png) _**Figure 13**: Aplicativos con sus Respectivas Rutas._

Además se encuentran otros aplicativos con sus respectivas rutas.

![](/assets/img/HTB/writeup-topology/otro-aplicativo.png) _**Figure 14**: Nuevos Aplicativos con sus Respectivas Rutas._

Para ingresar a los subdominios encontrados, debemos nuevamente agregarlos al archivo **/etc/hosts**

```bash
# Agregando subdominios al archivo /etc/hosts.
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.16.121 latex.topology.htb dev.topology.htb stats.topology.htb
```
{: file='/etc/hosts'}

Lo único interesante es el subdominio **dev.topology.htb**, que tiene un formulario de login, pero no podemos ingresar por falta de credenciales.

![](/assets/img/HTB/writeup-topology/dev-form-login.png) _**Figure 15**: Formulario de Inicio de Sesión._

Nuevamente utilizaremos el LFI para acceder al archivo **.htpasswd**, donde se supone que se guardan las credenciales de autenticación del servidor HTTP Apache.

![](/assets/img/HTB/writeup-topology/lfi-htpasswd.png) _**Figure 16**: LFI para el Archivo .htpasswd_

Como resultado obtenemos un usuario y una contraseña cifrada, aparentemente con el **algoritmo** de **hashing** que usa **Apache** por defecto, que es **APR1**.

![](/assets/img/HTB/writeup-topology/user-passwd.png) _**Figure 17**: Usuario y Contraseña Cifrada._

### Uso de la Herramienta John the Ripper 🕵️‍♂️
Con el uso de la herramienta [John the Ripper](https://www.kali.org/tools/john) desciframos la contraseña.

![](/assets/img/HTB/writeup-topology/passwd-deshasheda.png) _**Figure 18**: Contraseña Descifrada._

La contraseña obtenida es **calculus20** y la utilizaremos en el formulario de login junto al usuario **vdaisley** que obtuvimos anteriormente.

![](/assets/img/HTB/writeup-topology/creds-form-login.png) _**Figure 19**: Iniciando Sesión._

Al entrar no encontramos nada interesante, solo un software desarrollado por el personal de la **Universidad Miskatonic**.

![](/assets/img/HTB/writeup-topology/dev-pag.png) _**Figure 20**: Software Desarrollado por el Personal de la Universidad._

### Acceso al Sistema via SSH 🖥️​
Al haber escaneado los puertos anteriormente y obtener información de que el puerto 22 que corresponde al servicio SSH esta abierto, procedemos a conectarnos por dicho servicio, utilizando las credenciales obtenidas.

Al conectarnos exitosamente, podemos leer la <span style="color:blue"> **flag** </span> del **user**. 

![](/assets/img/HTB/writeup-topology/user-flag.png) _**Figure 21**: Flag del user._

### Listamiento de Directorios Interesantes 🤔
Una vez dentro, vemos en el directorio **opt** que se encuentra un directorio llamado **gnuplot**, el cual tiene permisos de **escritura** y **ejecución**, y cuyo propietario es **root**.

![](/assets/img/HTB/writeup-topology/gnuplot.png) _**Figure 22**: Directorio opt._

Investigando un poco encontre que **gnuplot** es un programa de interfaz de línea de comandos para generar gráficas de dos y tres dimensiones de funciones, datos y ajustes de datos.

## Escalada de Privilegios 👨‍💻​
### Uso de la Herramienta pspy ⚙️​
Para seguir enumernado la máquina victima utilizaré [pspy](https://github.com/DominicBreuker/pspy), que es una herramienta de monitoreo de procesos para sistemas Linux. Esta herramienta permitira enumerar procesos de la máquina victima.

Antes debo saber cuál es la arquitectura y la cantidad de bits del sistema Linux de la máquina víctima, para poder descargar el ejecutable de pspy para la arquitectura correspondiente.

Para eso ejecuto el comando **uname -m**, en el cual obtengo que la arquitectura de la máquina victima es de 64 bits.

![](/assets/img/HTB/writeup-topology/64-bits.png) _**Figure 23**: Arquitectura de 64 Bits._

Una vez teniendo este dato me descargo el ejecutable de pspy y creo en el servidor python en el puerto 80, para luego desde la máquina victima realizar una petición wget y pasarme el archivo de la herramienta.

Realizamos una petición wget en la máquina victima.

![](/assets/img/HTB/writeup-topology/wget.png) _**Figure 24**: Petición wget._

Le asignamos permisos de ejecución al ejecutable de pspy.

![](/assets/img/HTB/writeup-topology/chmod.png) _**Figure 25**: Asignando Permisos de Ejecución._
 
Una vez asignados los permisos de ejecución, procedemos a ejecutarlo y observamos que el usuario con **UID=0**, o sea el usuario **root**, ejecuta el comando **find** sobre el directorio **/opt/gnuplot**, donde busca todos los archivos con extensión **.plt** (que es la extensión que corresponde al programa gnuplot) y luego los ejecuta.

Luego ejecuta una serie de scripts y por ultimo ejecuta un archivo **networkplot.plt**

![](/assets/img/HTB/writeup-topology/resultado-pspy.png) _**Figure 26**: Procesos._

Nos damos cuenta en el directorio **/opt/gnuplot** tenemos permisos de escritura pero no de listamiento.

![](/assets/img/HTB/writeup-topology/permisos.png) _**Figure 27**: Permisos._

### Creación del Exploit 👾​
Lo que se me ocurre es crear un archivo asignando permisos **SUID** para convertir la **BASH** del sistema, de modo que luego root ejecute el script habilitando el **Bit SUID** y enlace una BASH con altos privilegios.

```bash
# Exploit.
--------------------------------------------------------------
nano root.plt
          
system "chmod u+s /bin/bash"
```

Una vez creado el archivo, ejecutamos de vuelta pspy para saber cuando root ejecuto el archivo.

![](/assets/img/HTB/writeup-topology/pspy-script.png) _**Figure 28**: Archivo Ejecutado._

Para confirmar hacemos un **ls -la** de la bash y vemos que tiene el Bit SUID activado.

![](/assets/img/HTB/writeup-topology/ls-l-bash.png) _**Figure 29**: Bit SUID Activado._

Simplemente ahora hacemos **bash -p** y conseguimos escalar privilegios y obtener la **flag** de **root**.

![](/assets/img/HTB/writeup-topology/whoami-flag.png) _**Figure 30**: Flag de root._

## Conclusión Final 💬
Esta máquina resultó muy entretenida, ideal para aquellos que recién comienzan en el pentesting resolviendo máquinas. La verdad es que fue bastante sencilla, en mi caso nunca había explorado ni explotado un Local File Inclusion (LFI) a través LaTeX Injection. Si no fuera por eso, la habría terminado antes. Después de eso, la parte de explotación y priv-esc no me dio problemas.

## Apéndice I Links de Referencia 📎​
### Herramientas Utilizadas en la Auditoria 🛠️​
* [Nmap:](https://nmap.org) [https://nmap.org](https://nmap.org) - [https://www.kali.org/tools/nmap](https://www.kali.org/tools/nmap) ---> Uso de nmap para el escaneo de puertos.
* [John the Ripper:](https://www.openwall.com/john) [https://www.openwall.com/john](https://www.openwall.com/john) - [https://www.kali.org/tools/john](https://www.kali.org/tools/john)[https://github.com/openwall/john](https://github.com/openwall/john) ---> Uso de John the Ripper para descifrar contraseña.
* [pspy - unprivileged Linux process snooping:](https://github.com/DominicBreuker/pspy) [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy) ---> Uso de pspy para monitorear procesos.

### Documentación ​📰​​
* [HackTricks: LaTeX Injection](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection) [https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection) 
* [Gnuplot Privilege Escalation:](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation) [https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation)
