---
layout: post
author: David Cuadrado
tag: [Hardware]
title: "Proxmark3: Debrickeando mi Proxmark con un JLink y Windows 10"
---
![PORTADA](https://davidc96.github.io/assets/images/posts/PASSPRT/AllStuffConnected.jpg?style=centerme)

Buenas de nuevo, aquí estoy de nuevo publicando otra de mis batallitas, pero esta vez con una Proxmark3, os voy a poner en situación.

El otro día me compré una Proxmark3 por AliExpress y finalmente me llegó. Yo todo ilusionado me leí un post de Hackplayers que dice como actualizar la Proxmark para utilizar el <a href="">fork</a> de Iceman. El problema fue que no leí bien y que antes de cargar el nuevo firmware había que modificar un fichero para especificar que Proxmark estás utilizando, resultado a los 5 minutos de haberla recibido la Proxmark ya no funcionaba.

![MEME_CRYING](https://davidc96.github.io/assets/images/posts/PROXMARK3/CryingMeme.jpg?style=centerme)

Buscando y buscando por Internet maneras de como desbrickearla encontré varias maneras, que si con un <a href="https://github.com/Proxmark/proxmark3/wiki/Debricking-Proxmark3-with-buspirate">BusPirate</a>, con una Raspberry Pi, con OpenOCD etc. Pero el metodo que yo vi mas asequible es con una máquina Windows y un SEGGER JLink.

Lo que se va a realizar es flashear directamente el firmware de la Proxmark3 utilizando el bus JTAG contra la placa.

## Material

![MATERIAL_PROXMARK3](https://davidc96.github.io/assets/images/posts/PROXMARK3/Material.jpg?style=centerme)

Para desbrickearla utilicé el siguiente material:

- Una Proxmark3 que no funcione (brickeada)
- Un portátil con Windows 10
- Cables de circuito integrado macho - macho con punta redondeada
- SEGGER JLink
- Cables de circuito integrado hembra - hembra chinos
- Un destornillador con punta pentagonal

Como no me gusta soldar, decidí utilizar unos cables macho - macho que tenía para montar circuitos en protoboards, es muy importante que sean los redondos debido a que los que tienen la punta cuadrada, no entran en los pines de la Proxmark3. Luego para conectar esos macho - macho al JLink, es necesario cables hembra - hembra ya que una de las puntas será para conectar el otro extremo del cable macho - macho y la otra punta será para conectarlo a los pines del JLink.

## Primeros pasos

Los pines de la Proxmark3 Easy se situan justo debajo de la placa, pero antes de conectar los pines es necesario quitar los tornillos de la Proxmark3 y os explico el contexto de todo esto.

Las Proxmarks (sobre todo las que vienen de AliExpress) vienen por defecto con el JTAG deshabilitado o mas bien desactivado precisamente para evitar flashear directamente a la memoria, es por eso que es necesario primero habilitar el JTAG. Para ello, es necesario cortocircuitar el pin 55 del procesador. El pin 55 es un ERASE que precisamente elimina esa restricción del JTAG. Para cortocircuitarlo, yo agarré un cable macho - macho y lo puse junto al pin 54 (que corresponde a un Vdd de 3.3 v) y tener la Proxmark conectada a la corriente. En esta imagen os dejo donde se encontrarían los pines del procesador que se corresponden desde el 49 al 64.

![PLACA_PROCESADOR](https://davidc96.github.io/assets/images/posts/PROXMARK3/MBPin.jpg?style=centerme)

Para encontrar el pin 55 solamente hay que contar las patitas y poner el cable macho - macho entre la patita 54 y 55. 

AVISO: Algunos cables macho - macho tienen un protector alrededor de la punta que sirve para evitar cortocircuitos, si quereis asegurar el tiro, conectar el extremo al pin 55 y el otro extremo al pin de 3.3v del JTAG y debería de funcionar.

Una vez hecho esto volveis a montar la Proxmark3, la desconectais de la corriente y ya podemos proseguir a conectar los pines.

## Conectando los pines al JLink

Para conectar los pines hay que tener en cuenta la siguiente tabla extraída de <a href="https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/jtag_notes.md">RfidResearchGroup</a>

  ---------  ---------<br>
 |1917151311 9 7 5 3 1|<br>
 |201816141210 8 6 4 2|<br>
  --------------------<br>

PM3     JLink<br>
TMS     7<br>
TDI     5<br>
TDO     13<br>
TCK     9<br>
GND     6<br>
3.3     2<br>

![JLINK_PINOUT](https://davidc96.github.io/assets/images/posts/PROXMARK3/jtag_pin.jfif?style=centerme)

En este caso hay que conectar los diferentes pines del Proxmark3 a su respectivo JLink, en mi caso, como no estoy soldando hay que tener cuidado de que los cables no se toquen entre sí ya que podía darse un cortocircuito y acabar de liarla.

![PROXMARK_CONNECTED](https://davidc96.github.io/assets/images/posts/PROXMARK3/PinConnect.jpg?style=centerme)

![PROXMARK_CONNECTED](https://davidc96.github.io/assets/images/posts/PROXMARK3/PinConnect2.jpg?style=centerme)

Cuando conectemos el 3.3v y el GND la Proxmark3 se encenderá. Si veis que parpadea, es decir, la Proxmark3 pita, se apaga y se enciende todo el rato etc. es porque está haciendo falso contacto con el pin de 3.3v (Normal porque no hemos soldado nada) es recomendable conectar el cable USB a la Proxmark y al ordenador para dar esa potencia, conectamos el JLink al PC y ya solamente faltaría el software.

## Instalando el software en el Windows 10
Antes de descargarse el firmware de la Proxmark3 hay que descargarse e instalar el programa para comunicarse con el JTAG que es este de <a href="https://www.segger.com/products/debug-probes/j-link/tools/j-link-commander/">aquí</a>

El último paso es descargar todo el software necesario para instalar el firmware directamente a la placa, para eso, se sigue el tutorial de Hackplayers que lo explica muy bien y puedes encontrarlo <a href="https://www.hackplayers.com/2021/07/nfc-proxmark3-chameleon.html">aquí</a>

En resumen hay que descargarse un repositorio para compilar el código de Proxmark3 conocido como ProxSpace. Para ello tiramos los siguientes comandos en un CMD con permisos de Administrador

```batch
git clone https://github.com/Gator96100/ProxSpace
cd ProxSpace
runme64.bat
```

Al ejecutar el fichero runme64.bat, veremos que se empiezan a instalar una serie de paquetes para poder compilar el firmware

![RUNME_RUNNING](https://davidc96.github.io/assets/images/posts/PROXMARK3/InstallingProxSpace.png?style=centerme)

Una vez finalizado, tendremos una consola Bash y ahí tiramos los siguientes comandos:
```sh
git clone https://github.com/Proxmark/proxmark3
cd proxmark3
make -j
```

Con esto estaremos descargando el repositorio de Proxmark para compilarlo

![PROXMAX_COMPILED](https://davidc96.github.io/assets/images/posts/PROXMARK3/ProxmarkRecovery.png?style=centerme)

Una vez compilado si hacemos un ls a la carpeta recovery, veremos un fichero llamado proxmark3_recovery.bin, ese binario hay que flashearlo a la placa. Si vamos al explorador de Windows donde tenemos Proxspace, hay una carpeta llamada pm3, ahí se encuentra la carpeta donde está el código, hay que copiar el path entero estilo C:\Donde tengas el repo\Proxspace\pm3\proxmark3\recovery

Después de esto, abrimos el programa J-link commander y ejecutamos los siguientes comandos:
```
exec device = AT91SAM7S512
exec EnableFlashDL
erase
loadbin <path en donde tengas el repo>\pm3\proxmark3\recovery\proxmark3_recovery.bin 0x100000
```

Si haces todo correctamente verás que la Proxmark3 ha vuelto a funcionar, incluso se escucha el pitido de Windows cuando detecta un nuevo USB :D

![PROXMAX_FLASHED](https://davidc96.github.io/assets/images/posts/PROXMARK3/ProxmarkProgrammed.png?style=centerme)

Si ejecutamos el cliente de proxmark3.exe o el de Iceman, veremos que ha vuelto a la vida :D

![PROXMARK_WORKING](https://davidc96.github.io/assets/images/posts/PROXMARK3/ProxmarkWorking.jpg?style=centerme)

## Notas

Como podeis observar, hemos instalado la versión original de Proxmark3 y no el fork de Iceman, esto es debido a que el fork de Iceman hay que añadir un paso adicional para compilar y flashear el firmware, ese paso muchas veces se obvia o no se hace lo que causa que la Proxmark se brickee xD.

Otro apunte a tener en cuenta es que a mi me daba error al flashear y era porque la Proxmark3 no tenía suficiente corriente, como los cables no estaban soldados, conecté el puerto microusb al PC para acabar de dar esa potencia.

También hay que tener en cuenta que exiten 2 modelos de Proxmark3, la de 256 y la de 512 y el nombre de CPU es diferente, si usas la versión de 256 el nombre de la CPU es AT91SAM7S256 en vez de AT91SAM7S512.

Con esto concluye este post, muchas gracias por leerlo y nos vemos en un siguiente post.
