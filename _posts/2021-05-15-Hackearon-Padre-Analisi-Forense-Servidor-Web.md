---
layout: post
author: David Cuadrado
tag: [Reverse Engineering, Web]
title: "¡¡¡Hackearon a mi padre!!! Analisis Forense y RE de un malware escrito en PHP"
---
![PORTADA](https://davidc96.github.io/assets/images/posts/AFPWE/portada.jpg?style=centerme)

Sinceramente no tenía intención de hacer este post pero el otro día a mi padre le hackearon la página web de su empresa y le añadieron a su sistema de ficheros, un archivo PHP bastante sospechoso bajo el nombre de framework.php. Pero antes de pasar al analisis de dicho fichero, vamos a contar como sucedieron las cosas, en pocas palabras vamos a hacer un analisis forense de este caso.

Para empezar a entender el contexto, mi padre tiene en Internet, una tienda donde el vende sus productos. Una mañana, al intentar entrar a su página web, los proveedores del servidor y del dominio le bloquearon el acceso. La explicación que le han dado es que se ha detectado actividad maliciosa dentro de su espacio reservado para su página web y que por ello han procedido bloquear el acceso a cualquiera que entre a su página web, con suerte, ya le han vuelto a habilitar el dominio y ya se puede acceder a su página web.

Después del trabajo realizado por los IR del proveedor de servicios, llega el turno de saber realmente ¿Que ha pasado?, ¿Como ha pasado?, ¿Cuando ha pasado? y ¿Por que ha pasado? Para ello, realizaré un análisis forense de lo sucedido a ver si puedo llegar a una conclusión.

Como no soy un experto en forense, seguramente me dejaré un montón de pruebas y técnicas que se pueden utilizar para este tipo de casos xD. No obstante, tengo alguna que otra noción de este tema y basicamente me preocuparé en revisar los logs y ver que ha pasado.

# Revisando los logs del Servidor Web

Para empezar a analizar, he pedido a mi padre que me facilitara el acceso al panel de control del servidor web para poder ver los logs. También he "entrevistado" a mi padre para que me contara un poco de detalles acerca de cuando cree que ha sucedido todo.

Según mi padre, el no pudo acceder a la página web el día 06-05-2021. También me ha comentado que la versión de Prestashop que utilizaba es la 1.4.4.1, según él es una version antigua de Prestashop, esto ya empieza a oler mal seguramente existan exploits de esta versión. Con suerte, palabras de él, que le hayan hackeado la página no le afecta en nada a su actividad comercial debido a que dicha página, era una versión antigua que había dejado en el servidor porque estabá haciendo la migración a una nueva versión de Prestashop y que lo que quedaba de la antigua página web era solamente restos, igualmente, vamos a analizar que es lo que ha pasado.

Al parecer, el servidor web tiene como panel de administración CPanel tal y como se muestra en la imagen:
![CPANEL_IMAGE](https://davidc96.github.io/assets/images/posts/AFPWE/cpanel.JPG?style=centerme)

Hacía mucho tiempo que no entraba en un CPanel (el único que recuerdo era el de 000webhost y era por la época de 2006 mas o menos xD), lo importante aquí es localizar los logs, en este caso me interesan:
* Logs de la página Web
* Logs de la cuenta FTP
* Algún que otro log que podría interesarme

Dentro del CPanel hay una opción que dice "Archivos de log sin procesar", voy a ir a esa página y me voy a descargar dichos logs sin procesar a ver que son.
![LOG_WINDOW](https://davidc96.github.io/assets/images/posts/AFPWE/acceso_procesar.JPG?style=centerme)

Al parecer hay 2, uno para el dominio en si y el otro para el mismo dominio pero SSL, nosotros descargamos ambos a ver que encontramos. También por la zona tenemos los logs FTP, habrá que hecharles un ojo, no vaya a ser que hayan entrado por ahí.

Después de revisarlos con detalle, no veo ninguna correlación entre los logs ni tampoco veo que se haya explotado ninguna vulnerabilidad. También puede ser que el momento en que realizaron el ataque, estos logs se perdieran, en todo caso parece que de momento estoy en un camino sin salida.

# Revisando los ficheros entregados por los proveedores del servidor web

Después de intercambiar varios mensajes con los proveedores del servicio de hosting, finalmente me han dado el fichero que había causado que la página web de mi padre se bloqueara. En este caso, me han dado 2 archivos, un fichero error_log y un archivo php llamado framework.php
![FILES_PROVIDED](https://davidc96.github.io/assets/images/posts/AFPWE/files.JPG?style=centerme)

Al revisar el error_log, veo que el script framework.php se ha ejecutado varias veces pero el primer día fue el 3 de Mayo de 2021 a las 2 de la mañana.
Después de esas 2 de la mañana, a las 16:00 se ha vuelto a ejecutar y se ha continuado ejecutando hasta las 21:03 del mismo 3 de Mayo.

Al día siguiente a las 21:34 ha vuelto a aparecer los errores hasta las 22:36:24, al finalizar, el día 5 de Mayo de 2021, volvieron los errores a partir de las 04:33:05 hasta las 04:39:56. Después de esto, no ha habido mas actividad. El motivo de que no haya mas actividad seguramente es debido a que IR bloqueo el acceso a ese fichero.
![ERROR_LOG](https://davidc96.github.io/assets/images/posts/AFPWE/error_log.JPG?style=centerme)

Al parecer, el error viene del fichero framework.php por lo que se va a proceder a analizarse.

# Ingenieria inversa del fichero framework.php: Analizando el Stager

Al abrir el fichero framework.php podemos observar lo siguiente:
![FRAMEWORK_PHP](https://davidc96.github.io/assets/images/posts/AFPWE/framework_php.JPG?style=centerme)
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
![MALWARE](https://davidc96.github.io/assets/images/posts/AFPWE/malware.php.JPG?style=centerme)

Quien logre entender esto así directamente, le voy a dar un premio porque no se entiende nada xD. He intentado maquillarlo un poco para ver si puedo sacar algo pero ni con esas me lo ha podido dar en un formato correcto, parece ser que tendré que hacerlo bonito manualmente.

Para hacerlo bonito, lo que he estado haciendo es buscar ; y { para añadir un espacio, también he localizado los ifs y los endifs he ido indentando uno a uno y al final... ¡El código en total son unas 681 líneas! Vale, no nos pongamos nerviosos y vamos a entender que es todo esto.

Al parecer existen 2 funciones clave para todo esto, estas 2 funciones son _kr() y _tt(). Empezaré por la _tt() ya que es la mas senzilla. Esta función crea un array de base64s con información como se puede ver en la siguiente imagen:
![FUNCTION_K](https://davidc96.github.io/assets/images/posts/AFPWE/fun_kr.JPG?style=centerme)

He intentado decodificar todo el base64 que tiene ese array pero parece ser que no arroja mucha información pero la clave está en la función $_k(). 
Esta función dado dos parametros, recupera la información del array de base64s y parece ser que la desencripta con XOR para luego devolverla.
Es más para entender que esta pasando, voy a utilizar un compilador online de PHP, voy a pegar el array de base64s y la función que supuestamente desencripta y veremos que hace.

En este caso, como ejemplo voy a elegir la línea 21 que hace un ini_set() donde en uno de sus parametros llama a _kr('_0','_1') y ¡voila! ¡Aparece algo entendible!
![FUNCTION_KR_WORK](https://davidc96.github.io/assets/images/posts/AFPWE/funct_kr_decr.JPG?style=centerme)

Es decir, podemos afirmar que el parametro $_cmc es el base64 que te interesa y el parametro $_tic es la clave para descifrarlo. Con esto en mente, si conseguimos todas las claves podemos descifrar todo lo que hay en el array de base64s.

Para ello voy a utilizar Linux y grep para poder obtener todas las llamadas a la función _kr. El comando que he utilizado para grepear todas las funciones kr es el siguiente:

```sh
$ cat payload_beatufied.php | grep -o -E "kr\(([^)]+)\)" > allKr
```

El comando aparte de obtener el resultado que queremos, lo guarda en un fichero llamado allKr
![GREP_KR](https://davidc96.github.io/assets/images/posts/AFPWE/grep_kr.JPG?style=centerme)

Una vez realizado esto, vamos a un compilador PHP online (como por ejemplo Paiza.io) y pegamos ahí, el array con todos los base64, la función de cifrado y todas las funciones kr que hemos obtenido con anterioridad:
![B64_DECODE](https://davidc96.github.io/assets/images/posts/AFPWE/b64_decoded.JPG?style=centerme)

De todos los strings que hay, tenemos desde cabeceras HTTP, cabeceras SMTP y URLs:
```
b.barracudacentral.org
xbl.spamhaus.org
sbl.spamhaus.org
zen.spamhaus.org
bl.spamcop.net
```

Lo que se me ocurre ahora mismo es crear un script que reemplaze las funciones kr con sus respectivos outputs sin base64 para poder entender mejor el código. El script en Python en cuestión es el siguiente:
```python
php_payload = ""
output = []
all_functs = []
funct_replace_dict = {}

with open("payload_beatufied.php", "r") as f:
    php_payload = f.read()

with open("output_b64.txt", "r") as f:
    output = f.readlines()

with open("allFuncts.txt", "r") as f:
    all_functs = f.readlines()

counter = 0
result = ""
# Replace the kr functions to his outputs
for out in output:
    out = out.replace('\n', '')
    st = out.split("result: ")[1]
    php_payload = php_payload.replace("_t::_"+all_functs[counter].replace("\n","").replace(";",""), "'" + st + "'")
    counter+=1

# Write to file the cleanest PHP file without kr functions
with open("clean.php","w") as f:
    f.write(php_payload)
```

Al ver de nuevo el código en PHP, ya podemos deducir varias cosas interesantes:
![MALWARE_MAIL](https://davidc96.github.io/assets/images/posts/AFPWE/malware_mail.jpg?style=centerme)

En este fragmento de código, podemos observar que de alguna manera se está construyendo un mail. Si observamos mas adelante en el código:
![MALWARE_MAIL_ATTACHMENT](https://davidc96.github.io/assets/images/posts/AFPWE/malware_mail_attachment.jpg?style=centerme)

Al parecer también incluye una especie de C2C para poder pedirle al script que es lo que quieres hacer. En la siguiente tabla se detalla que es cada comando dentro de este fichero:

|Comando| Explicación                                                                          |
|:------|--------------------------------------------------------------------------------------|
|u      | Al parecer te desubscribes de la newsletter y lo guarda en un fichero de log         |
|lu     | Imprime el fichero de log por pantalla                                               |
|ce     | Al parecer hace un parse_string para guardarlo dentro del POST                       |
|ch     | Envia un correo para probar si el servidor soporta el envío de correos               |
|sn     | Envía el correo de SPAM                                                              |

Con toda esta información, creo que ya puedo buscar en San Google si existe alguna campaña de malware en PHP que envía correos y efectivamente encontré un artículo del portal TrendMicro (Link: <a href="https://www.trendmicro.com/en_us/research/19/i/spam-campaign-abuses-php-functions-for-persistence-uses-compromised-devices-for-evasion-and-intrusion.html" >https://www.trendmicro.com/en_us/research/19/i/spam-campaign-abuses-php-functions-for-persistence-uses-compromised-devices-for-evasion-and-intrusion.html</a>).

Acorde a la página de TrendMicro, los atacantes para inyectarte este fichero en PHP lo que hacen es conectarse al servidor a través de credenciales débiles o ya comprometidas y lo ejecutan para poder hacer estas campañas de spam masivo. En nuestro caso no he podido ver en acción el malware porque el proveedor de servicios lo bloqueó a tiempo pero lo interesante del artículo es ver como actua el malware en acción, recomiendo echarle un vistazo porque es muy interesante.
![TRENDMICRO_ATK](https://davidc96.github.io/assets/images/posts/AFPWE/spam_attack.png?style=centerme)

Por último, ya teniendo bastante claro que ese fichero es un spammer en PHP, existe un fichero llamado logsubsc.log dentro del código así que decidí buscar ese fichero en Google a ver si me salía un poco más de información acerca del malware. Lo que encontré fue servidores infectados por este malware que Google había indexado. Para evitar problemas, he censurado las URL por si las moscas.
![AFFECTED_SERVERS](https://davidc96.github.io/assets/images/posts/AFPWE/infected_servers.JPG?style=centerme)

# Conclusiones
Si hacemos un recopilatorio de todo el post, empezariamos diciendo que a mi padre, le llego un correo del proveedor de servicios de su página web porque se ha detectado actividad maliciosa dentro de su servidor. Tras contactar con ellos, nos facilitaron el malware en cuestión para poder analizarlo.

El malware está formado por 2 partes:
- Un stager que se encarga de ofuscar y ocultar el verdadero malware a ojos de los antivirus y firewalls, este stager utiliza base64 y gzip para ocultar dicho malware y luego lo ejecuta con eval()
- El malware en sí que parece ser un archivo minificado al máximo pósible llegando al punto de que sus cadenas de texto están cifradas con una especie de cifrado XOR

Al desminificar el malware y descifrar sus cadenas de texto mediante varios scripts desarrollados durante el análisis, se ha llegado a la conclusión de que el malware en cuestión es un Spammer, es decir, un malware que básicamente se encargará de enviar correos spam a tu nombre o nombre de la empresa.
El atacante mediante un sistema de C2C, controlará dicho malware para poder enviar los correos.

Gracias al artículo de TrendMicro, se puede saber que este tipo de malwares infectan tus servidores debido a credenciales débiles o ya comprometidas.
Con esto concluimos el post, espero que os haya parecido interesante y nos vemos en el siguiente artículo.

# Materiales
Como regalo al haber llegado al final, teneis en la ruta /assets/samples/PHPRevEng, en mi repositorio, el malware en cuestión y los diferentes scripts y txts que he ido utilizando en un zip con la contraseña "infected". Utilizar este tipo de malware con fines educativos y no para dañar a nadie porque estareis cometiendo un delito. Dicho esto, me despido.

Gracias por todo, que tengais un buen día.