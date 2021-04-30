---
layout: post
author: David Cuadrado
tag: Esteganografía
title: "MHMB: Ocultando información en notas musicales"
---

![MIDI_IMAGE_PROGRAM](../assets/images/posts/MHMB/MIDI_LOGO.svg?style=centerme)
El otro día, estube investigando acerca de técnicas para poder exfiltrar información sin ser detectados. Las técnicas que pasan mas desapercibido a los ojos de un SOC son las técnicas de esteganografía, y el codificar información dentro de una imagen es una práctica muy habitual, tal es así, que actualmente, los nuevos Next Generation Firewall ya analizan las imagenes en busca de bits de mas o incluso mediante IA, detectan si una imagen contiene información o no. Un día me levanté y me pregunté: ¿Es posible poder ocultar información dentro de un fichero MIDI?

Las técnicas que vi por Internet consistian en hacer mas grande el fichero MIDI para que, al final de este, pudieras poner lo que tu quisieras. También, encontré otra técnica que implicaba en codificar el mensaje dentro de los metadatos de este tipo de ficheros, pero claro, no es lo que yo busco porque al fin y al cabo, alguién con nociones de arrastrar-soltar-scrollear el fichero contra un editor hexadecimal podría descifrar el mensaje. Lo que buscaba era poder ocultarlo, que necesites 2 programas (Un Encoder y un Decoder) y junto a una clave privada, poder crear un fichero MIDI, que suene como un fichero MIDI (Sin que el Reproductor de Windows Media de un error) y que no se viera a ojos de un editor hexadecimal y de ahí nació la idea de
MHMB o por sus siglas Midi Hidden Message Builder.

# Esteganografía en ficheros MIDI
Para poder realizar este proyecto, estaba pensando en donde podría ocultar el mensaje. Como bien he dicho, no quiero que se vea a ojos de un editor hexadecimal y a su vez tampoco quiero complicarme la vida por lo que me planteé la siguiente pregunta ¿Y si creo música?

## Notas musicales y desarrollo de la aplicación
En este punto seguramente estais pensando que ya se me ha ido la cabeza ¿Que tiene que ver crear música con ocultar un mensaje?. Darle una vuelta de tuerca y pensarlo mejor. Una canción está creada por notas (DO RE MI FA SOL LA SI). ¿Con una técnica de substitución, puedo convertir una letra a una nota músical?

¡Venga va! ¡¿Como vas a poder hacer una substitución del abecedario que contiene 25 letras con tan solo 7 notas?! Tranquilo, estamos obviando muchas cosas. Un piano esta formado por teclas blancas y negras, las 8 notas mencionadas con anterioridad se corresponden a las 8 blancas, pero también podemos utilzar las teclas negras.
![PIANO](../assets/images/posts/MHMB/Piano.jpg?style=centerme)

Las teclas negras augmentan un semitono de la tecla blanca anterior, este semitono es conocido como "sostenido" o "bemoll" y se representa con el símbolo #. En este caso, tendriamos la siguiente escala: DO DO# RE RE# MI MI# FA SOL SOL# LA LA# SI teniendo en total unas 12 notas.

