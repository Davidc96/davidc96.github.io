---
layout: post
author: David Cuadrado
tag: [Reverse Engineering, Videojuegos]
title: "Destripando el glitch del Mundo -1 de Super Mario Bros: Ingenieria Inversa: Estudiando el Glitch a bajo nivel (Parte 2)"
---

![MUNDO_MENOS_1_PORTADA](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/NESAsm.png?style=centerme)
En la <a href="https://davidc96.github.io/2021/05/03/Destripando-Super-Mario-Bros-Introduccion.html">primera parte</a>, pudimos descubrir porque el mundo se llama mundo -1 y esto es debido a que realmente no estamos en el mundo -1, si no en el mundo 36-1 (24h-1) siendo el número 36 un espacio en blanco si lo llegamos a convertir en un caracter. Ahora bien, para demostrar todo esto, lo que vamos a hacer es debuggear el juego con FCEUX e investigaremos que es lo que pasa cuando entramos en la tubería hacia el mundo -1.

# FCEUX El debugger por excelencia para NES
Antes de entrar en materia, hay que conocer un poco el debugger que vamos a utilizar. Al abrirlo nos aparecerá una pantalla como la siguiente:
![FCEUX_DEBUGGER](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/FCEUXDebugger.jpg?style=centerme)

En la parte izquierda del Debugger tenemos el código fuente del juego escrito en Assembler 6502.

Luego un poco más a la derecha tenemos 7 botones:</br>
-   Run: Vuelve a ejecutar el programa</br>
-	Step Into: Ejecuta la siguiente instrucción</br>
-	Step Out: Ejecuta la anterior instrucción</br>
-	Step Over: Ejecuta hasta encontrar un RTS (return de una subrutina)</br>
-	Run Line: Ejecuta la línea actual</br>
-	128 Lines: Ejecuta 128 líneas</br>
-	SeekTo: Hace un Jump a una dirección de memoria y ejecuta esa línea.</br>

Después tenemos el PC, el PC es el Personal Computer (es broma) Program Counter, un registro que te indica la dirección de memoria de la instrucción que se va a ejecutar.l uego tenemos los valores de los registros A X e Y junto con la Pila de valores Stack.

Por último en la parte derecha del debugger tenemos los Breakpoints. Un BreakPoint como bien dice su nombre es un punto de rotura de código, es decir, es un punto donde el programa siempre se parará. Los Breakpoints siempre se ponen en las instrucciones del programa y sirven para ver el estado de los registros y la pila dicho de otra manera es como si hicieramos una fotografía del programa en ese instante de tiempo.
Existen 3 tipos de Breakpoints:
-	BreakPoint Ejecución: El programa se parará cuando se está a punto de ejecutar la instrucción indicada por el BreakPoint. </br>
-	BreakPoint Lectura: El programa se parará cuando haya una lectura en una posición memoria específica.</br>
-	BreakPoint Escritura: El programa se parará cuando haya una escritura en una posición memoria específica.</br>

Otra ventaja que aporta los Breakpoints es que podemos ejecutar el código línea por línea de manera manual para ver que va sucediendo en cada momento.

# Analizando de manera dinámica el Glitch con FCEUX

Después de haber hecho una pequeña introducción a la herramienta, vamos a comenzar la parte interesante. Primero de todo colocaremos un Breakpoint de solo lectura en la dirección de memoria $87F2. El mótivo de esto es porque en esa dirección, se encuentra el array anterior de las Warpzones (warpzoneNumber[3][3]). El breakpoint que colocaremos será de tipo lectura porque nos interesa saber en que momento el juego necesita de ese array.

Al pasar por la zona, hacer el WalkThrough Walls y entrar en la primera tubería, podemos observar que el Breakpoint ha saltado con el siguiente fragmento de código.
```
01:DF0C:AD D6 06    LDA $06D6 = #$01
01:DF0F:F0 39       BEQ $DF4A
....
01:DF16:A5 86    	LDA $0086 = #$28
01:DF18:C9 60     	CMP #$60
01:DF1A:90 06     	BCC $DF22
01:DF1C:E8        	INX
01:DF1D:C9 A0    	CMP #$A0
01:DF1F:90 01     	BCC $DF22
01:DF21:E8        	INX
01:DF22:BC F2 87  	LDY $87F2,X @ $87F7 = #$05 <--- BREAK
01:DF25:88        	DEY
01:DF26:8C 5F 07  	STY $075F = #$00
01:DF29:BE B4 9C  	LDX $9CB4,Y @ $9CB9 = #$17
01:DF2C:BD BC 9C    LDA $9CBC,X @ $9CC1 = #$28
01:DF2F:8D 50 07  	STA $0750 = #$C2
01:DF32:A9 80     	LDA #$80
....
```

