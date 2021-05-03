---
layout: post
author: David Cuadrado
tag: [Reverse Engineering, Videojuegos]
title: "Destripando el glitch del Mundo -1 de Super Mario Bros: Introducción, Hardware de la NES y conociendo el Glitch a alto nivel"
---

![MUNDO_MENOS_1_PORTADA](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/Mundo361Portada.jpg?style=centerme)
En este post voy a explicar una de las investigaciones que hice hace unos años sobre uno de los glitches más conocidos por el paronama gamer de uno de los videojuegos que marcaron historia, el Super Mario Bros para la Nintendo Entertainment System (NES).

Esta investigación la dividiré en 3 post. En este primer post haré una pequeña introducción acerca de la NES así como una explicación muy por encima acerca del mundo -1. En los siguientes post, entraremos mas en detalle acerca del glitch y explicaremos porqué este se produce a bajo nivel. Durante el proceso, voy a ir explicando conceptos básicos de Reversing y Ensamblador.

# Nintendo Entertainment System
La Nintendo Entertainment System o por sus siglas NES, es una de las primeras consolas de Nintendo en 1983. Dicha consola fue una revolución en aquella época, vendiendo más de 60 millones a nivel mundial. Lo que hacía único a esta consola era la calidad gráfica que ofrecía frente a la otra competencia  como Commodore o Spectrum con colores vivos, juegos medianamente optimizados, precio asequible para aquella época (estamos hablando que en aquella época existian calculadoras por 60 u 80 €) y unos controles únicos adaptados a todo el mundo.

Como bien he dicho, la NES es la primera consola de Nintendo pero las primeras versiones de esta consola solo se vendieron en Oriente bajo el nombre de Famicom, por su acrónimo Family Computer. Debido al éxito que tuvo tanto en Japón, China y demás países de la zona, Nintendo vendió la consola bajo el nombre como conocemos actualmente como NES o Nintendo Entertainment System.
![FAMICOM_IMAGE](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/Famicom.jpg?style=centerme)

## Características técnicas

La NES es una consola formada por una CPU y 3 registros A, X, Y de tamaño 8-bits por parte del procesador y una VRAM de 16kB donde se guardaban los valores que venían de los cartuchos de la ROM o de la PPU (Picture Pixel United), el controlador de video de la consola.

La CPU es un modelo MOS 6502 de 8 bits de registros con 16 bits para escribir direcciones (A0-A15) y 8 bits(D0-D7) para recibir datos de las direcciones.
![CPU_MOS](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/NESCPU.jpg?style=centerme)

La ALU, está formada por 3 registros adicionales (flags) N(Negative), Z(zero), C(Carry), D(Decimal), V(Overflow), I(Interrupcion)

![CPU_MOS_DE](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/NESCPUDiagram.jpg?style=centerme)

### ASSEMBLY del procesador 6502

Esta CPU es de arquitectura de 8 bits por lo que hay un total de 256 instrucciones aproximadamente, las mas destacadas son las siguientes:

|Instrucción| Explicación                                                                          |
|:----------|--------------------------------------------------------------------------------------|
|LDA        | Guarda el valor que hay en memoria o directamente y lo almacena en el registro A     |
|LDX        | Guarda el valor que hay en memoria o directamente y lo almacena en el registro X     |
|LDY        | Guarda el valor que hay en memoria o directamente y lo almacena en el registro Y     |
|STA        | Guarda el contenido que hay en el registro A y lo almacena en memoria                |
|STX        | Guarda el contenido que hay en el registro X y lo almacena en memoria                |
|STY        | Guarda el contenido que hay en eñ registro Y y lo almacena en memoria                |
|TAX        | Mueve lo que hay en el registro A al registro X                                      |
|TAY        | Mueve lo que hay en el registro A al registro Y                                      |

### Estructura de una instrucción en el 6502

La estructura de una instrucción en la cpu 6502 es la siguiente:

|<span style="color:blue">02</span>:<span style="color:red">824C</span>:<span style="color:GoldenRod">BD</span> <span style="color:black">65 07</span>|

Azul: Identificación de la instrucción en la memoria VRAM<br/>
Rojo: Dirección de memoria en la ROM donde está la instrucción<br/>
Dorado: OpCode de la instrucción en este caso es un LDA<br/>
Negro: Direccion de memoria VRAM que coje el operando $0765<br/>

# Super Mario Bros y el mundo -1

El juego de Super Mario Bros es uno de los juegos más vendidos para la Nintendo Entertainment System o NES vendiendo mas de 40 millones de copias. A pesar de los años que tiene el juego actualmente es un juego muy aclamado por los SpeedRuners, una competición que consiste en completar el juego en el menor tiempo posible siendo el record actual de 4 minutos y 54 segundos.

Es un juego de plataformas que consiste en ir completando un total de 32 niveles. El primer nivel es el 1-1 y va incrementando en 1-2, 1-3 y 1-4 antes de pasar al nivel 2-1.

El juego se puede completar de 2 maneras diferentes, una de ellas es pasando por los 32 niveles pero, el juego incluye unos atajos llamados WarpZones que permite saltarte niveles.
![WARP_ZONE_12](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/WarpZone12.jpg?style=centerme)

Existen un total de 3 Warpzones distribuidas de las siguiente manera:
* Una Warpzone en 1-2 que permite llevarte al mundo 4, 3 o 2.
* Una Warpzone en 4-2 que permite llevarte al mundo 5.
* Una Warpzone oculta en 4-2 que permite llevarte al mundo 8, 7 o 6.

Quedaros con el orden de las Warpzones por que va ha ser muy importante para mas adelante.

