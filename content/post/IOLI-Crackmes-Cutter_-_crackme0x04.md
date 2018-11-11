---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x04"
date: 2018-10-29T11:12:38-06:00
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
draft: true
---

El día de hoy toca resolver el 5to crackme de IOLI, el cual consta de dos funciones para realizar las comparaciones y en cada una realiza un tipo de conversión y comparación muy interesante, comencemos
<!--more-->

## Footprint

Comenzamos como hemos resuelto otros crackmes, por realizar un file, un ldd y un readelf:

![Initial info about crackme0x04](/img/cutter05/initial.png)

Podemos observar que la mayoría de la información que se ha presentado en otros crackmes se mantiene, así que por este momento el preCutter se detiene aquí.

Cargamos el binario en Cuttter:

![Load on Cutter](/img/cutter05/load.png)

Y como en otros ejercicios, veamos si el dashboard nos ayuda con alguna otra información adicional:

![Dashboard](/img/cutter05/dashboard.png)

Como podemos observar y si recordamos otros ejercicios, parece que has sido construido de la misma manera que los demás crackmes, así que hasta aquí llegamos en esta sección.

## Entrypoints

Ahora toca el turno de analizar el entry point que vimos con readelf, el cual era 0x080483d0, veamos que dice Cutter al respecto:

![Entry Points](/img/cutter05/entrypoints.png)

La información coincide y es mas, podemos ver también la llamada de entry0 que tenemos para este entry point dentro de las funciones.

## Strings

Veamos si en los strings podemos encontrar algún indicio diferente o la variación entre este crackme y los anteriores:

![Strings](/img/cutter05/strings.png)

Con esta captura, concluimos que los strings no son de mucha ayuda en este crackme.

## Imports

Pasemos ahora a las funciones que se importan de este crackme:

![Imports](/img/cutter05/imports.png)

Como podemos observar, tenemos una nueva función, la cual se llama sscanf, también marcada como insegura (explotable) junto con strlen y scanf.

### SSCANF

