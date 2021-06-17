---
layout: post
author: David Cuadrado
tag: [Reverse Engineering, Videojuegos]
title: "Destripando el glitch del Mundo -1 de Super Mario Bros: Estudiando el mundo acuatico detrás del mundo -1 (Parte 3)"
---

![MUNDO_MENOS_1_PORTADA](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/SMBMinus.jpg?style=centerme)

Post relacionados:
-   <a href="https://davidc96.github.io/2021/05/03/Destripando-Super-Mario-Bros-Introduccion.html">Primera parte</a></br>
-   <a href="https://davidc96.github.io/2021/05/20/Destripando-Super-Mario-Bros-Ingenieria-Inversa-Glitch.html">Segunda parte</a></br>

En los post anteriores, descubrimos porque el mundo -1 se llama así, que condiciones se tienen que dar para poder acceder a dicho mundo y porque podemos acceder a él a muy bajo nivel. En esta ocasión estudiaremos porque el mundo -1 es un mundo acuatico e investigaremos si es posible hacer que el mundo -1 pueda llegar a ser otro mundo.

Antes de entrar en materia vamos a mostrar todos los mapas que forman parte del juego

![OVERWORLD](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/AllWorlds.jpg?style=centerme)

Como podemos observar, el juego de Super Mario Bros, se compone de 8 mundos con 4 niveles cada mundo, es decir, un total de 32 mundos.
Cada mundo, tiene asociada una ID definida por la siguiente tabla:
|ID (Hex)| World Number |
|:-------|--------------|
| 25h    | 1-1          |
| 29h    | 1-2A         |
| C0h    | 1-2B         |
| 26h    | 1-3          |
| 60h    | 1-4          |
| 28h    | 2-1          |
| 29h    | 2-2A         |
| 01h    | 2-2B         |
| 27h    | 2-3          |
| 62h    | 2-4          |
| 24h    | 3-1          |
| 35h    | 3-2A         |
| 20h    | 3-2B         |
| 63h    | 3-3          |
| 22h    | 4-1          |
| 29h    | 4-2A         |
| 41h    | 4-2B         |
| 2Ch    | 4-3          |
| 61h    | 4-4          |
| 2Ah    | 5-1          |
| 31h    | 5-2          |
| 26h    | 5-3          |
| 62h    | 5-4          |
| 2Eh    | 6-1          |
| 23h    | 6-2          |
| 2Dh    | 6-3          |
| 60h    | 6-4          |
| 33h    | 7-1          |
| 01h    | 7-2          |
| 27h    | 7-3          |
| 64h    | 7-4          |
| 30h    | 8-1          |
| 32h    | 8-2          |
| 21h    | 8-3          |
| 65h    | 8-4          |

Como se puede observar en la tabla, algunos mundos son exactamente iguales, por ejemplo el mundo 1-4 y el mundo 6-4 tienen la misma ID (60h) o el mundo 7-2 y el 2-2B también tienen la misma ID (01h).

El array de los mundos está situado en la dirección de memoria 0x9CB4 y está representado como un vector.

Si entramos en materia ya acerca del mundo -1, podemos observar que el mundo -1 es una copia del mundo 2-2 o 7-2. Para demostrar esto lo que se va a hacer es utilizar SMBUtil, una herramienta que permite editar el juego de Super Mario Bros, para añadir una tubería en el mundo 7-2 y de paso pondremos un Goomba.
![SMBUTIL_WORLD_EDIT](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/72Edit.jpg?style=centerme)

Ahora al guardar los cambios, ejecutar el juego en FCEUX e ir a los dos mundos (7-2 y -1)
![7_2_1WORLD](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/72_1.jpg?style=centerme)

Como podemos observar, los dos mundos han sido editados, eso quiere decir que de alguna manera, al entrar a la tubería que lleva al mundo -1, el juego interpreta que ese nuevo mundo es un mundo acuático. ¿Porque?