## Glitch Mundo -1

El Mundo -1 es un glitch muy conocido que consiste precisamente en entrar a dicho mundo debido a una serie de circunstancias que detallaremos en posteriores posts. El mundo -1 es conocido por ser un nivel de mario acuatico y su característica principal es que es infinito, es decir, cuando piensas que has acabado el nivel, este vuelve a empezar otra vez desde el principio. La única manera de escapar de dicho mundo es por Time out.

### Acceso al mundo -1

Para poder acceder a este mundo primero de todo hay que llegar al nivel 1-2, al final del nivel existe un tejado que si vas por encima llegas a la WarpZone de 1-2, si vas por la tubería vas al siguiente que es el 1-3, pero, si rompes el penultimo bloque antes de llegar a la pared y saltas hacia atrás, provocarás un bug conocido como Walk Through Walls.
![GIF_WTW](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/WTWGif.gif?style=centerme)

Este bug es producido debido a que el juego no calcula bien las colisiones haciendo que puedas atravesar paredes, como este comportamiento es inusual, con la inercia del salto hacia atrás, el juego te expulsará de la pared atravesandola. Al atravesar la pared y entrar por la primera tubería que vemos, llegaremos al mundo -1. No solamente eso, lo curioso es que si en vez de entrar a la primera tubería de la Warpzone, llegamos a la segunda, acabaremos en el nivel 5-1. Esto nos deja con la duda de ¿No existía una Warpzone cuya tubería del centro llevaba al mundo 5? ¿Y si realmente por algún extraño motivo, que actualmente desconocemos, al provocar el bug anterior se carga la Warpzone del mundo 4-2 cuya tubería del centro lleva al mundo 5? Vamos a responder estas preguntas haciendo ingenieria inversa del juego.

Un consejo que doy para aquellos que quieran iniciarse a la ingenieria inversa es haceros esta pregunta: ¿Si yo fuera desarrollador y me dicen que tengo que implementar un sistema de 1 o 3 tuberías para ir a otros mundos, como lo haría?

Respondiendo a esta pregunta diriamos, bueno, lo más fácil es tener un Array, en este caso como piden 3 Warpzones que pueden tener de 1 a 3 tuberías, yo lo que haría es crear un Array multidimensional de 3 x 3 (3: Warpzones con 3 tuberías cada una)

Claro, el array lo inicializaría tal y como he comentado antes acerca de que Warpzones hay:
* El primer caso (primera fila del array) corresponde a la Warpzone de 1-2, en ese caso como esta solamente va al mundo 4, 3 o 2 pues inicializaré el array con 0x04, 0x03 y 0x02
* El segundo caso (segunda fila del array) corresponde a la Warpzone de 4-2 pero con solo una tubería. Al ser C, no puedo dejar espacios en blanco, he de sí o sí inicializarlo con 3 valores y como la tubería está en el centro, pues pondré en el valor del centro el número 5 (0x05) y el resto por ejemplo 0xff
* El tercer caso (tercera fila del array) es igual que el primero pero con los mundos 6, 7 y 8 (0x08, 0x07 y 0x06)

Mi array en C quedaría algo así:

```c
char warpzoneNumber[3][3] = {
    {0x04, 0x03, 0x02}, // Warpzone de 1-2
    {0xff, 0x05, 0xff}, // Warpzone de 4-2
    {0x08, 0x07, 0x06} // Warpzone oculto de 4-2
};
```
Aquí uno puede decir: Claro sí, una cosa es que lo programe yo pero otra cosa es como realmente esté programado. Piensa una cosa, a veces los programadores buscamos cosas sencillas y creeme esta solución no equidista mucho de como realmente está programado, al fin y al cabo por muy espectacular que sea el programa, redúcelo a la cosa mas sencilla y te aseguro que no fallarás.

Volviendo al tema, al tener una referencia de como está construido, si buscamos en HxD esos valores seguidos (0x04, 0x03, 0x02) veremos que encontraremos esta información:

.db $04 $03 $02 $00<br/>
.db $24 $05 $24 $00<br/>
.db $08 $07 $06 $00<br/>

¿Veis? De la deducción que hemos sacado sobre como programarlo a realmente como está programado, no equidista de mucho jeje.
La única diferencia que veo es que en la segunda fila en vez de usar 0xff, utilizan 0x24 pero esto tiene una explicación muy interesante.

En la NES el 0x24 corresponde al caracter " " (vacío) algo muy interesante teniendo en cuenta que en la Warpzone se imprimen los números encima de las tuberías. Como el desarrollador, al acceder a la Warpzone de 4-2 de una sola tubería, no quiere imprimir números a los lados, solamente el 5, decidió inicializar su array con un 24 que significa caracter " " (vacío). 

¿Por que esto es importante? Porque al acceder al mundo bugeado, nos aparece en pantalla un _-1, no aparece el número de delante porque en verdad estamos accediendo al mundo 0x24-1 y como 0x24 es un caracter vacío, es por eso que no se muestra en pantalla, de ahí que se llame mundo -1.
![MUNDO_-1](https://davidc96.github.io/assets/images/posts/DSMB/Introduccion/Mundo361.jpg?style=centerme)

# Conclusión

En esta primera parte, hemos dado a conocer un poco la historia de la consola y sus detalles técnicos sin entrar muy en profundidad. También sabemos que es el mundo -1 y como podemos acceder a él. El siguiente post entraremos en detalle en porque se produce este glitch y conoceremos un poco porque se habilita la Warpzone de 4-2 cuando producimos el bug de Walk Through Walls.