Sin lugar a dudas la línea interesante en este momento es la $01DF22 ya que lo que realiza es un LDY, es decir, escribe en el registro Y lo que haya en $87F2 cuyo índice es X. Sería el equivalente a hacer esto en pseudocódigo:
```C++
REG_Y = warpzoneNumber[X];
```

En estos momentos, el valor de X es 4, por lo que vamos a hacer es con un lápiz y papel, vamos a deducir que valor del Array tiene índice 4

|:------|---------|-------|---------|-------|---------|-------|-------|---------|-------|--------|----------|
|0      |1        |2      |3        |4      |5        |6      |7      |8        |9      |10      |11        |
|:------|---------|-------|---------|-------|---------|-------|-------|---------|-------|--------|----------|
|04     |03       |02     |00       |24     |05       |24     |00     |08       |07     |06      |00        |

Como podemos observar cuando X es 4, el valor del array es 24 por lo que podemos confirmar de que sí, la Warpzone que se ejecuta al acceder al mundo -1 es la que está en 4-2 con una sola tubería.
![REMEMBER_PIPE_4-2](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/WarpZone42.jpg?style=centerme)

Es más si vamos con cuidado a la segunda tubería, podemos observar que iremos al mundo 5-1.

En este momento, sale una duda: ¿Porque X vale 4? Para ello, vamos a colocar un Breakpoint en la instrucción $01DF0C. Al hacer esto y entrar a la primera tubería, vemos que en $06D6 hay un 1 y luego de ello vemos que ese 1 se va convirtiendo en un 4 hasta llegar a la instrucción LDY. Bien, vamos a ver de donde sale ese 1 inicial.

Al parecer el culpable de que se ponga un 1 es concretamente el siguiente bloque que marco en la imagen.
![NUMBER_1_BLOCK](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/Block1.jpg?style=centerme)

Sinceramente tendriamos que investigar mas el mótivo de porque se pone ese valor a 1, para ello, vamos a intentar entender el código fuente gracias a un proyecto en Github llamado <a href="https://gist.github.com/1wErt3r/4048722">A Comprehensive Super Mario Bros Disassembly</a>

Este repositorio contiene una versión decompilada de Super Mario Bros NES y nos facilita mucho el análisis estático del juego debido a que explica el código muy bien. En este repositorio vamos a buscar el objeto WarpZoneControl cuya dirección es la $0626. Ahora nuestro objetivo es encontrar el mismo fragmento de código que inserta un 1 en WarpZoneControl cuando pasamos por el bloque anterior.
Vamos a entender que está pasando aquí gracias al código en Github:
```asm
WarpZoneObject:
      lda ScrollLock         ;check for scroll lock flag
      beq ExGTimer           ;branch if not set to leave
      lda Player_Y_Position  ;check to see if player's vertical coordinate has
      and Player_Y_HighPos   ;same bits set as in vertical high byte (why?)
      bne ExGTimer           ;if so, branch to leave
      sta ScrollLock         ;otherwise nullify scroll lock flag
      inc WarpZoneControl    ;increment warp zone flag to make warp pipes for warp zone
      jmp EraseEnemyObject   ;kill this object
```
Lo primero que hace es comprobar si ya no se puede hacer más Scroll, en pocas palabras, si estamos al final de la pantalla. Como ese es el caso y encima estamos en las coordenadas que tocan, creamos la WarpZone y eliminamos ese flag.

En pocas palabras lo primero que hacemos es comprobar si estamos al final de pantalla al pisar ese bloque, como es el caso el juego añade una extensión más de mapa para crear la WarpZone y una vez hecho todo esto, elimina ese bloque para no volver a crear de nuevo dicha Zona. Resumido en imagen quedaría algo así:
![DIAGRAM_WZ](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/DiagramWZ.jpg?style=centerme)

Para poder acceder de manera legal a la nueva zona creada por el juego, hay que pasar sí o sí por arriba del mapa, existe un bloque que al pasar por encima de él, cambia el valor de WarpZoneControl a 4. Ese valor 4 sirve para poder indicarle a la nueva zona, que WarpZone es.
![WARPZONE_12](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/WarpZone12.jpg?style=centerme)

