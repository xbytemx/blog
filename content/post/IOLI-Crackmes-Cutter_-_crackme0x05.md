---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x05"
date: 2019-02-09T22:02:44-06:00
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
draft: false
---

Continuamos con el siguiente crackme, el cual presenta una variación del reto anterior que resolvimos, en el cual se realizaban varias comparativas diferentes, así como cast entre tipos de datos.
<!--more-->

## Footprint

Comenzamos como hemos resuelto otros crackmes, por realizar un file, un ldd y un readelf:

![Initial info about crackme0x05](/img/cutter06/file.png)

Podemos observar que la mayoría de la información que se ha presentado en otros crackmes se mantiene, así que por este momento el preCutter se detiene aquí.

Cargamos el binario en Cuttter:

![Load on Cutter](/img/cutter06/load.png)

Y como en otros ejercicios, veamos si el dashboard nos ayuda con alguna otra información adicional:

![Dashboard](/img/cutter06/dashboard.png)

Como podemos observar y si recordamos otros ejercicios, parece que has sido construido de la misma manera que los demás crackmes, así que hasta aquí llegamos en esta sección.

## Entrypoints

Ahora toca el turno de analizar el entry point que vimos con readelf, el cual era 0x080483d0, veamos que dice Cutter al respecto:

![Entry Points](/img/cutter06/entrypoints.png)

La información coincide y es mas, podemos ver también la llamada de entry0 que tenemos para este entry point dentro de las funciones.

## Strings

Veamos si en los strings podemos encontrar algún indicio diferente o la variación entre este crackme y los anteriores:

![Strings](/img/cutter06/strings.png)

Con esta captura, concluimos que los strings no son de mucha ayuda en este crackme.

## Imports

Pasemos ahora a las funciones que se importan de este crackme:

![Imports](/img/cutter06/imports.png)

## Functions

En la sección de funciones encontraremos las que ya analizamos están importadas via libc, y adicionalmente observaremos que aparte de main, tenemos una función llamada check.

![Functions](/img/cutter06/functions.png)

Continuemos con el disassembly de este binario:

## Disassembly

### Función main

Al ser el main el glue de todo el programa, iniciamos por esta función, la cual tiene el siguiente código:

![pdf Main](/img/cutter06/main.png)

Siguiendo la misma mecánica que con los crackme anteriores, vamos parte por parte destripando el programa.

![Main - 1](/img/cutter06/main1.png)

_0x08048540 ... 0x08048549_

Tenemos el inicio del prologo de la función, donde se comienza a definir el stack frame para la función.

_0x0804854c ... 0x08048554_

EAX es igual a 0, luego le sumamos 15, y luego le volvemos a sumar 15. EAX=30

_0x08048557 ... 0x0804855d_

EAX / 2^4 y después EAX * 2^4. Resultado: EAX =16. Restamos 16 al ESP

![Main - 2](/img/cutter06/main2.png)

_0x0804855f ... 0x08048572_

Subimos la ubicación del string de LVL al ESP, y llamamos a la función printf. Este proceso se repite otra vez con el string de "Password: ".

![Main - 3](/img/cutter06/main3.png)

_0x08048577_ Subimos la dirección de local\_78h a EAX

_0x0804857a_ Movemos esa dirección almacenada en EAX a local\_4h (ESP+4h)

_0x0804857e_ Movemos la ubicación del string '%s', la cual es 0x080486b2, a ESP

_0x08048585_ Llamamos a scanf, pero ojo, recibe el argumento1 via ESP y el argumento2 via ESP+4h como en los ejercicios anteriores.

![Main - 4](/img/cutter06/main4.png)

_0x0804858a_ Bueno, parece que ahora esa dirección ya tiene un valor via scanf, ahora volvemos a llevar la dirección a EAX
_0x0804858d_ Cargamos EAX en el ESP
_0x08048590_ Y llamamos a otra función pasándole como parámetro la dirección del valor que tomamos de scanf.

