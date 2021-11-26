---
layout: post
author: David Cuadrado
tag: Hardware Hacking
title: "Proyecto Evil Hidden Router: Creando un router oculto para pruebas de penetración Wireless"
---
![PORTADA](https://davidc96.github.io/assets/images/posts/PHR/portada.jpg?style=centerme)
Hace mucho tiempo que no publico nada en el blog pero esta vez quiero enseñaros un proyecto que he estado haciendo durante estos días. La motivación de este proyecto viene debido a una
charla que se realizó en una conferencia online en 2020 (No recuerdo bien que conferencia era) que hablaban sobre las Drop boxes para equipos de Red Team.

Una Drop box por decirlo así es el hecho de ocultar algo malintencionado dentro de un objeto cotidiano con el fin de que pase desapercibido a ojos de cualquier persona. Los mas conocidos
serían los Rubber Ducky que aparentan ser una USB normal y corriente, pero en realidad es un HID que actua como teclado para poder ejecutar comandos.

![RUBBER_DUCKY](https://davidc96.github.io/assets/images/posts/PHR/rubber_ducky.jpg?style=centerme)

También podemos encontrar teclados modificados o Keyloggers que incorporan un pequeño chip con la finalidad de capturar todo lo que escribe la víctima. Conociendo la marca mas popular en una empresa, se puede llegar a crear ese tipo de teclados y pasarían bastante desapercibidos.

![HIDDEN_KEYBOARD](https://davidc96.github.io/assets/images/posts/PHR/keylogger.jpg?style=centerme)

Pero no todo se resume a modificaciones hardware o tratar de realizar ataques a nivel de capa física. Con la llegada de los miniordenadores, es posible construir proyectos que cumplan los dos requisitos mencionados: Que sirva para hacer auditorias y que pase desapercibido a ojos de cualquiera.

## Creando el Drop Box

Nuestro Drop Box consiste en ocultar una Raspberry Pi dentro de un libro con la finalidad de hacer auditorias Wifi. Para ello utilizaremos la nueva Raspberry Pi Zero 2 W que promete tener una potencia equivalente a la Raspberry Pi 3. También necesitaré un pincho Wifi compatible para hacer inyección de paquetes y auditorias Wireless, para ello utilizaré un TL-722N v1 (De momento solo lo he probado con la versión 1, pero creo que es posible también hacerlo funcionar con la v3 de la tarjeta Wifi, eso si en la v2 no funciona)

Para la caja, como me apena romper un libro (Por si acaso lo quiero leer xD) he decidido buscar por AliExpress a ver si existian cajas en forma de libro decoratívas y tal, y al final encontré una que me convenció, eso si, para ahorrarme problemas al montar la caja, busqué una ya montada aunque tenga que pagar un poquito más.

En resumen el material necesario és:
- Una Raspberry Pi Zero 2 W
- Un adaptador de micro USB a USB
- Una tarjeta de Wifi USB, en mi caso uso la TL-722N v1 pero podeis utilizar cualquier otra compatible con auditorias Wireless
- Una batería externa, en mi caso la compré en el Decathlon xD
- Una MicroSD
- Celo para pegar los componentes a la caja

![MATERIALS](https://davidc96.github.io/assets/images/posts/PHR/materials.jpg?style=centerme)

El sistema operativo que voy a utilizar va a ser el Raspberry OS o Raspbian y utilizaré el repositorio de <a href="https://github.com/v1s1t0r1sh3r3/airgeddon">Airgeddon</a> para poder automatizar los ataques a redes Wifi.

Para acceder a la caja, utilizaremos la interfaz Wifi que viene por defecto en la Raspberry, lo que haremmos es asociarla a un AP creado por nosotros (ya sea creada por un móvil o creada por un router nuestro) y nos conectaremos por SSH.

## Instalando el software necesario

Una vez quemado el sistema operativo en nuestra MicroSD, primero configuramos la red wifi que queremos que se conecte, para eso lanzamos el comando:

```sudo raspi-config```

Una vez configurado la red wifi procedemos a actualizar repositorios y el sistema operativo

```sudo apt update & sudo apt upgrade -y```

Una vez actualizado instalamos git

```sudo apt install git```

Y por último descargamos airgeddon directamente de los repositorios

```git clone https://github.com/v1s1t0r1sh3r3/airgeddon```

Cuando ejecutemos por primera vez airgeddon, nos dirá que nos faltan mucho de los paquetes para poder ejecutar la herramienta. La mayoría de ellos son bastante triviales y fáciles de instalar con apt install nombre_del_paquete pero hay 4 cuya instalación no es tan trivial, estos son:

- ettercap
- bettercap
- hostapd-wpe
- asleap
- Drivers de la TL-722N

### Instalando ettercap
Este sin lugar a dudas es el mas fácil de todos, tan solo tienes que ejecutar este comando:

```sudo apt install ettercap-text-only```

Y ya lo tendríamos instalado

### Instalando bettercap
Para bettercap, he seguido la <a href="">guía</a> de FideliusFalcon y a modo resumen esto son los pasos que hay que realizar:

1. Instalar Golang ```sudo apt install golang```
2. Abrimos el fichero .profile con nano y añadimos las siguientes lineas

```sh
export GOPATH=$HOME/go  
export PATH=$PATH:$GOROOT/bin:$GOPATH/bi
```

Luego hacemos un source a .profile

3. Instalamos las dependencias

``` sudo apt install build-essential libpcap-dev libusb-1.0-0-dev libnetfilter-queue-dev```

4. Descargamos bettercap con go (Suele tardar un poco dependiendo de la velocidad de Internet y no muestra output)
``` go get github.com/bettercap/bettercap ```

5. Y procedemos a instalarlo

```sh
cd $GOPATH/src/github.com/bettercap/bettercap
make build
sudo make install
```

Con todo esto, ya tendríamos instalado el bettercap

### Instalando asleap

Para instalar asleap, me he basado en un script que encontré en github pero básicamente lo que hay que hacer es:

1. Descargar lxcrypt y lo instalamos

```sh
wget http://http.us.debian.org/debian/pool/main/libx/libxcrypt/libxcrypt1_2.4-3.1_armhf.deb
wget http://http.us.debian.org/debian/pool/main/libx/libxcrypt/libxcrypt-dev_2.4-3.1_armhf.deb
dpkg -i libxcrypt1_2.4-3.1_armhf.deb libxcrypt-dev_2.4-3.1_armhf.deb
```

2. Clonamos el repositorio, compilamos y lo instalamos

```sh
git clone https://github.com/joswr1ght/asleap
cd asleap
make
sudo cp asleap /usr/bin/asleap
```

Con esto ya tendríamos asleap instalado

### Instalando hostapd-wpe
Para hostapd-wpe, lo que hay que hacer es descargarse un parche disponible en Github que sirve para hacer credential harvesting entre otras cosas.

1. Para ello primero instalamos las dependencias de hostapd

```sudo apt install libssl1.0-dev libnl-genl-3-dev libnl-3-dev pkg-config libsqlite3-dev```

2. Descargamos el parche de Github

```git clone https://github.com/OpenSecurityResearch/hostapd-wpe```

3. Descargamos el código fuente de hostapd versión 2.9 (Desconozco si funciona para versiones posteriores) y lo descomprimimos
```sh
wget https://w1.fi/releases/hostapd-2.9.tar.gz
tar -zxf hostapd-2.9.tar.gz
```
4. Aplicamos el parche y compilamos el proyecto
```sh
cd hostapd-2.9
patch -p1 < ../hostapd-wpe/hostapd-wpe.patch
cd hostapd
make
sudo make install
```

Con todo esto ya tenemos el software necesario para hacer funcionar airgeddon, si es necesario ejecutamos de nuevo el script de airgeddon para ver si falta algo.

### Instalando el driver TL-722N
El último punto, pero no menos importante, es instalar el driver para la tarjeta Wifi. Para ello se ejecutan los siguientes comandos
```sh
git clone https://github.com/lwfinger/rtl8188eu.git
cd rtl8188eu
make
sudo make install
sudo modprobe 8188eu
```

Luego habría que hacer un reboot para que el driver funcione correctamente

### Últimos retoques

Para acabar, lo importante es activar el servicio de SSH para poder conectarnos de manera remota a la caja y al entrar, ejecutar automáticamente Airgeddon.
Para el primer punto es tan fácil como ejecutar el comando ```sudo systemctl enable ssh``` para así activar el servicio SSH cuando se haga reboot.

El segundo punto, sería editar el fichero .bashrc para que al final del todo, ejecutaramos el comando ```sudo bash /home/pi/airgeddon/airgeddon.sh```

Con esto ya tendríamos el software listo

## Acomodando todo en la caja

La ventaja es que la caja es lo suficientemente grande para que entre todo, yo he ido pegando los materiales en plan cutre con celo para que no se caigan pero lo suyo es utilizar algun pegamento o algo.

Mi distribución es la siguiente

![ALL_CONNECTED](https://davidc96.github.io/assets/images/posts/PHR/all_connected.jpg?style=centerme)

Como podeis ver, he intentado acomodarlo como he podido aprovechando los espacios y pegando con celo los cables para que no molestaran al cerrar la caja

Una vez hecho esto, si hacemos la puesta en marcha este sería el resultado

![FINAL_RESULT](https://davidc96.github.io/assets/images/posts/PHR/final_result.jpg?style=centerme)

## Conclusiones

Como conclusiones, la verdad que ha sido un proyecto divertido aprovechando la nueva Raspberry Pi Zero 2 W que ha salido recientemente por el precio de 16 €. Como puntos de mejora para este proyecto sería el poder conectarse a la caja mediante Bluetooth, permitiendo así poder liberar la tarjeta de red integrada y poder utilizarla también para auditar.

Muchas gracias por leer este post y nos vemos en el siguiente.