Al pasar por ese bloque, podemos observar que se imprimen por pantalla los diferentes números correspondientes a la WarpZone, por detrás, también se ha asignado el valor 4. El motivo del valor 4 es debido a que el indice está enmascarado bajo la siguiente fórmula:
```
INDEX = (WARPZONECONTROL & 3)* 4;
```
En pocas palabras, si tomamos como ejemplo el valor 4 en WARPZONECONTROL al hacer la operación sería tal que:
```
REG_A = (WARPZONECONTROL & 011b) = 100b & 011b = 000b
INDEX = REG_A * 4 = 000b * 4 = 0
```
El INDEX = 0, corresponde al array de la WarpZone de 1-2. En función de la tubería que entres, la de la izquierda, la del centro o la de la derecha, se incrementa el INDEX para escoger uno u otro mundo.

Ahora bien, como bien he dicho, el juego espera a que tu para acceder a esa WarpZone, si o si tengas que pasar por el bloque anterior. ¿Que pasa si no es el caso? ¿Que pasa si vamos directamente a la WarpZone sin pasar por ese bloque? Gente, ahí esta la mágia del Glitch del Walkthrough Walls.
![GLITCH_WALKTHROUGH_WALLS](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/WTWGif.gif?style=centerme)

Recordemos que para hacer ese Glitch, sí o sí tenemos que pasar por ese bloque que crea la WarpZone, recordemos que ese bloque pone el valor de WarpZoneControl a 1 para que el juego pueda crear la WarpZone. El problema es que al hacer el Glitch, llegaremos antes a la tubería que al bloque que inserta el valor WarpZoneControl a 4. Por lo tanto, si entramos en la primera tubería y hacemos la anterior operación matematica pero con el valor 1 en vez de 4:
```
REG_A = (WARPZONECONTROL & 011b) = 001b & 011b = 001b
INDEX = REG_A * 4 = 001b * 4 = 100b 
```

¿Que WarpZone corresponde al INDEX = 4? ¡Efectivamente! ¡La WarpZone de 4-2 con una tubería! En pocas palabras, con valor de WarpZoneControl = 1, estamos cargando la WarpZone de 4-2 que solamente tiene una tubería.

En modo resumen si lo hicieramos de manera legal, esto es lo que está pasando
![DIAGRAM_WZ_EXP](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/DiagramWZExp.jpg?style=centerme)

Pero al hacer el Glitch de Walkthrough Walls sucede esto:
![DIAGRAM_WZ_UN](https://davidc96.github.io/assets/images/posts/DSMB/SecondPart/DiagramWZUn.jpg?style=centerme)

Pregunta: ¿Porque el valor de WarpzoneControl en un primer momento es 1? Bien, esto es debido al propio check de las tuberías verticales y cuando Mario entra a ellas.

Cuando Mario entra a una de ellas, el juego siempre verifica si la tubería que está entrando pertenece a una WarpZone o no. Para saberlo hay que fijarse en estas 2 líneas situadas justo al principio de la función:
```
01:DF0C:AD D6 06    LDA $06D6 = #$01
01:DF0F:F0 39       BEQ $DF4A
```

LDA hace una cosa especial mas aparte de cargar un valor a un registro y es setear el flag Z:
- Si el valor que carga al registro A es más grande de 0: Z = 1
- Si el valor que carga al registro A es igual a 0: Z = 0

BEQ saltará si Z = 0, entonces como en A al ejecutar el LDA es 1, Z = 1 por lo tanto BEQ no se ejecutará.

Como curiosidad, al poner el WarpZoneControl a 1, estamos diciendo que cualquier tubería vertical que Mario encuentre, formará parte de la Warpzone. Un poco bizarro pero bueno xD

# Conclusiones
Como conclusión a este post, podemos decir que para cargar una Warpzone es necesario pasar por 2 puntos estratégicos:
- El primer punto sirve para decirle al juego que existe una Warpzone en la zona, por lo tanto añade esa nueva extensión.
- El segundo punto sirve para indicarle al juego, que Warpzone hay que cargar, eso se hace mediante un índice enmascarado que se revela al utilizar una fórmula

Para que la Warpzone funcione correctamente, hay que pasar por los 2 bloques antes de llegar a las tuberías, pero, el Glitch the Walkthrough Walls nos permite pasar unicamente por 1 lo que provoca que el juego piense que estamos en la Warpzone de 4-2 y no la de 1-2.

Con esto concluye la segunda parte del glitch, en la tercera parte, veremos porque el mundo -1 es un mundo acuatico y si podemos cambiar ese mundo acuatico por el mundo que queramos.

Muchas gracias por leerme y nos vemos en el siguiente post.