![Main - 5](/img/cutter06/main5.png)

Cuando ya retornemos de la función check, únicamente retornamos 0 vía EAX y acaba la función con leave+ret.

### Función check

Ahora si, la función buena a resolver:

![pdf Check](/img/cutter06/check.png)

No se ve tan monstruosa como la anterior, pero tiene su truco.

![Check - 1](/img/cutter06/check1.png)

La primera parte es equivalente al prologo de la función, donde se define un stack de 40 bytes para la función. 

![Check - 2](/img/cutter06/check2.png)

Inicializamos local\_8h y local\_ch a 0. Si vemos la descripción que nos entrega cutter, veremos que estas son variables locales.

![Check - 3](/img/cutter06/check3.png)

_0x080484dc_ Asignamos el valor de s a EAX, ¿y quien es s? Pues no es mas que el parámetro que la pasamos a la función, la dirección donde se encuentra el valor que ingresamos via scanf

_0x080484df_ Ahora subimos esa dirección a ESP (whoot) 

_0x0804842e_ Llamamos a strlen (que recibe la dirección del string). strlen básicamente toma un string y ejecuta un loop while incrementando un contador buscando la primera incidencia de un carácter null \x00. Al encontrarlo devuelve el valor del contador o por cuantos caracteres tuvo que evaluar para encontrar el carácter null, es decir, la longitud del string. El valor devuelto es un numero natural n>0 que se almacena en EAX.

_0x080484e7_ Tomamos el valor de EAX y lo comparamos con el valor de local\_ch (la variable que habíamos inicializado a 0), el resultado levanta las flags de ZF y CF según corresponda. En la primera ejecución local\_ch es menor que EAX, por lo que la flag de CF se levanta.

_0x080484ea_ Tenemos un salto condicional, si el flag de CF no se levanto, entonces ejecuta el jump. Hasta que local\_ch no sea igual o mayor a la longitud del string ingresado, no se ejecutara el jump.

![Check - 4](/img/cutter06/check4.png)

Si el jump no se ejecuta;

_0x080484ec_ Movemos el valor del contador a EAX

_0x080484ef_ Sumamos el counter y la dirección de s (el valor ingresado via scanf)

_0x080484f2_ Movemos el byte de AL a EAX (ojo, movimos el valor) y dejamos EAX lleno de ceros a la izq. 

_0x080484f5_ Subimos ese valor (el byte) en local\_dh. 

Ahora local\_dh vale el valor de la letra en curso según el ciclo for.

![Check - 5](/img/cutter06/check5.png)

_0x080484f8_ Subimos la dirección de local\_4h a EAX

_0x080484fb_ Cargamos EAX en local\_8h\_2 (el que esta mas cerca del ESP)

_0x080484ff_ Movemos el valor que esta en 0x08048668 (%d)

_0x08048507_ Cargamos en EAX la dirección de local\_dh (la letra en curso del texto ingresado)

_0x0804850a_ Subimos esta letra a ESP desde EAX

_0x0804850d_ Llamamos a la función sscanf, que recibe como parámetros el char \*s[i], el string '%d', y el entero local\_4h

La función sscanf() se encarga de procesar un string que recibe de parámetro inicial y al igual que su pariente, scanf, recibe también el formato y opcionalmente donde almacenar el resultado. Esto nos da una clara idea de que lo que ingresemos de input, sera convertido a int y almacenado en local\_4h.

![Check - 6](/img/cutter06/check6.png)

_0x08048512_ Movemos el valor de local\_4h a EDX

_0x08048515_ Subimos la dirección de local\_8h a EAX

_0x08048518_ Sumamos el valor de EDX (el entero de la letra en curso) y EAX (que era 0 al inicio del for, fue la primera en inicializarse), y lo guardamos en EAX

_0x0804851a_ Comparamos el valor de EAX contra 0x10 (16 en decimal)

