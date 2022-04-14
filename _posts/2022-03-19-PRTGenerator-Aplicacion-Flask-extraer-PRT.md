---
layout: post
author: David Cuadrado
tag: [Windows, Web, Cloud]
title: "PRTGenerator: Aplicación escrita en FLASK para extraer el token PRT"
---
![PRT_PORTADA](https://davidc96.github.io/assets/images/posts/TOOLPRT/portada.jpg?style=centerme)

Buenas a todos, en este post voy a enseñaros una herramienta desarrollada por mi que sirve para poder hacer ataques Pass The PRT y así conseguir acceder a un recurso en Azure que esté restringido sin necesidad de utilizar una máquina autorizada.

A modo resumen, PRT viene de las siglas, Primary Refresh Token y es un token que sirve para poder obtener un token de acceso a la aplicación, si quereis información detallada de como funciona el ataque Pass the PRT y como montaros vuestro propio laboratorio totalmente gratuíto os dejo el <a href="">enlace a mi post que habla sobre el tema</a>. 

Para extraer este token, es necesario estar en una máquina la cual si tenga acceso a ese recurso y como necesitaba extraer ese token de manera rápida y eficiente decidí desarrollar una herramienta en FLASK.

## PRTGenerator: Una herramienta para extraer paso a paso el token PRT

PRTGenerator es una aplicación desarrollada en FLASK que permite paso a paso extraer el token PRT de una maquina que si tiene acceso a ese recurso. Esta herramienta levanta un servicio web para poder extraer el token a través del navegador.

![FLASK_WEB](https://davidc96.github.io/assets/images/posts/TOOLPRT/FLASK_APP.JPG?style=centerme)

### Primeros pasos

La herramienta se encuentra en el siguiente github: <a href="http://github.com/Davidc96/PRTGenerator">https://github.com/Davidc96/PRTGenerator</a>

![IMAGEN_REPO](https://davidc96.github.io/assets/images/posts/TOOLPRT/GITHUB_REPO.png?style=centerme)

Una vez clonado el repositorio, creamos un entorno virtual con venv ejecutando el siguiente comando:

```python3 -m venv .\venv```

Activamos el entorno virtual ejecutando el script

```.\venv\Scripts\activate.bat```

Instalamos FLASK, seteamos la variable FLASK_APP que apunte a main.py.

```
pip install flask
set FLASK_APP=main.py
```

Antes de ejecutar la aplicación es necesario obtener una URL dummy para generar un token PRT no funcional. Para ello abrimos el navegador en modo incognito y vamos a la URL <a href="https://teams.microsoft.com">https://teams.microsoft.com</a> y aparecerá la pantalla de login de Microsoft.

![LOGIN_MICROSOFT](https://davidc96.github.io/assets/images/posts/TOOLPRT/LOGIN_MICROSOFT.png?style=centerme)

Copiamos la URL que aparece en el login y la pegamos dentro del fichero main.py en la línea 4 dentro de la variable DUMMY_URI, esto solo lo hacemos una vez.

![DUMMY_URI_PYTHON_CODE](https://davidc96.github.io/assets/images/posts/TOOLPRT/DUMMY_URL_CHANGE.png?style=centerme)

Ahora si, ejecutamos la aplicación con ```flask run --host 0.0.0.0```

### Uso de la aplicación

Al entrar a la URL http://IP DE LA MÁQUINA:5000, aparecerá la siguiente pantalla:

![MAIN_SCREEN](https://davidc96.github.io/assets/images/posts/TOOLPRT/FLASK_APP.JPG?style=centerme)

La pantalla tiene un paso a paso que se debe de seguir para poder obtener el token PRT.

Pasos:
1. Copiamos la URL de la primera caja de texto y el primer token PRT generado en la segunda caja de texto, entramos y creamos una cookie de la siguiente manera

- Nombre: x-ms-RefreshTokenCredential
- Value: El base64 entregado
- HttpOnly: True

2. Al hacer F5 a la página, volvemos de nuevo al login pero con una URL diferente, copiamos la URL y la pegamos en la tercera caja de texto y le damos a generar.

![F5_LOGIN_COOKIE](https://davidc96.github.io/assets/images/posts/TOOLPRT/TOKEN_GENERATED.png?style=centerme)

3. Copiamos el valor del nuevo token PRT generado y lo ponemos en la cookie x-ms-RefreshTokenCredential y al darle a F5 ya hemos accedido al recurso o a la ventana de MFA.

![LOGIN_MFA](https://davidc96.github.io/assets/images/posts/TOOLPRT/MFA_SCREEN.png?style=centerme)

Y con esto ya estaríamos dentro de la aplicación sin necesidad de credenciales

## Troubleshootings

Este ataque solo funciona en Google Chrome o derivados (creo que si funciona en Microsoft Edge) pero no en Firefox, al menos, no he conseguido que funcione.
En Microsoft Teams, a veces se muestra un error que dice: Ops, algo sale mal. Para resolver esto, haga clic en Cerrar sesión e intente iniciar sesión nuevamente. No te preocupes cuando se cierre la sesión porque cuando vuelvas a iniciarla se utilizará el PRT ya generado.

En Microsoft Teams al entrar con el token PRT, entrá en un bucle infinito, para resolver esto, simplemente escribe en la URL https://teams.microsoft.com y se resolverá el problema.
