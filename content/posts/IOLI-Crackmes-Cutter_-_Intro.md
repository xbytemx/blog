---
title: "Reversing de IOLI Crackmes con Cutter - Consiguiendo Cutter y los crackmes de IOLI"
date: 2018-10-18T13:32:59-05:00
draft: false
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
---

Ok, esta es una pequeña guía de como resolver los retos crackme de IOLI con cutter. La verdad me he divertido mucho usando este GUI de radare2 y creo que merece la pena que veamos más como explotar la herramienta.

Lo primero es conseguir la herramienta y los crackmes
<!--more-->

Lo primero es conseguir la herramienta y los crackmes, esto es bastante sencillo, solo accedemos a las siguientes urls:

Cutter: https://github.com/radareorg/cutter/releases

IOLI Crackmes: https://dustri.org/b/files/IOLI-crackme.tar.gz http://pof.eslack.org/tmp/IOLI-crackme.tar.gz

Yo he utilizado la versión 1.7.2 de Cutter, pero con cualquier versión posterior no deberían tener problemas.

Para los crackme, estaré usando la versión de bin-linux, ya que mi estación de trabajo esta montada sobre Debian GNU/Linux. Les recomiendo bajar una Maquina Virtual para mantener sus herramientas al día y no mezclar con el sistema operativo principal.

Ok, una vez descargada la versión 1.7.2 de Linux.AppImage, solo debemos realizar el siguiente comando para darle permisos de ejecución y la ejecutamos:

```
2018-10-18 13:56:10 tony@portatil:~$ chmod +x Descargas/Cutter-v1.7.2-x86_64.Linux.AppImage
2018-10-18 13:56:11 tony@portatil:~$ Descargas/Cutter-v1.7.2-x86_64.Linux.AppImage
Setting r2 prefix = "/tmp/.mount_CutterwAGsLe/usr"  for AppImage.
```

Nos aparecerá una ventana en donde Cutter buscara cargar los archivos para analizarlos:

![Open-File](/img/cutter00/open-file.png)

Tenemos entre otras opciones, abrir un archivo, abrir un shellcode o abrir algún proyecto que anteriormente estemos trabajando (oh si, puedes continuar tu trabajo justo en donde lo dejaste). En la opción de abrir un archivo, también podemos seleccionar el input, es decir, podemos usar Cutter con un debugger remoto o desde algún sitio web, inclusive sobre tipos de archivo específicos como zip, permitiendo la misma flexibilidad que tiene radare2 a la hora de analizar.

En mi caso como he abierto otros ejecutables me aparece mi historial de otros archivos. De momento, seleccionamos "Don't open any file" y seleccionamos "Open".

Esto nos llevara a la siguiente pantalla, que es la de como cargar el archivo a analizar:

![Load-Options](/img/cutter00/load-options.png)

En esta ventana vemos muchas opciones tales como el programa, si realizaremos un análisis y el nivel del mismo, hasta opciones más complejas como la arquitectura del CPU, el formato, el tipo de kernel y los offsets. Particularmente, observo poderoso que nos permita cargar scripts y PDBs para subirle mas valor al análisis o realizar tareas mas complejas vía scripts.

Seleccionamos Ok para continuar. Nos aparece la siguiente ventana en donde ya tenemos acceso completo a Cutter, vamos a ir analizando parte por parte sobre lo que nos incluye la herramienta:

![Main-Cutter](/img/cutter00/main-cutter.png)

Al abrir Cutter, yo al menos usualmente reinicio el layout, por lo que "view--\>reset layout" como opción el cual me deja sobre la sección overview de la ventana Dashboard. En esta podemos ver al lado izquierdo las funciones del binario, abajo un buscador rápido y en el panel superior unas flechas para movernos entre lo que vamos explorando de un binario.

![Reset-Layout](/img/cutter00/fresh-layout.png)

Es importante señalar que el botón de "Play" lleva un pequeña letra "E". Esto es porque dentro de las capacidades de Cutter esta la de emular el código que estemos revisando, es decir, replicar el comportamiento que tendría si estuviera realmente ejecutándose. Aun lado derecho tenemos un buscador de flags y direcciones, para movernos mas rápido dentro del binario.

![Emulate](/img/cutter00/panel-superior.png)

Abajo de este panel superior, podemos observar como esta clasificado el binario, este en una barra negra de momento.

Ya en el dashboard, la información sobre el binario que estemos analizando, desde si contiene canarios hasta la entropía del mismo.

![Dashboard](/img/cutter00/dashboard.png)

La siguiente ventana justo a un lado del dashboard es Disassembly, la cual contiene las ubicaciones y las instrucciones del binario que estemos analizando, aquí es donde pasaremos mas tiempo analizando el código de los retos del crackme, por lo que más adelante estaremos revisando algunos de los hotkeys para trabajar en esta sección.

![Disassembly](/img/cutter00/disassembly.png)

Continuando con Graph, otra sección donde estaremos trabajando a menudo, esta nos permite visualizar las instrucciones y navegar entre ellas con algunas teclas, permitiéndonos continuar la emulación de una manera mas simple mientras vamos viendo que sucede con las instrucciones.

![Graph](/img/cutter00/graph.png)

Continuamos con la sección de Hexdump que no es nada mas que la salida en una tabla hexadecimal del binario que estemos analizando. Importante señalar que la pestaña parsing, nos permite realizar ajustes para ver mas haya de lo evidente.

![Hexdump](/img/cutter00/hexdump.png)

Aun lado de Hexdump, encontraremos el poderoso strings y import. Ambos muy útiles para conocer las llamadas que realiza el binario, que tipo de texto tiene guardado en su interior y como podemos encontrar rápidamente la utilidad y mensajes que guarda nuestro binario a analizar.

![Strings](/img/cutter00/strings.png)

![Import](/img/cutter00/import.png)

Esto ha sido todo por el momento, en la siguiente entrada analizaremos el primer crackme y daremos vida a cada una de las secciones. Espero que les haya gustado.

Saludos,
