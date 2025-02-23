---
title: TwoMillion - Hack The Box
description: >-
  ¡El especial dos millones! La plataforma HackTheBox presenta una máquina Linux de nivel fácil con el antiguo diseño de la plataforma, incluyendo el código de invitación. Para conseguir acceso a la máquina, primero debemos completar el HTB Challenge al registrarnos. Luego de completarlo y crear la cuenta, la usaremos para enumerar la API y obtener permisos como admin, además de realizar una CMD injection en la generación de la VPN. Posteriormente, aprovecharemos una Reverse Shell mediante la enumeración de archivos de variables de entorno (el sistema tiene un archivo que suele contener credenciales de bases de datos o información de conexión a servicios externos). Por último, para obtener acceso de root, debemos explotar la vulnerabilidad en el kernel CVE-2023-0386.
author: Mariano Alfonso
date: 2023-07-06
categories: [HackTheBox, Easy, CFTs]
tags: [Pentesting, Linux, CVE, Api, JavaScript, OverlayFS Vulnerability]
image:
  path: /assets/img/HTB/writeup-twomillion/TwoMillion.png
  alt: Máquina TwoMillion de la Plataforma Hack The Box.
---

---

## Introducción 📄
El presente Write Up explica los pasos para resolver la máquina <span style="color:orange"> **TwoMillion** </span> de la plataforma [HackTheBox](https://hackthebox.com).

## Reconocimiento 🔍
### Herramienta nmap 👁️
Lazamos la herramienta nmap para averiguar los puertos y servicios abiertos.

```bash
# Primer lanzamiento de la herramienta en nmap
--------------------------------------------------------------
nmap -p- --open -v 10.10.11.221 
```

![](/assets/img/HTB/writeup-twomillion/Nmap1.png) _**Figure 1**: Puerto SSH y HTTP Abiertos._

Obtenemos dos puertos abiertos, el puerto <span style="color:blue"> 22 </span> que pertence al protocolo **ssh** y el puerto <span style="color:red"> 80 </span> que pertence al protocolo **http**.

Vamos a tirar nmap otra vez, pero ahora vamos a especificar la versión del servicio.

```bash
# Segundo lanzamiento de la herramienta en nmap
--------------------------------------------------------------
nmap -p 22,80 -sC -sV 10.10.11.221 -oN Information
```

![](/assets/img/HTB/writeup-twomillion/Nmap2.png) _**Figure 2**: Resultado del Segundo Escaneo con Nmap._

Vemos que en el puerto 80 intenta redireccionar la conexión al dominio **2million.htb**, pero no tiene éxito.

## Enumeración 📌
Vamos a tratar de entrar al dominio <span style="color:violet"> 2million.htb </span>, para eso hay que modificar el archivo de **/etc/hosts**

```bash
# Modificamos el archivo que hace el redireccionamiento y agregamos la IP de la máquina con el dominio que obtuvimos con el segundo escaneo
--------------------------------------------------------------
nano /etc/hosts

10.10.11.221 2million.htb
```
{: file='/etc/hosts'}

Ahora si queremos acceder al sitio web, podemos hacerlo.

### Investigando el sitio web 🕵️‍♂️
Podemos ver la antigua plataforma de [HackTheBox](https://hackthebox.com).

![](/assets/img/HTB/writeup-twomillion/2million.htb.png) _**Figure 3**: Antigua Plataforma de HackTheBox._

Podemos ver el antiguo sitio de HackTheBox y los apartados de los mismo **abount, FAQ, join, commercial, labs, hall of fame ,login**.
Nos interesa loguearnos, pero tenemos un problema y es que no tenemos permiso para eso, tampoco no tenemos una cuenta, pero si vamos a <span style="color:green"> join </span> hay directorio <span style="color:pink"> /invite </span>, vamos a mirar por ahi.

![](/assets/img/HTB/writeup-twomillion/ir_a_join.png) _**Figure 4**: Apartado join._

![](/assets/img/HTB/writeup-twomillion/-invite.png) _**Figure 5**: Directorio invite._

Claro como es la plataforma antigua se trata del CTF que teaniamos que hacer todos para poder registrarnos, mirando el codigo HTML encuentro este directorio **/js/inviteapi.min.js**

![](/assets/img/HTB/writeup-twomillion/-js-inviteapi-min-js.png) _**Figure 6**: Directorio /js/inviteapi.min.js en el Código HTML._

Si miramos al directorio <span style="color:red"> 2million.htb/js/inviteapi.min.js </span>, encontramos algo interesante:

```bash
# Código ofuscado
--------------------------------------------------------------
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```
{: file='inviteapi.min.js'}

Es código <span style="color:yellow"> JS </span> y esta ofuscado, asi que usare la herramienta  [beautifier.io](https://beautifier.io)

Obtenemos este código legible:

```bash
# Código legible
--------------------------------------------------------------
function verifyInviteCode(code) {
            var formData = {
                "code": code
            };
            
            $.ajax({
                type: "POST",
                dataType: "json",
                data: formData,
                url: '/api/v1/invite/verify',
                success: function(response) {
                    console.log(response);
                },
                error: function(response) {
                    console.log(response);
                }
            });
        }
        
        function makeInviteCode() {
            $.ajax({
                type: "POST",
                dataType: "json",
                url: '/api/v1/invite/how/to/generate',
                success: function(response) {
                    console.log(response);
                },
                error: function(response) {
                    console.log(response);
                }
            });
        }
```
{: file='inviteapi.min.js'}

Vemos una función **makeInviteCode**, si lo ejecutamos en la consola, obtenemos información cifrada con **ROT13**.

![](/assets/img/HTB/writeup-twomillion/makeinvitecodepng.png) _**Figure 7**: Función makeInviteCode Ejecutada en Consola._

[rot13.com](https://rot13.com)

![](/assets/img/HTB/writeup-twomillion/decipher.png) _**Figure 8**: Decifrado del Contenido Cifrado en rot13._

Dice que hagamos un **POST** a esa ruta **/api/v1/invite/generate**, para eso usamos curl.

```bash
# Resultado
--------------------------------------------------------------
  {
    "0": 200,
    "success": 1,
    "data": {
    "code": "UEJKWk4tVlFSVFAtS1dXSEwtOU1NTEE=",
    "format": "encoded"
            }
  }
```

Veo que esta en **base64**, para verlo mejor, lo deciframos y lo ingresamos en **/invite**

![](/assets/img/HTB/writeup-twomillion/insert-code.png) _**Figure 9**: Insertando Código de Invitación._

Vemos que nos dirije a registrar un usuario.

![](/assets/img/HTB/writeup-twomillion/registercode.png) _**Figure 10**: Registro de Usuario._

Registramos un usuario y accedemos a **/home** , donde solo tenemos acceso a Access, en url seria **/home/access**
    
En Access podemos descargar y regenerar VPN, tal y como lo hacemos siempre en HackTheBox. 
    
Lo mejor sera interceptar la petición con <span style="color:orange"> BurpSuite </span> de Connection Pack y Regenerate.
    
El <span style="color:red"> GET </span> de **Conection Pack** dirije a **/api/v1/user/vpn/generate**

![](/assets/img/HTB/writeup-twomillion/generate.png) _**Figure 11**: Respuesta GET de Conection Pack._

El <span style="color:red"> GET </span> de **Regenerate** dirije a **/api/v1/user/vpn/regenerate**

![](/assets/img/HTB/writeup-twomillion/regenerate.png) _**Figure 12**: Respuesta GET de Regenerate._

## Api ⬜
Podemos probar haciendo una petición a **/api** para ver que sucede.

![](/assets/img/HTB/writeup-twomillion/-api.png) _**Figure 13**: Petición a la api._

Nos muestra la versión, <span style="color:pink"> v1 </span> y si le realizamos petición a **/api/v1**, obtenemos información de rutas y acciones de la api.

```bash
# Información sobre la API
--------------------------------------------------------------
{
        "v1":{ 
        "user": {
        "GET": {
                "/api/v1": "Route List",  
                "/api/v1/invite/how/to/generate": "Instructions on invite code generation", 
                "/api/v1/invite/generate": "Generate invite code",
                "/api/v1/invite/verify": "Verify invite code",
                "/api/v1/user/auth": "Check if user is authenticated",
                "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
                "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
                "/api/v1/user/vpn/download": "Download OVPN file"
                },

        "POST": {
                "/api/v1/user/register": "Register a new user",
                "/api/v1/user/login": "Login with existing user"
                }
                },
        "admin": {
        "GET": {
                "/api/v1/admin/auth": "Check if user is admin"
                },
        "POST": {
                "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
                },
        "PUT": {
                "/api/v1/admin/settings/update": "Update user settings"
                }
                }
                }
}
```
## Explotación 🔥
### Obteniendo acceso como admin 👨‍💼
Podemos averiguar si somos admin (spoiler no lo somos) haciendo un <span style="color:violet"> GET </span> a **/api/v1/admin/auth** , por lo tanto podemos jugar un poco cambiando la configuración haciendo un <span style="color:pink"> PUT </span> a **/api/v1/admin/settings/update**

Primero accedemos a la ruta y nos dice que el **Invalid content type**.

![](/assets/img/HTB/writeup-twomillion/put-update.png) _**Figure 14**: Tipo de Contenido no Válido._

Al parecer para representar datos estructurados se nececita <span style="color:blue"> json </span>, colocamos **Content-Type: application/json**

![](/assets/img/HTB/writeup-twomillion/content-type.png) _**Figure 15**: Indicando que el Contenido del Cuerpo de la Solicitud._

Nos dice que indiquemos como parametro un email, usemos el que usamos para registrar nuestra cuenta.

Nos pide un nuevo parametro: **"is_admin"** ,  hay que indicar 0 o 1 si indicas otro parametro, va a decirte que indiques 0 o 1

![](/assets/img/HTB/writeup-twomillion/message-is_admin.png) _**Figure 16**: Parámetro permitido is_admin_

![](/assets/img/HTB/writeup-twomillion/is_admin_1.png) _**Figure 17**: Indicando Valor 1._

Al parecer funciona.

![](/assets/img/HTB/writeup-twomillion/login-en-burpsuite.png) _**Figure 18**: Datos Obtenidos._

Ahora verifiquemos si somos admin, haciendo <span style="color:orange"> GET </span> a **/api/v1/admin/auth**

![](/assets/img/HTB/writeup-twomillion/true-auth.png) _**Figure 18**: Comprobando el User._

Nos reporta <span style="color:orange"> true </span>, por lo tanto somos admin.

Ahora probemos con un **POST** a **/api/v1/admin/vpn/generate**

![](/assets/img/HTB/writeup-twomillion/post-vpn.png) _**Figure 19**: Petición Post._

Nos pide de vuelta el **username** para generar la VPN, tal y como lo hace HackTheBox.

![](/assets/img/HTB/writeup-twomillion/username-2twomillion.png) _**Figure 20**: Indicando el Username._

### Shell como www-data ⌨️

Probablemente este campo de username sea vulnerable, intentemos inyetando comandos, para eso hay que agregar una **;** para dividirlo con el usuario, ingresamos el comando **whoami**, despues hay que un **#** para comentar luego.

![](/assets/img/HTB/writeup-twomillion/inyect-code-malicious.png) _**Figure 21**: Inyectando Comandos._

Vamos que funciona, ahora lo bueno es poder hacer una Reverse Shell.

```bash
# Nos ponemos en escucha por netcat desde nuestra terminal 
--------------------------------------------------------------
nc -nlvp 8002
```

![](/assets/img/HTB/writeup-twomillion/rshell.png) _**Figure 22**: Inyectando Código para Obtener las Reverse Shell._

Listo, antes de continuar hacemos un tratamiento a la **tty**.

```bash
# Tratamiento a la tty 
--------------------------------------------------------------
script /dev/null -c bash
Script started, output log file is '/dev/null'
^Z
[1]+  Stopped                 nc -nlvp 443
stty raw -echo;fg
              reset xterm
export TERM=xterm
export SHELL=bash
```

Si listamos detalladamente el contenido de un directorio, encontramos un archivo <span style="color:red"> .env </span>, que suele contener variables de entorno como claves de API, credenciales de base de datos, contraseñas, tokens de acceso, rutas de archivos.

![](/assets/img/HTB/writeup-twomillion/-env.png) _**Figure 23**: Listado del Directorio._

Entonces miremos por ahi.

```bash
# Visualizando en archivo .env
--------------------------------------------------------------
cat .env
```

![](/assets/img/HTB/writeup-twomillion/user-pass.png) _**Figure 24**: Contenido del Archivo .env_

Vemos un user y una password, lo usaremos para conectarnos por **ssh**.

```bash
# Conexición por ssh usando usuario y password que encontramos en el archivo .env
--------------------------------------------------------------
shh admin@10.10.11.221

SuperDuperPass123
```

Listamos contenido del directorio y encontramos la **flag** de usuario.

![](/assets/img/HTB/writeup-twomillion/user-flag.png) _**Figure 25**: Flag del User._

## Escalada de privilegios 👩‍💻

Si buscamos directorios desde raiz, encontramos el directorio **/var**, miremos ahi.

![](/assets/img/HTB/writeup-twomillion/admin-var.png) _**Figure 26**: Contenido del Archivo /var_

Si entramos vemos otro directorio interesante **/mail**.

![](/assets/img/HTB/writeup-twomillion/admin-email.png) _**Figure 27**: Listado del Directorio /var y Contenido del Archivo admin._

Vemos un correo que nos dice.

```text
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```
{: file='admin'}

En resumen nos dicen que si podemos actualizar el Sitema Operativo del servidor por hay nuevos CVE que tienen que ver con el Kernel de Linux, habla de un **CVE OverlayFS / FUSE**, investigando un poco,encontre información sobre:

- [Informacion de OverlayFS vulnerability DATADOG](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/)

- [CVE-2023-0386](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-0386)

No me pondre a explicar en que consiste la vuln pero si quieres profundizar el tema, te dejo de vuelta el enlace del **blog** de **DATADOG**, aqui que explica detalladamente la vuln [Informacion de OverlayFS vulnerability DATADOG](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/), pero dice que los sistemas de **Versión del Kernel inferior a 6.2 son vulnerables**, con las distribuciones de **Ubuntu, Debian, Amazon Linux, Red Hat**

Entonces comprobemos que versión y que distribución tiene la máquina victima.

Comando **uname -a** para proporcinar info del Kernel.

![](/assets/img/HTB/writeup-twomillion/uname-a.png) _**Figure 28**: Información del Kernel._

Vemos que tiene una versión **5.15.70**, es inferior a la versión de 6.2, bien miremos ahora cual es la distro de la máquina.

Comando **lsb_release -a** para  mostrar info de la distribución de Linux.

![](/assets/img/HTB/writeup-twomillion/lsb-release-a.png) _**Figure 29**: Información de la Distribución de Linux._

La máquina tiene distribución **Ubuntu**, entonces es vulnerable.

Buscando como explotar la vuln, encontre este repositorio: [xkaneiki / CVE-2023-0386](https://github.com/xkaneiki/CVE-2023-0386)

Descargamos el repositorio, lo descomprimimos y seguimos los pasos.

![](/assets/img/HTB/writeup-twomillion/7z.png) _**Figure 30**: Descomprimiendo el .zip_

![](/assets/img/HTB/writeup-twomillion/ls-cve.png) _**Figure 31**: Contenido del Archivo Comprimido._

```bash
# Ejecutamos el comando make all
--------------------------------------------------------------
make all
```

Nos genera **tres binarios**, hay que pasarlos a la máquina victima, iniciamos un server en python3 por el puerto **80**.

![](/assets/img/HTB/writeup-twomillion/python3.png) _**Figure 32**: Iniciando Servidor con Python._

Damos permisos de ejecución a los archivos y luego ejecutamos lo siguiente **./fuse ./ovlcap/lower ./gc**

![](/assets/img/HTB/writeup-twomillion/chmod.png) _**Figure 33**: Asignando Permisos de Ejecución a los Archivos del exploit y Luego lo Ejecutamos._

Ok ahora dejamos esa terminal y nos vamos a otra, nos conectamos de vuelta por ssh y ejecutamos el siguiente comando **./exp**

![](/assets/img/HTB/writeup-twomillion/-exp.png) _**Figure 34**: Ejecutando el Exploit en una Nueva Sesión SSH._

Y somos root.

![](/assets/img/HTB/writeup-twomillion/root.png) _**Figure 35**: Usuario root._

Busquemos la **Flag**.

![](/assets/img/HTB/writeup-twomillion/root-flag.png) _**Figure 36**: Flag de root._

## Mensaje de agradecimiento 💬
### thank_you.json 📃

Hay otro archivo que se llama **thank_you.json**, vamos a ver que contiene.

![](/assets/img/HTB/writeup-twomillion/cat-thak_you-json.png) _**Figure 37**: Contenido de Archivo thank_you.json_

Esta encoding con **"url"**, usamos [CyberChef](https://cyberchef.org).

![](/assets/img/HTB/writeup-twomillion/encoding-url.png) _**Figure 38**: Decoding con CyberChef._

El resultado dice que esta encoding en **"hex"**.

![](/assets/img/HTB/writeup-twomillion/encoding-hex.png) _**Figure 39**: Contenido Encoding en Hex._

Vemos que esta cifrado con **"xor"** y  tenemos la key **"HackTheBox"** y a la par esta encoding en **"base64"**

![](/assets/img/HTB/writeup-twomillion/message.png) _**Figure 40**: Contenido Cifrado en Xor y Encoding Base64._

```text
Dear HackTheBox Community,

We are thrilled to announce a momentous milestone in our journey together. With immense joy and gratitude, we celebrate the achievement of reaching 2 million remarkable users! This incredible feat would not have been possible without each and every one of you.

From the very beginning, HackTheBox has been built upon the belief that knowledge sharing, collaboration, and hands-on experience are fundamental to personal and professional growth. Together, we have fostered an environment where innovation thrives and skills are honed. Each challenge completed, each machine conquered, and every skill learned has contributed to the collective intelligence that fuels this vibrant community.

To each and every member of the HackTheBox community, thank you for being a part of this incredible journey. Your contributions have shaped the very fabric of our platform and inspired us to continually innovate and evolve. We are immensely proud of what we have accomplished together, and we eagerly anticipate the countless milestones yet to come.

Here's to the next chapter, where we will continue to push the boundaries of cybersecurity, inspire the next generation of ethical hackers, and create a world where knowledge is accessible to all.

With deepest gratitude,

The HackTheBox Team
```
{: file='thank_you.json'}

## Opinión 💬 [#](#opinion) {#opinion}
Esta máquina se toco un **CVE**, y mucho **BurpSuite**, tambien el concepto de la máquina me entretuvo y me gusto ya que me uni a [HackTheBox](https://hackthebox.com) este año 2023 y no tuve la posibilidad de realizar el CTF de invitación, tambien el mensaje de mail, para buscar info de como explotar el CVE y el mensaje de agradecimiento, que me gusto mucho y recalco la primera oración del segundo parrafo y el ultimo parrafo.

**"From the very beginning, HackTheBox has been built upon the belief that knowledge sharing, collaboration, and hands-on experience are fundamental to personal and professional growth"**

<span style="color:red"> Span </span>:

"Desde el principio, HackTheBox se ha basado en la creencia de que el intercambio de conocimientos, la colaboración y la experiencia práctica son fundamentales para el crecimiento personal y profesional"

**"We will continue to push the boundaries of cybersecurity, inspire the next generation of ethical hackers, and create a world where knowledge is accessible to all"**

<span style="color:red"> Span </span>:

"Seguiremos ampliando los límites de la ciberseguridad, inspirando a la próxima generación de hackers éticos y creando un mundo en el que el conocimiento sea accesible para todos"