Como podemos también observar [aquí](https://www.tutorialspoint.com/c_standard_library/c_function_sscanf.htm) sscanf es una función que lee la información de un string y la procesa en un formato especifico que puede ser de varios tipos. También retorna un valor, el cual es el numero de de variables que pudo asignar con los valores del formato y el string de entrada.

El ejemplo propuesto por el enlace es el siguiente:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main () {
   int day, year;
   char weekday[20], month[20], dtm[100];

   strcpy( dtm, "Saturday March 25 1989" );
   sscanf( dtm, "%s %s %d  %d", weekday, month, &day, &year );

   printf("%s %d, %d = %s\n", month, day, year, weekday );

   return(0);
}
```

En este ejemplo dtm se inicializa con el string origen, después sscanf lo procesa tomando como parámetros el string origen, el formato y las variables donde depositara lo que se procese entre formato y string origen. Un punto importante, es que los argumentos son las ubicaciones en memoria donde esta el espacio del valor.

## Functions

En la sección de funciones encontraremos las que ya analizamos están importadas via libc, y adicionalmente observaremos que aparte de main, tenemos una función llamada check.

![Functions](/img/cutter05/functions.png)

Continuemos con el disassembly de este binario:

## Disassembly

### Función main

Al ser el main el glue de todo el programa, iniciamos por esta función, la cual tiene el siguiente código:

![Main](/img/cutter05/main.png)

Para agilizar las explicaciones y retenerlas, estaré agregando comentarios a todos las imágenes conforme vamos analizando el código.

> HIT: ";" para agregar comentarios

Empezamos por la primera parte, salvar el punto de retorno y realizar el stack frame:

![dis1](/img/cutter05/dis1.png)

Esta parte la hemos platicado con anterioridad, básicamente definimos el espacio donde nuestras dos variables estarán trabajando (local_78h y local_4h) las cuales radare2 nos indica se tratan de variables tipo enteras.

![dis2](/img/cutter05/dis2.png)

La siguiente parte también ya hemos visto que se repite en estos crackmes, se trata de subir al ESP la dirección del string que se encuentra en .rodata y después llamar a la función printf que se encargara de mostrar el string en pantalla durante la ejecución.

![dis3](/img/cutter05/dis3.png)

La siguiente parte se trata del scanf que también hemos resulto ya. Este recibe dos argumentos, el primero es el formato en que manejara la información entregada desde el input y la segunda la dirección en memoria donde almacenara la información. El primer argumento se trata nada mas y nada menos que de "%s" que es usado para manera strings. La segunda parte por otro lado se trata de la dirección de local_78h, la cual se guarda en ESP-0x4 justo después de "%s".

![dis4](/img/cutter05/dis4.png)

La siguiente parte carga la dirección donde se almaceno el string que ya guardamos en EAX, para posteriormente subirlo al ESP, de esta manera la función check podrá usar el string para realizar alguna operación.

![dis5](/img/cutter05/dis5.png)

Finalmente parece que check es tipo void, porque no nos interesa si nos devuelve algo en EAX, por lo que la siguiente parte de instrucción hace alusión a un _return 0_

Si tuviera que escribir el código en C para estas instrucciones, quedaría de la siguiente manera:

```c
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  char input[120];

  printf("IOLI Crackme Level 0x04\n");
  printf("Password: ");
  scanf("%s", input);
  check(input);

  return 0;
}
```

La función main completa con comentarios es la siguiente:

![dis10](/img/cutter05/dis10.png)

Analicemos ahora la función check que recibió nuestro string.

### Función check

El vistaso general a la función queda de la siguiente manera:

![Check](/img/cutter05/check.png)

Como podemos empezar a ver, por ahí debe haber algún tipo de loop o if que este causando los _jmp_. Pero primero lo primero:

![dis11](/img/cutter05/dis11.png)

Al comenzar a analizar las instrucciones lo primero que nos percatamos es que tenemos 6 variables y 1 argumento. Si, el mismo string que ya tomamos vía scanf es uno de los parámetros.

Stack Frame y salvar EBP de esta función, eso es lo que tenemos en la primera parte. Continuemos:

![dis12](/img/cutter05/dis12.png)

Interesante, tenemos una asignación a 0 de variables, local_8h y local_ch que se encuentran particularmente juntas. Veamos para que son usadas antes de renombrarlas.

![dis13](/img/cutter05/dis13.png)

El siguiente bloque de instrucciones vemos que es un retorno de algún jump, y realiza unas instrucciones que ya vimos en el ejercicio anterior; obtener la longitud del parámetro input (tipo string). Recordemos que esta función retorna su valor que es un entero sin signo, en EAX.

![dis14](/img/cutter05/dis14.png)

La siguiente instrucción después de conseguir la longitud del texto es compararlo con local_ch... esto vimos que se aplico en el ejercicio anterior, por lo que visitemos rápidamente el final del loop para identificar si tenemos un incremento antes de retornar... donde efectivamente encontramos _lea eax,local\_ch_ y _inc eax_, lo que definitivamente nos indica que es un loop, inclusive me atrevo a decir que es parecido al loop del crackme anterior. Eso significa que local_ch debe ser el contador de este for. Renombremos la primera variables

> HIT: "shift + n" para renombrar variables

Continuemos en el paso por paso:

![dis15](/img/cutter05/dis15.png)

Esta parte es parecida al crackme anterior, lo que sucede es que guardamos el valor del counter, ejemplo 0, sobre EAX. Continuamos por sumar 0 + la ubicación del string, es decir, EBP+0x8, guardando el valor final en EAX. De esta manera conforme el counter avance, estaría haciendo EBP+0x9, EBP+0x10, etc. hasta acabar con el string (antes de llegar a null).

Las siguientes dos instrucciones _movzv eax, dword\[s\]_  y _mov byte\[local_dh\], al_, se encargan de extraer el byte _n_ del string, osea el carácter tipo s\[i\] y almacenarlo en el char local_dh (EBP-0xd).

![dis16](/img/cutter05/dis16.png)

Nuestro siguiente bloque de instrucciones, es básicamente la preparación y llamada a sscanf. Iniciamos por guardar la dirección de EBP-0x4 en EAX, para después guardarla en ESP+0x8 (3er parámetro). Continuamos por mover el string "%d" a ESP+0x4 (2do parámetro) y finalmente subimos s\[i\] en el ESP (1er parámetro).

Nuestra llamada a la función sscanf queda de la siguiente manera: sscanf(s\[0\], "%d", \&valor_entero)

Ahora que sabemos un poco más para que se usan las variables local_dh, local_4h y local_8h_2, renombremos las para no olvidar que hacen:

![dis17](/img/cutter05/dis17.png)

Continuemos con el siguiente bloque de instrucciones:

![dis18](/img/cutter05/dis18.png)

Iniciamos por salvar en EDX el valor entero del carácter _n_ del string que estamos checando. Luego guardamos la dirección en memoria de EBP-0x8 (recordemos que al inicio le dimos el valor de 0) en EAX. Luego sumamos el valor de EBP-0x8 con el valor entero, salvando el resultado en EBP-0x8. Esto significa que si EBP-0x8 valia 0, ahora valdrá el valor_entero.

Si repito el loop, el EBP-0x8 sera igual a la suma total de cuantos caracteres tenga el string. Renombremos EBP-0x8 como total.

Finalmente, comparo el valor de EBP-0x8 con 0xf (15)... veamos caso falso que no es igual (not jump)

![dis19](/img/cutter05/dis19.png)

Si el valor total es igual a 15, entonces imprimo "Password OK!\\n" y termino de manera exitosa el programa con un exit 0.

![dis21](/img/cutter05/dis21.png)

Si el valor total no es 15 (caso donde si brinco), incremento el contador y regreso al inicio del loop para volver a ejecutar todo.

![dis22](/img/cutter05/dis22.png)

Finalmente si ni una vez logre sumar 15 y ya se me acabaron las letras en el string, brinco fuera del loop y retorno a main.

Si tuviera que reescribir lo que entendí de estas instrucciones en lenguaje C, seria como el siguiente:

```c
void check (char *s)
{
  int total=0, counter, valor;

  for(counter=0;counter<strlen(s);counter++)
  {
    sscanf(s[counter], "%d", &valor);
    total += valor;
    if (valor == 15)
    {
      printf("Password OK!\n");
      exit(0);
    }
  }

  printf("Password Incorrect!\n");
}
```

Toda la función check queda de la siguiente manera:

![dis20](/img/cutter05/dis20.png)

## Grafico
### Función main

![Graph de Main](/img/cutter05/graph-main.png)

### Función check

![Graph de Check](/img/cutter05/graph-check.png)

## Validación

Como la información que analizamos de este binario, concluimos que no hay una única respuesta, sino varias. La condición es que los números en serie que ingresemos puedan sumar 15 exacto, es decir, números como 12345 y 555 son validos. Números como 5542 no porque suman 16, pero números como 5555 o 123456789 si porque los primeros números suman 15.

Validemos:

![Validación](/img/cutter05/validacion.png)

Excelente! Hemos resuelto el quinto crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,