Esto básicamente quiere decir que hasta este punto, local\_8h es una variable fuera del ciclo donde se va sumando los valores individuales de cada carácter ingresado en el primer scanf. También significa que debemos ingresar números, puesto que la comparación se realizo contra 16, cantidad que ninguna letra puede hacer frente.

![Check - 7](/img/cutter06/check7.png)

Entramos en un if, que dice así;

Si el resultado de la comparación fue falso (no iguales), entonces brinca a 0x0804852b.

_0x0804852b_ Cargamos la dirección de contador en EAX

_0x0804852e_ Incrementamos el valor de EAX

_0x08048530_ Brincamos hacia arriba hasta 0x080484dc, donde el loop comenzó

Si el resultado de la comparación fue verdadero (iguales), entonces continua.

_0x0804852b_ Movemos la dirección de s a EAX

_0x0804852b_ Cargamos esa dirección en ESP

_0x0804852b_ Llamamos a la función parell que recibe de parámetro s.

Las ultimas tres instrucciones son básicamente cuando el loop finalizo, en donde se carga el mensaje de "Password Incorrect" para posteriormente imprimirlo en pantalla via printf. Tenemos un leave/ret que finaliza la función y nos regresa a main. 

### Función parell

Hemos llegado hasta esta función después de recibir en main un string, enviarlo de parámetro a check, en donde fue validado  caracter por caracter, tras una conversón a entero, que la sumatoria total de todos los valores es de 16, por lo que check llama a parell enviando el string que recibió de main.

![Parell](/img/cutter06/parell.png)

Como podemos ver, esta función es mas pequeña que check y main, pero veremos que secretos nos aguarda.

![Parell - 1](/img/cutter06/parell1.png)

Esta primera parte del prologo de la función define un stack frame de 0x18 para la función.

![Parell - 2](/img/cutter06/parell2.png)

_0x0804848a_ Cargamos la dirección de local\_4h en EAX

_0x0804848d_ Cargamos el valor de EAX en local\_8h (Cerca de ESP)

_0x08048491_ Continuamos subiendo el valor '%d' a format aka ESP+0x4

_0x08048499_ Movemos la dirección del string s hacia EAX

_0x0804849c_ Cargamos la dirección del string s en ESP

_0x0804849f_ Llamamos a sscanf.

Ok, sscanf (\*s, '%d', local\_4h), como en la función anterior.

![Parell - 3](/img/cutter06/parell3.png)

_0x0804849f_ Movemos el valor de local\_4h a EAX (osea el numero ingresado)

_0x0804849f_ Realizamos un and entre el numero y 1. Esto quiere decir que básicamente evalúa que la salida sea o no par, aplicando una operación el bit menos significativo.

_0x0804849f_ Se realiza un test entre EAX y EAX para validar que el **and** haya resultado verdadero

![Parell - 4](/img/cutter06/parell4.png)

En caso de que el numero sea par, se imprime el mensaje "Pasword OK!" y se finaliza el programa con estatus 0.

En caso de que el numero no sea par, parell retorna a check y check a su vez finalizar el for, concluyendo con el mensaje de "Password Incorrect."


## Gráfico

Los grafos de las tres funciones principales son:

### Función main

![graph main](/img/cutter06/graph_main.png)

### Función check

![graph check](/img/cutter06/graph_check.png)

### Función parell

![graph parell](/img/cutter06/graph_parell.png)

## Validación

Como la información que analizamos de este binario, concluimos que no hay una única respuesta, sino varias. La condición es que los números en serie que ingresemos puedan sumar 16 exacto y que el numero sea par, es decir, números como 04444 y 22222222 son validos. Números como 55421 no porque a pesar de que sus primeros números en serie suman 16, el numero es impar, mientras que números como 5560 o 1234512 si porque los primeros números suman 16 y son números pares.

Validemos:

![Validación](/img/cutter06/validacion.png)

Excelente! Hemos resuelto el sexto crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,