Para eso, vamos a volver de nuevo a volver a analizar el código ensamblador que se encarga de llevarnos a otro mundo al pasar por las tuberías pero lo analizaremos a partir de donde lo habiamos dejado:
```asm
...
01:DF22:BC F2 87  	LDY $87F2,X @ $87F7 = #$05 <--- BREAK
01:DF25:88        	DEY
01:DF26:8C 5F 07  	STY $075F = #$00
01:DF29:BE B4 9C  	LDX $9CB4,Y @ $9CB9 = #$17
01:DF2C:BD BC 9C    LDA $9CBC,X @ $9CC1 = #$28
01:DF2F:8D 50 07  	STA $0750 = #$C2
01:DF32:A9 80     	LDA #$80
```

Si recordamos este fragmento de código del segundo post, la línea 0x01DF22 es aquella que se encarga de traer cual es el siguiente mundo que vamos a entrar. Recordar que, el mundo -1 en realidad es el mundo 23h-1h o 35-1 en decimal solo que, la NES interpreta el valor 23h como un espacio en blanco, de ahí que el nombre sea _-1.

Si empezamos desde esa primera línea el estado de los registros es el siguiente:
|Registro|Valor|
|:-------|-----|
|   A    |  4h |
|   X    |  4h |
|   Y    |  24h|

La siguiente Línea es un Decremento de Y y lo que hace que el valor 24h pase a ser 23h, después este valor lo almacena en 075F que es la posición de memoria donde se imprime el número del mundo por pantalla.

Pasado esto vamos a la siguiente línea
```asm
...
01:DF22:BC F2 87  	LDY $87F2,X @ $87F7 = #$23
01:DF25:88        	DEY
01:DF26:8C 5F 07  	STY $075F = #$00
01:DF29:BE B4 9C  	LDX $9CB4,Y @ $9CB9 = #$33 <-- BREAK
01:DF2C:BD BC 9C    LDA $9CBC,X @ $9CC1 = #$01
01:DF2F:8D 50 07  	STA $0750 = #$25
01:DF32:A9 80     	LDA 
```

Con la instrucción LDX lo que se hace es seleccionar el mundo en el que estamos, mientras que con LDA seleccionamos el tipo de background que va a tener este mundo.

Para entenderlo mejor, hay que entender estas dos direcciones de memoria (0x9CB4 y 0x9CBC) como si fuera una matriz N x n
```
WorldMatrix -------------------- X -----------------
|   World 1: 1-1, 1-2A, 1-2B, 1-3, 1-4
|   World 2: 2-1, 2-2A, 2-2B, 2-3, 2-4
|   Wordl 3: 3-1, 3-2A, 3-2B, 3-3, 3-4
|   ....
Y
|
|
|
```
Donde el eje Y corresponde el mundo y el eje X es considerado el nivel (-1, -2, -3, -4).
El carro de valores hexadecimales donde se almacena todo esto es el siguiente:

85 E8 60 <span style="color:blue">00 05 0A 0E 13 17 1B 20 </span><span style="color:red">25 29 C0 26 60 28 29 01 27 62 24 35 20 63 22 29 41 2C 61 2A 31 26 62 2E 23 2D 60 33 29 01 27 64 30 32 21 65</span> 1F 06 1C 00 70 97 B0 DF 0A 1F 59 7E 9B A9 D0 01

Lo marcado en azul es el array que corresponde al eje de las Y, mmientras que lo marcado en rojo son las diferentes ID de cada nivel tal y como se ha mostrado en la primera tabla dentro de este post.
Como podemos observar los valores del array que se corresponde al eje Y son los diferentes offsets que indican donde está cada nivel dentro del array de color azul. Por ejemplo, si la Y = 5 el valor blueArray[5h] = 13h, Si ese 13 lo ponemos como indice X, y lo buscamos en el array de color rojo redArray[13h] = 2A que corresponde al mundo 5-1.

Este caso se daría en condiciones normales, pero estamos hablando de que estamos accediendo a la tubería de la WarpZone antes de tiempo, por lo tanto. ¿Que va a pasar en esa situación?

