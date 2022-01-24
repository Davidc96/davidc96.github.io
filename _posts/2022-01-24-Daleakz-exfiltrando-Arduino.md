---
layout: post
author: David Cuadrado
tag: Hardware Hacking
title: "Proyecto DaLeakz: Exfiltrando información utilizando Arduino"
---
![PORTADA](https://davidc96.github.io/assets/images/posts/DALEAKZ/main_pic.jpg?style=centerme)

Primer post del año y esta vez os traigo un proyecto que pensé mientras buscaba maneras de saltarse el DLP de una empresa. Hoy en día, las empresas, para evitar fugas de información, instalan en los ordenadores de sus empleados para evitar la extracción de documentos.

Normalmente suelen ser programas que van controlando todos aquellos movimientos que realizan los empleados, desde a que páginas se conectan a Internet como los dispositivos externos que se conectan mediantes los puertos USB. Estos softwares se conocen como DLP o Dataleak Prevention. Un ejemplo típico que toda empresa tiene como DLP es en sus servidores de correos, donde si se detecta un envio de documentos a una cuenta personal, el correo se bloquea.

Esto es debido a que la mayoría de filtraciones se realizan mediante el correo pero existen otras maneras de filtrar documentos o archivos sin necesidad del correo y aquí es donde entra en juego las USBs.

La USB es el método más fácil y rápido para filtrar información, no obstante, existen soluciones que bloquean dichos puertos USB para evitar el envío de datos. Sin ir mas lejos, HP tiene su propio software de bloqueo de puertos USB donde el Administrador de sistemas puede crear una Whitelist de dispositivos oficiales utilizados por la empresa, ya que cada USB tienen un VID o Vendor ID y un PID o Product ID.

A pesar de existen muchas maneras de saltarse el DLP, el reto que me he planteado es poder filtrar información mediante el puerto USB y ahí es donde nació la herramienta DaLeakz.

# DaLeakz: Exfiltrando datos utilizando un Arduino

![DALEAKZ](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzCLI.jpg?style=centerme)

El reto personal que me había planteado, derivado de otro reto de un compañero xD, es poder filtrar información a través de la USB, y ahí es cuando he caido en utilizar el puerto Serie y un Arduino.

¿Porque Arduino? Para empezar, no utilizaría un Arduino normal como un Arduino UNO, sino que utilizaría un Arduino con procesador ATMega32u4. El motivo principal es porque ese procesador es detectado por el sistema operativo como un HID haciendo que este sea detectado como un dispositivo externo como si fuera un teclado o un ratón. El otro motivo es que para algunos Arduinos, es necesario instalar el software de Arduino que incluye el Driver para poder operar con ello. Con suerte, gracias al ATMega32u4 no es necesario instalar el Driver propio de Arduino si no que con el generico, ya se pueden hacer cositas.

La idea general es "convertir" el Arduino en una unidad de almacenamiento externa pero no podemos utilizar el protocolo estandar utilizado ya que este sería facilmente detectado por el DLP, por lo que utilizaremos el puerto Serie y un protocolo creado desde 0 sencillo para poder enviar datos al Arduino.

Otro problema es que el Arduino tiene una capacidad muy limitada (demasiado), por lo que buscaremos por Aliexpress algún ATMega32u4 que tenga el módulo de MicroSD incorporado y con buscando me he topado con el DN-3212 BADUSB ATMega32u4

![ATMEGA32U4](https://davidc96.github.io/assets/images/posts/DALEAKZ/atmega32u4.jpg?style=centerme)

La verdad me hizo gracia el aparato porque tiene un ESP32 incorporado (aunque para instalar el firmware del ESP32 es necesario hacer un puente...) y para este proyecto no va a ser necesario. Vosotros podéis compraros uno sin el módulo ESP32, yo lo compré porque me hizo gracia xD.

## Instalación y puesta en marcha

Para poder utilizar DaLeakz, primero es necesario que os cloneis el <a href="https://github.com/Davidc96/Daleakz">repositorio</a>

El repositorio está formado por 2 carpetas:
- La carpeta arduino_project, tiene todo lo necesario para poder cargar el sketch al Arduino
- La carpeta python_interface contiene el CLI para poder comunicarse con el Arduino

### Instalando el Sketch al Arduino

El primer paso es tener una MicroSD y formatearla en FAT32 (Muy importante debido a que la librería de Arduino para las SD es una extensión obsoleta de SDLib por lo que el único formato que soporta es FAT32)

Una vez tenemos la MicroSD preparada y montada en el Arduino, conectamos el Arduino al PC, ejecutamos el Arduino ID y abrimos el archivo main.ino situado en la carpeta arduino_project

![ARDUINO_IDE](https://davidc96.github.io/assets/images/posts/DALEAKZ/arduinoide.jpg?style=centerme)

Una vez abierto, seleccionais el puerto donde está conectado el USB, la placa la seleccionais como Arduino Leonardo y le dais a la Flecha y si todo ha tenido éxito, ya tenéis el Arduino preparado.

### Utilizando el CLI de Python

Para poder utilizar el CLI, vamos a la carpeta python_interface y ejecutamos los siguientes comandos:

```
python3 -m pip install pyserial pyfiglet
python3 -h daleakz.py
```

Una vez hecho esto, se nos mostrará la siguiente pantalla

![DALEAKZ_HELP](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzCLI.jpg?style=centerme)

Enviando un PDF al Arduino

![DALEAKZ_WRITE](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzwrite.jpg?style=centerme)

Recibiendo un archivo desde el Arduino

![DALEAKZ_READ](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzread.jpg?style=centerme)

Obteniendo una lista de los ficheros dentro de la MicroSD

![DALEAKZ_LIST](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzlist.jpg?style=centerme)

Eliminando un fichero dentro de la MicroSD

![DALEAKZ_REM](https://davidc96.github.io/assets/images/posts/DALEAKZ/daleakzremove.jpg?style=centerme)

## Desarrollando un nuevo CLI para el Arduino

En el caso de que quieras desarrollar un nuevo CLI en otro idioma, el Arduino cuenta con una "API" muy sencilla para poder enviar y recibir datos.
En el README.md de la carpeta arduino_project, hay un apartado donde explica el protocolo con todo detalle, es un protocolo muy sencillo y muy fácil de entender basandose en 5 comandos: leer, escribir, eliminar, obtener el tamaño de un fichero y obtener el listado de directorios. Lo único que tienes que tener son estas consideraciones:

- Arduino tiene limitaciones a nivel de memoria, si quieres enviar archivos grandes, es necesario fraccionarlo y enviarlos mediante multiples comandos, el mismo caso se aplica a la hora de leerlos.
- Actualmente, para el dispositivo que he comprado, el máximo de información que se puede leer o escribir es de <b>50 bytes por comando</b>, desconozco si se puede mas o menos.
- El contenido del fichero se envía al Arduino en base64, esto es para evitar lo conocido como NULL Bytes. Estos son bytes que rompen el String haciendo que se creen comportamientos extraños, el byte más común es el 0x00. Lo más fácil es enviar caracteres printables sacrificando un poco el tamaño del fichero.
- Una vez recibido el contenido, se decodifica utilizando el base64 y se escribe en el disco.

# Conclusiones

Como conclusiones a este proyecto, la verdad, he conseguido superar el reto saltandome algún que otro DLP que no voy a mencionar. El proyecto en si me ha dado algún que otro dolor de cabeza, sobretodo por la limitación que tiene Arduino pero lo importante es que se ha conseguido el objetivo.

El uso de este proyecto es tu responsabilidad y está creado con fines educativos y de investigación, no me hago responsable por el uso indebido.

Sin más preambulos, muchas gracias por leerme y nos vemos en un siguiente post.