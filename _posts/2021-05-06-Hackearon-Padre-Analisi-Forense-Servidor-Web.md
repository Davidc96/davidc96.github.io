---
layout: post
author: David Cuadrado
tag: [Reverse Engineering, Videojuegos] Prestashop 1.4.4.1
title: "NOTPUB¡¡¡Hackearon a mi padre!!! Analisis Forense de la página web de su empresa"
---
Sinceramente no tenía intención de hacer este post pero el otro día a mi padre le hackearon la página web de su empresa y le añadieron a su sistema de ficheros, un archivo PHP bastante sospechoso bajo el nombre de framework.php. Pero antes de pasar al analisis de dicho fichero, vamos a contar como sucedieron las cosas, en pocas palabras vamos a hacer un analisis forense de este caso.

Para empezar a entender el contexto, mi padre tiene en Internet, una tienda donde el vende sus productos. Una mañana, al intentar entrar a su página web, los proveedores del servidor y del dominio le bloquearon el acceso. La explicación que le han dado es que se ha detectado actividad maliciosa dentro de su espacio reservado para su página web y que por ello han procedido bloquear el acceso a cualquiera que entre a su página web, con suerte, ya le han vuelto a habilitar el dominio y ya se puede acceder a su página web.

Después del trabajo realizado por los IR del proveedor de servicios, llega el turno de saber realmente ¿Que ha pasado?, ¿Como ha pasado?, ¿Cuando ha pasado? y ¿Por que ha pasado? Para ello, realizaré un análisis forense de lo sucedido a ver si puedo llegar a una conclusión.

Como no soy un experto en forense, seguramente me dejaré un montón de pruebas y técnicas que se pueden utilizar para este tipo de casos xD. No obstante, tengo alguna que otra noción de este tema y basicamente me preocuparé en revisar los logs y ver que ha pasado.

# Revisando los logs del Servidor Web

Para empezar a analizar, he pedido a mi padre que me facilitara el acceso al panel de control del servidor web para poder ver los logs. También he "entrevistado" a mi padre para que me contara un poco de detalles acerca de cuando cree que ha sucedido todo.

Según mi padre, el no pudo acceder a la página web el día 06-05-2021. También me ha comentado que la versión de Prestashop que utilizaba es la 1.4.4.1, según él es una version antigua de Prestashop, esto ya empieza a oler mal seguramente existan exploits de esta versión. Con suerte, palabras de él, que le hayan hackeado la página no le afecta en nada a su actividad comercial debido a que dicha página, era una versión antigua que había dejado en el servidor porque estabá haciendo la migración a una nueva versión de Prestashop y que lo que quedaba de la antigua página web era solamente restos, igualmente, vamos a analizar que es lo que ha pasado.

Al parecer, el servidor web tiene como panel de administración CPanel tal y como se muestra en la imagen:
![CPANEL_IMAGE]()

Hacía mucho tiempo que no entraba en un CPanel (el único que recuerdo era el de 000webhost y era por la época de 2006 mas o menos xD), lo importante aquí es localizar los logs, en este caso me interesan:
* Logs de la página Web
* Logs de la cuenta FTP
* Algún que otro log que podría interesarme

Dentro del CPanel hay una opción que dice "Archivos de log sin procesar", voy a ir a esa página y me voy a descargar dichos logs sin procesar a ver que son.
![LOG_WINDOW]()

Al parecer hay 2, uno para el dominio en si y el otro para el mismo dominio pero SSL, nosotros descargamos ambos a ver que encontramos. También por la zona tenemos los logs FTP, abrá que hecharles un ojo, no vaya a ser que hayan entrado por ahí.

Después de revisarlos con detalle, no veo ninguna correlación entre los logs ni tampoco veo que se haya explotado ninguna vulnerabilidad. También puede ser que el momento en que realizaron el ataque, estos logs se perdieran, en todo caso parece que de momento estoy en un camino sin salida.

# Revisando los ficheros entregados por los proveedores del servidor web

Después de intercambiar varios mensajes con los proveedores del servicio de hosting, finalmente me han dado el fichero que había causado que la página web de mi padre se bloqueara. En este caso, me han dado 2 archivos, un fichero error_log y un archivo php llamado framework.php
![FILES_PROVIDED]()