Volvamos al código ensamblador y vamos a seguir paso por paso lo que está pasando:
```asm
...
01:DF22:BC F2 87  	LDY $87F2,X @ $87F7 = #$23
01:DF25:88        	DEY
01:DF26:8C 5F 07  	STY $075F = #$00
01:DF29:BE B4 9C  	LDX $9CB4,Y @ $9CB9 = #$33 <-- BREAK
01:DF2C:BD BC 9C    LDA $9CBC,X @ $9CC1 = #$01
01:DF2F:8D 50 07  	STA $0750 = #$25
01:DF32:A9 80     	LDA 
```

Empezamos desde la instucción 0x1DF26, recordemos que 9CB4 es un array donde se encuentran los diferentes offsets para 0x9CBC (la famosa matriz que hemos hablado anteriormente).
En este caso Y = 23, cuando ejecutamos la instrucción LDX $9CB4,Y vemos que algo no cuadra. $9CB4 es un array de 8 posiciones y le estamos pidiendo que nos de la posicion 23 de 8. Si esto fuera un programa actual, el sistema operativo nos hubiera saltado con un Segmentation Fault pero como esto es la NES, le da igual pedir la posicion 23 de 8 aunque se salga del limite.

La posicion número 23h desde el principio de blueArray[23/8] es el valor 33, y si, como estamos viendo está cogiendo un valor que viene del array donde se encuentran las IDs de los diferentes mundos

85 E8 60 <span style="color:blue">00 05 0A 0E 13 17 1B 20 </span><span style="color:red">25 29 C0 26 60 28 29 01 27 62 24 35 20 63 22 29 41 2C 61 2A 31 26 62 2E 23 2D 60 <b>33</b> 29 01 27 64 30 32 21 65</span> 1F 06 1C 00 70 97 B0 DF 0A 1F 59 7E 9B A9 D0 01

La siguiente instrucción LDA $9CBC,X hará lo siguiente worldIDArray[33h] lo mismo vuelve a pasar el worldIDArray es un array de 23 posiciones y se necesita la posición 33h de 23 (32h), todo muy gracioso la verdad xD.

Dicho valor es 1F que se encuentra justo después del último valor del array worldIDArray.

85 E8 60 <span style="color:blue">00 05 0A 0E 13 17 1B 20 </span><span style="color:red">25 29 C0 26 60 28 29 01 27 62 24 35 20 63 22 29 41 2C 61 2A 31 26 62 2E 23 2D 60 33 29 01 27 64 30 32 21 65</span><b> 1F</b> 06 1C 00 70 97 B0 DF 0A 1F 59 7E 9B A9 D0 01

Si vamos a la tabla de mundos el 1F se corresponde al mundo 2-2 y 7-2, entonces una vez almacenado ese valor en $0750 el juego interpreta que el mapa escogido es como el mundo 7-2 o 2-2 es por eso que el mundo -1 es un mundo acuático.

### Jugando un poco con el mundo -1

Para demostrar esto, vamos a cambiar el valor 1F por cualquier otro mapa, por ejemplo por el de 1-1, fijaos el resultado:
![_1_WORLD_1_1](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/1_11.jpg?style=centerme)

También lo he cambiado por el mundo 1-2 y esto ha pasado:
![_1_WORLD_1_2](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/1_12.jpg?style=centerme)

Lo curioso fue que al cambiar el mundo -1 por el mapa del mundo 1-2, 1-1 cambió de una manera rara:
![1_1_WTF](https://davidc96.github.io/assets/images/posts/DSMB/ThirdPart/11WTF.jpg?style=centerme)

Al parecer ese valor seguramente está ligado con los NPC que aparecen en 1-1 porque si no, no se explica pero creo que eso lo dejaremos para un futuro post jaja.

### Conclusiones
Como conclusión a estos 3 post se puede deducir que el mundo -1 se produce por un cumulo de casualidades y overflows que hacen que el juego se comporte de una manera poco usual. Al fin y al cabo, es posible acceder al mundo -1 porque accedemos a la tubería de la WarpZone antes de tiempo. Es curiosos como por muy poco que te salgas de lo normal en un juego de estos, como es posible que el juego se destabilize tanto.

Lo que faltaría para acabar de concluir el mundo -1 es porque es un mundo que no tiene fin, que al llegar al final, vuelva al principio pero creo que eso también lo dejaremos para otro post.

Muchas gracias por leer esta aventura y nos vemos en los siguientes post.