Pero aún siguen faltando 13 notas más y e aquí donde entra en juego las octavas.
Una Octava es una agrupación de 12 notas (DO DO# RE RE# MI MI# FA SOL SOL# LA LA# SI) cuya diferencia radica en que tan grave o tan agudo es esa nota.
Estas octavas están agrupadas en este orden en un piano:
```
(DO DO# RE RE# MI MI# FA SOL SOL# LA LA# SI)(DO DO# RE RE# MI MI# FA SOL SOL# LA LA# SI)(DO DO# RE RE# MI MI# FA SOL SOL# LA LA# SI)....
<------------- Octava 0 -------------------><------------- Octava 1 -------------------><------------- Octava 2 ------------------->....
```

Siendo la Octava 0 la agrupación más grave y la Octava 9 la agrupación mas aguda.
Si hacemos las matemáticas tenemos que 12 notas, que forma una octava, multiplicado por 9 octavas, da un total de: ¡108 notas! Mas que suficiente para hacer nuestro diccionario por substitución, incluso podemos incluir minusculas, como por ejemplo:
A -> DO (oct. 0)
a -> DO# (oct. 0)  
B -> RE (oct. 0)  
b -> RE# (oct. 0) 
etc.

El algoritmo que hariamos para poder substituir una letra del abecedario por una nota sería tal que así:

```
variable: dictionary(key: letra_abecedario, value: nota_musical)
variable: string(user_string) INPUT
variable: string(encoded_string_result) OUTPUT

for letter in user_string do:
    encoded_string_result += dictionary(key: letter)
endfor

return encoded_string_result
```
## Mejorando el algoritmo, añadiendo un poco más de seguridad a nuestro algoritmo
Al hacer la substitución directa, se puede utilizar una cadena de texto para crear música, el problema es que es muy fácil descubrir nuestro primer diccionario, al fin y al cabo, substituimos la primera nota de la primera octava por la primera letra del abecedario. Es por eso que vamos a darle un poquito de seguridad a esto.

En esteganografía, no es solamente importante tener la muestra y los dos programas que codifican y decodifican, es importante también tener una "key", una llave para que, en el caso de que nos robaran el software, que no les parezca tan fácil decodificar el MIDI.

Como tenemos un total de 108 notas, podemos jugar un poco con eso para poder definir nuestra key. En este caso, lo que se me ocurre es que la key sea la octava donde se va a definir el diccionario.

Entonces si el usuario decide como key 4, el abecedario empezará a partir de la octava numero 4 e ira incrementandose hasta que se haya definido definitivamente.

## Prueba de conceptos
Si has llegado hasta aquí, quiere decir que el post te ha interesado, así que despues de tanta teoría vamos a pasar a la práctica.
Para ello nos vamos a clonar la herramienta MHMB
```sh
git clone https://github.com/Davidc96/Midi-Hidden-Message-Builder
```

Una vez clonada, la abrimos con Visual Studio 2017 o 2019 y compilamos la solución, esto no debe de llevar ni 4 segundos.
![VS_Compiled](../assets/images/posts/MHMB/VS2019Compiled.JPG?style=centerme)

Copiamos los 2 ejecutables generados, tanto el MHMBEncoder.exe y el MHMBDecoder.exe, y la DLL  Melanchall.DryWetMidi.dll a un directorio nuestro y abrimos una consola de comandos:
![Files_Compiled_working_dir](../assets/images/posts/MHMB/CMDWorkingDir.JPG?style=centerme)

### Encoding
Para codificar, hay que crear un fichero .txt que le llamaremos Hola-mundo.txt y pondremos como texto "Hola Mundo!".
Una vez realizado ejecutaremos el comando dentro del directorio de trabajo.
```sh
MHMBEncoder.exe -h
```
Veremos como funciona la herramienta
![MHMB_Options](../assets/images/posts/MHMB/MHMBEncoderOptions.JPG?style=centerme)

Al parecer como parametros tenemos el -i que sirve para indicarle un fichero de texto, el -o para indicarle el nombre del fichero midi y -k para asignar una llave.
Vamos a poner como ejemplo el siguiente comando:
```sh
MHMBEncoder.exe -i Hola-mundo.txt -o hello.mid -k 4
```
Este comando generará un fichero MIDI con llave 4 a partir de nuestro .txt, al ejecutarlo este es nuestro resultado:
![MHMBEnc_Results](../assets/images/posts/MHMB/MHMBEncResults.JPG?style=centerme)

Podemos abrir el fichero generado con el reproductor de Windows, cuando lo escuché me pareció bastante tétrico la verdad jaja

### Decoding
Para decodificarlo se usa el MHMBDecoder.exe para ello ejecutamos el siguiente comando:
```sh
MHMBDecoder.exe -i hello.mid -k 4
```
![MHMBD_Results](../assets/images/posts/MHMB/MHMBDecResults.JPG?style=centerme)
¡Voila! Ya tenemos el mensaje decodificado

## TURNO PARA VOSOTROS 
¿Que pasaría si intentaramos decodificar el midi con una clave diferente?
¿Se os ocurre alguna manera de poder decodificar el mensaje si perdemos la clave?

Probarlo vosotros mismos y sacar vuestras propias conclusiones.

Muchas gracias por leer el post y que tengais una buen día