Al revisar el error_log, veo que el script framework.php se ha ejecutado varias veces pero el primer día fue el 3 de Mayo de 2021 a las 2 de la mañana.
Después de esas 2 de la mañana, a las 16:00 se ha vuelto a ejecutar y se ha continuado ejecutando hasta las 21:03 del mismo 3 de Mayo.

Al día siguiente a las 21:34 ha vuelto a aparecer los errores hasta las 22:36:24, al finalizar, el día 5 de Mayo de 2021, volvieron los errores a partir de las 04:33:05 hasta las 04:39:56. Después de esto, no ha habido mas actividad. El motivo de que no haya mas actividad seguramente es debido a que IR bloqueo el acceso a ese fichero.
![ERROR_LOG]()

Al parecer, el error viene del fichero framework.php por lo que se va a proceder a analizarse.

# Ingenieria inversa del fichero framework.php: Analizando el Stager

Al abrir el fichero framework.php podemos observar lo siguiente:

Vaya, parece que este fichero para nada tiene pinta de ser malicioso xD (Cumple todo el manual 101 de como saber si un PHP es malicioso o no)
Al analizarlo, se puede observar que se ejecutan 3 funciones, si vamos a la documentación de PHP veremos que hacen esas 3 funciones:
* error_reporting(0); -> No imprime ningún error ni warning si estos se producen
* set_time_limit(150); -> Limita el tiempo máximo de ejecución en segundos
* ignore_user_abort(true); -> Evita que el script se aborte si el usuario sale de la página

Si seguimos analizando el código tenemos una función llamada abort() que si el usuario envia por GET el parametro remove, hace un unlink de la variable $name. La función unlink() borra el fichero.

Segumos analizando y vemos que hace un register_shutdown_function() para ejecutar la función abort cuando se quiera abortar el script y si como parametro se envia el parametro check, se cierra el script borrando cualquier archivo.

Después de ello vemos dos variables, g y b, cuyo significado si concatenamos los scripts llaman a la función gzinflate() y base64_decode() y para finalizar llama al metodo eval() con las funciones g() y b() para ejecutar un base64.

Al parecer este fichero tiene pinta de ser o un Stagger o un Dropper, estos ficheros se encargan de simplemente ejecutar el malware. En pocas palabras el verdadero bicho se encuentra dentro de ese Base64.

# Ingenieria inversa del fichero framework.php: Analizando el Malware
Al parecer primero se decodifica con base64 y luego se descomprime con gzinflate(), al hacer eso, podemos conseguir el siguiente PHP.
![MALWARE]()

Quien logre entender esto así directamente, le voy a dar un premio porque no se entiende nada xD. He intentado maquillarlo un poco para ver si puedo sacar algo pero ni con esas me lo ha podido dar en un formato correcto, parece ser que tendré que hacerlo bonito manualmente.

Para hacerlo bonito, lo que he estado haciendo es buscar ; y { para añadir un espacio, también he localizado los ifs y los endifs he ido indentadndo uno a uno. Para que veais el curro que me he pegado teneis la siguiente imagen con todo el código identado.
![IDENTED_CODE]()

¡El código en total son unas XXX líneas! Vale, no nos pongamos nerviosos y vamos a entender que es todo esto.

Al parecer existen 2 funciones clave para todo esto, estas 2 funciones son _kr() y _tt(). Empezaré por la _tt() ya que es la mas senzilla. Esta función crea un array de base64s con información como se puede ver en la siguiente imagen:
![FUNCTION_K]()

He intentado decodificar todo el base64 que tiene ese array pero parece ser que no arroja mucha información pero la clave está en la función $_k(). 
Esta función dado dos parametros, recupera la información del array de base64s y parece ser que la desencripta con XOR para luego devolverla.
Es más para entender que esta pasando, voy a utilizar un compilador online de PHP, voy a pegar el array de base64s y la función que supuestamente desencripta y veremos que hace.

En este caso, como ejemplo voy a elegir la línea 21 que hace un ini_set() donde en uno de sus parametros llama a _kr('_0','_1') y ¡voila! ¡Aparece algo entendible!
![FUNCTION_KR_WORK]()

Es decir, podemos afirmar que el parametro $_cmc es el base64 que te interesa y el parametro $_tic es la clave para descifrarlo. Con esto en mente, si conseguimos todas las claves podemos descifrar todo lo que hay en el array de base64s.