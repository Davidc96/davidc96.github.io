---
layout: post
author: David Cuadrado
tag: [Windows, Web, Cloud]
title: "Ataques Laterales en el Cloud: Pass The PRT"
---
![PORTADA](https://davidc96.github.io/assets/images/posts/PASSPRT/portada.jpg?style=centerme)

Buenas a todos, después de un tiempo bastante ocupado con otros temas, he estado investigando Azure Active Directory, ya que, la mayoría de empresas están migrando a esta nueva tecnología en el Cloud.

Por decirlo así, Azure Active Directory, es un producto de Microsoft Azure que funciona de una manera muy parecida al convencional Active Directory. Permite gestionar usuarios, grupos, dispositivos, políticas etc. con la finalidad de poder gestionar aplicaciones web.

Es muy útil, por ejemplo, para poder acceder desde la cuenta corporativa a cualquier web interna de la empresa, ya sea una intranet o a las herramientas de Office 365 web (OWA), incluyendo Outlook Web, Teams, Sharepoint etc.

Azure Active Directory permite gestionar el inicio de sesión de usuarios mediante SSO o Single Sign-On que permite que con solo loguearte una vez, puedas acceder a multiples aplicaciones, también tiene compatibilidad con MFA y lo mas importante... TIENE UNA VERSIÓN 100 % GRATUITA PERO LIMITADA por lo que se vuelve una herramienta muy interesante para montar laboratorios :P

En la actualidad, es muy normal ver empresas con Active Directory híbrido, es decir tienen su Active Directory On-Premises, o como lo llama Microsoft AD FS, y Azure Active Directory debido a que en Azure AD, no se pueden configurar políticas globales (GPO) a nivel de cada estación cliente, Azure AD está pensado para implementar políticas pero a nivel de aplicaciones Web y no de estaciones finales. Gracias a una aplicación que proporciona Microsoft llamada AD Sync, se sincronizan los AD.

Una de las políticas que contiene Azure AD es la política de restricción por dispositivo, lo que dice es que un dispositivo ha de estar previamente autorizado por la empresa para que pueda acceder a la página web, en caso contrario aparecería un mensaje como este.

![NOT_AUTH_MESSAGE](https://davidc96.github.io/assets/images/posts/PASSPRT/noauthmessage.jpg?style=centerme)

Cuando vi este mensaje me hice la siguiente pregunta ¿Como sabe Microsoft que estoy accediendo desde un ordenador que no está registrado en la web? De ahí empezé a investigar pero antes de nada vamos a montarnos un pequeño laboratorio de Azure Active Directory para poder continuar que encima es gratis :P

## Construyendo el laboratorio

Para crear nuestro propio laboratorio de Azure Active Directory, es necesario crearnos una cuenta de Microsoft (Si tenéis Github, os sirve igual) y acceder al <a href="https://portal.azure.com/">Portal de Azure</a>

Una vez accedido, aparecerán 3 cuadros grandes, en uno de ellos pone Administrar Azure Active Directory > Ver, en el caso de que no os apareciera, debajo de las cajas, vereis un icono que pone Azure Active Directory, le dais click. Si no os aparece en ningún lado, ir al buscador y poneis Azure Active Directory.

![AZURE_PORTAL](https://davidc96.github.io/assets/images/posts/PASSPRT/PortalAzure.jpg?style=centerme)

Por defecto, Azure te crea un Azure Active Directory, yo en mi caso, como podemos crear tantos Azure AD como queramos, se crea uno nuevo, pero si quereis, podeis usar el que ya viene por defecto.

Crear un Azure AD es muy fácil, en las opciones de arriba > Administrar Inquilinos > Crear, seguiís los pasos y ya lo tendréis creado.

![CREATE_TENANT1](https://davidc96.github.io/assets/images/posts/PASSPRT/CreateTenant1.JPG?style=centerme)

![CREATE_TENANT2](https://davidc96.github.io/assets/images/posts/PASSPRT/CreateTenant2.JPG?style=centerme)

Yo en mi caso voy a usar un Azure AD que ya tenía creado para pruebas cuando intenté sacarme la certificación de Azure 500. Lo primero que vamos a hacer es crear un usuario, lo llamaré dcuadrado (por mi nombre David Cuadrado), para eso vamos a la pantalla inicial de Azure AD, Agregar > Usuario

![ADD_USER](https://davidc96.github.io/assets/images/posts/PASSPRT/AddUser.JPG?style=centerme)

Vais rellenando los campos que piden y en Roles, le dais click a Usuario y lo cambiais por Administrador Global

<b> Pequeño Parentesis: Azure AD tiene diferentes roles de Administrador, cada rol de administrador administra (nunca mejor dicho) diferentes partes del Azure, ya sea para aplicaciones, otro usuarios, otro políticas o incluso administradores para crear máquinas virtuales dentro de Azure. Recomiendo leeros que hace cada rol si vais a montar algún laboratorio sencillo en vuestra casa. En nuestro caso, le vamos a dar super admin a este nuevo usuario para evitar problemas de permisos</b>

Una vez creado el usuario necesitamos crear una máquina virtual con Windows 10 para registrar esa máquina al nuevo Azure AD que hemos creado. Yo en mi caso uso VMWare, ya que lo veo más sólido que VirtualBox xD.

Una vez instalado Windows 10, hay que unirlo al Azure Active Directory, para ello vamos a Configuración > Cuentas > Obtener acceso a trabajo o escuela > Conectar

![CONNECT_AAD](https://davidc96.github.io/assets/images/posts/PASSPRT/AADConnect.jpg?style=centerme)

Una vez aparezca la nueva ventana, le damos a Unir este dispositivo a Azure Active Directory, ingresamos nuestras credenciales y ya estaremos unidos al Azure Active Directory

<b>CURIOSIDAD 1: Si queremos agregar un dispositivo al Active Directory On-premises es necesario unirlo con una cuenta que sea Domain Controller, en Azure AD, da igual si el usuario es administrador global o no, cualquiera puede registrar un dispositivo. Esta opción puede ser cambiada dentro de Azure Active Directory, os invito a investigar :D
Por defecto un usuario sea Domain controller o no, puede agregar hasta 10 dispositivos al AD, lo mismo pasa con Azure AD. Esto es considerado una mala práctica y es recomendable que un usuario no pueda agregar maquinas al dominio. Os dejo un link por si quereis saber mas sobre el tema <a href="https://windowserver.wordpress.com/2015/11/12/quin-puede-unir-mquinas-al-dominio-y-cuntas/">Link</a>. Agradecimientos a @jlmacal por la aclaración</b>
</br>
<b>CURIOSIDAD 2: Al unirse el dispositivo al Azure Active Directory, la cuenta que se ha utilizado para unirse pasará automáticamente a ser Administrador local de la máquina</b>

![AZUREAD_JOINED](https://davidc96.github.io/assets/images/posts/PASSPRT/AzureADJoin.jpg?style=centerme)

Con esto, ya tendremos creado nuestro laboratorio, vamos a proceder a realizar uno de los ataques que he leído :D.

## Microsoft Login

Antes de pasar directamente al ataque, vamos a hablar un poco mas acerca de como funciona el login en Azure Active Directory y como es posible poder vincular un dispositivo al Azure.

Por decirlo así, Microsoft para hacer conectarse al Azure Active Directory, utiliza el protocolo Oauth2, un protocolo muy utilizado para poder hacer SSO.

Oauth es un protocolo que consiste en que una vez insertado las credenciales, el servidor te va a dar 2 tokens, un token de acceso (Access Token) que permite poder acceder al recurso y un token de refresco (Refresh token) que permite pedir un access token sin necesidad de hacer login de nuevo. Este refresh token es utilizado cuando el access token caduca o cuando quiero acceder a un recurso nuevo.

Si esquematizamos esto en pasos quedaría algo como:

1. El usuario inserta su usuario y contraseña
2. El servidor valida y le entrega un Access token que le permite acceder al recurso y un Refresh token que permite renovar el Access token si este caduca
3. El usuario entrega el access token al recurso

En el caso de que el usuario se ha logeado y quiera acceder a otro recurso del cual no ha accedido

1. El cliente envia el Refresh token al servidor.
2. El servidor valida tanto la caducidad como la validez del Refresh Token.
3. Una vez validado, el servidor entrega el access token al nuevo recurso al usuario.
4. El cliente entrega el access token al recurso

![MICROSOFT-OAUTH](https://davidc96.github.io/assets/images/posts/PASSPRT/microsoft-oauth.svg?style=centerme)

Oauth no solamente lo utiliza Microsoft, también es utilizado por grandes plataformas como Google, Facebook, Amazon entre otras.

Si pensamos a nivel de seguridad, el protocolo Oauth tiene un "fallo" en su diseño y eso es conocido como Refresh Token. Ese token es un token que tiene una caducidad de hasta 90 días y si un atacante tiene acceso a ese token, puede crear tantos Access token como el quiera, accediendo a los diferentes recursos sin necesidad del usuario y contraseña, incluso es capaz de saltarse algún que otro MFA dependiendo de como esté configurado el protocolo.

Hoy en día, los ciberdelincuentes al troyanizarte el ordenador, buscan las cookies de tu navegador predeterminado precisamente para buscar los Refresh Token y así poder acceder a tus cuentas de Facebook o Google sin necesidad de saber tu contraseña y sin necesidad de tener tu dispositivo móbil para el factor de doble autenticación (2-Factor authentication o actualmente Multi Factor Authentication), en un futuro haré un post hablando sobre este tipo de cookies.

Volviendo al tema, Microsoft llama a su Refresh Token como PRT viniendo de sus siglas Primary Refresh Token, este token actua igual que un Refresh Token de Oauth pero contiene mas información y muy interesante.

## PRT: Información y ataque

El PRT, o Primary Refresh Token es un Token que permite al cliente solicitar un nuevo token de acceso si este ha caducado o no existe. Gracias a una nueva caracteristica llamada Windows Hello For Business, el token PRT se puede pedir sin necesidad de las credenciales de usuario, eso si, el dispositivo ha de estar si o si vinculado al Azure Active Directory. Esto es debido a que cuando nosotros hacemos login al iniciar Windows, el sistema operativo puede solicitar un token PRT para poder tener acceso al recurso. Esto se ve reflejado cuando nosotros accedemos al Edge con la cuenta ya logueada.

![EDGE_LOGIN](https://davidc96.github.io/assets/images/posts/PASSPRT/EdgeLoginAuto.jpg?style=centerme)

Lo que sucede en este caso es que el Sistema Operativo pide, con todos los datos del dispositivo, el PRT a Microsoft, con eso ya puedes acceder a cualquiera de los recursos dentro de ese dispositivo.

Este proceso se puede hacer con Mimikatz, abajo en la bibliografía os dejaré una guía de como hacerlo con Mimikatz pero aquí utilizaremos otro metodo más "legal" y más transparente.

### Pass The PRT: BrowserCore.exe

En Google Chrome, existe una extensión llamada Windows 10 Accounts que sirve para poder autenticarse a un Azure Active Directory que tiene políticas que restringen el acceso a dispositivos registrados. Esta extensión guarda todas las sesiones de Microsoft que tengas activas para que, cuando tu quieras acceder a un recurso, automáticamente accedas a ese recurso como hace Microsoft Edge.

![WINDOWS_ACCOUNTS](https://davidc96.github.io/assets/images/posts/PASSPRT/windowsaccounts.jpg?style=centerme)

Lo que hay detrás de esta extensión es muy curioso. En la bibliografía os voy a dejar un blog al post de un usuario que ha analizado toda la extensión de Chrome. Es muy interesante si te interesa temas de reversing ya que lo explica con bastante detalle.

La conclusión que se llego es que la extensión de Chrome, lo que hace es solicitar el PRT a Microsoft, Microsoft entrega dicho PRT y luego lo guarda para generar los tokens de acceso de las diferentes aplicaciones.

Tu al token PRT puedes acceder a el, pero es necesario Mimikatz y tener permisos de SYSTEM para lograr acceder a él, cabe pensar que la extensión de Google Chrome, no tiene un Mimikatz por detras XD pero si utiliza un binario que viene instalado en Windows por defecto llamado BrowserCore.exe

BrowserCore, está localizado en la carpeta Archivos de Programa > Windows Security > BrowserCore, es un programa que se encarga de pedir el token PRT a Microsoft. 

Con que tu solamente le pases un JSON con la acción que quieres, la URL de login y la URI de sesión, este automáticamente enviará, junto a mas información, todo lo necesario a Microsoft para que te de el token PRT.

Lo increible de este método es que no necesitas permisos de administrador y es fácilmente scripteable en Python. Algo a destacar es que el input de esta aplicación, lo recibe mediante PIPELINES por lo que ejecutar este programa no es trivial.

Actualmente adjunto el script en Python de quien hizo esta investigación:

```python
import subprocess
import struct
import json
process = subprocess.Popen([r"C:\Archivos de Programa\Windows Security\BrowserCore\browsercore.exe"], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
inv = {}
inv['method'] = 'GetCookies'
inv['sender'] = "https://login.microsoftonline.com"
inv['uri'] = ''
text = json.dumps(inv).encode('utf-8')
encoded_length = struct.pack('=I', len(text))
print(process.communicate(input=encoded_length + text)[0])
```

Este script lo que hace es ejecutar BrowserCore.exe pasando como input un JSON a través de la PIPE:

- Method: Es la acción que queremos hacer con BrowserCore, en este caso, queremos el PRT.
- Sender: Es la página de login de microsoft.
- URI: La página de login de Microsoft entera con todos sus campos.

Para hacer la prueba, abriremos una ventana de incognito en Chrome para que no tengamos la sesión de Microsoft y accederemos a https://login.microsoftonline.com

Una vez dentro, copiaremos toda la URL, la pegaremos en el campo inv['uri'] y ejecutaremos el script.

![TOKEN_GENERATED](https://davidc96.github.io/assets/images/posts/PASSPRT/TokenGenerated.jpg?style=centerme)

Ya tendríamos nuestro token PRT generado, Microsoft lo llama x-ms-RefreshTokenCredential.

Ahora lo que hay que hacer es ir al navegador donde está la página de login, borrar todas las cookies e inyectar esta nueva cookie:

- Nombre: x-ms-RefreshTokenCredential
- Value: el base 64 generado
- HttpOnly: True

Una vez inyectada, ya tenemos acceso al portal de office sin necesidad de la contraseña del usuario.

![OFFICE_ACCESS](https://davidc96.github.io/assets/images/posts/PASSPRT/OfficeOK.jpg?style=centerme)

<b>DATO 1: Al parecer este ataque solo me ha funcionado en Chrome en modo de incognito, en Edge no me ha acabado de funcionar.</b>
</br>
<b>DATO 2: Si al seguir los pasos no funciona a la primera, eso es porque en la URI que te ha dado al principio Microsoft no incluye en client_request_id. Simplemente deja que no funcione y copia de nuevo la URI, pegala al script y genera la cookie de nuevo</b>
</br>
<b>DATO 3: Este token PRT tiene una caducidad de 14 días como máximo, pasado esos 14 días hay que solicitar de nuevo el token PRT</b>
</br>
<b>DATO 4: A pesar de que este ataque se ha probado en la misma máquina virtual asociada al Azure AD, es posible extraer este token e inyectarlo en una máquina no asociada al dominio.</b>

## Conclusiones

Como conclusiones, este ataque aprovecha un "fallo", y lo ponemos entre muchas comillas, en Oauth acerca de que pasaría si el atacante obtiene el Refresh Token. Como siempre, Microsoft hace de las suyas y sin necesidad de Mimikatz o de permisos elevados, podemos obtener un token PRT de 14 días de duración gracias a la extensión de Google Chrome.

A diferencia del token generado por BrowserCore, el token generado por Mimikatz es un token que tiene una duración de 90 días debido a que no incluye según que campos especificos de la URI. Antes con BrowserCore, podías generar tokens de 90 días pero eso lo caparon (creo que por razones obvias xD).

Con esto finalizamos este post, espero que lo hayais disfrutado y como siempre, hacerlo todo en un entorno controlado. Hasta el siguiente post

## Bibliografía

<a href="https://dirkjanm.io/abusing-azure-ad-sso-with-the-primary-refresh-token/">Abusing Azure AD SSO with the Primary Refresh Token</a>
</br>
<a href="https://stealthbits.com/blog/lateral-movement-to-the-cloud-pass-the-prt/">LATERAL MOVEMENT TO THE CLOUD WITH PASS-THE-PRT (MIMIKATZ WAY)</a>