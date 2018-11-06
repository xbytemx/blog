---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x00"
date: 2018-10-20T00:07:58-05:00
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
draft: false
---

En la entrada anterior, descargamos y revisamos muy por encima lo que Cutter nos presentaba con sus paneles y ventanas. Hoy daremos inicio al análisis del primer crackme de IOLI
<!--more-->

En la entrada anterior, descargamos y revisamos muy por encima lo que Cutter nos presentaba con sus paneles y ventanas. Hoy daremos inicio al análisis del primer crackme de IOLI. Para la gente acostumbrada al Reversing, esto sera cosa de todos los días. Pero si eres nuevo, te invito a analizar a detalle lo que a continuación se presenta.

> Si estas completamente iniciando, te invito a leer el material (tutoriales, writeups, videos) que se ha escrito en la comunidad de CrackLatinos (CLS), es impresionante. En estos tutoriales omito algunas cosas mas asociadas a entender el detalle de las instrucciones y la estructura de los binarios.

Comenzamos por iniciar Cutter y presionar "Select", navegamos hasta la carpeta del crackme0x00 y lo seleccionamos. En mi caso queda de la siguiente manera:

![Open](/img/cutter01/open.png)

Continuamos por dar a open y nos aparecerá la ventana de "Load Options", de momento continuamos con los valores por defecto y damos a "Load"

![Load](/img/cutter01/load.png)

Tras acabar la identificación y análisis de nuestro binario, nos deberá aparecer una ventana como la siguiente:

![initial-load](/img/cutter01/initial-load.png)

## Footprint

Iniciamos realizando una identificación y exploración del binario, por lo que cambiamos al panel de dashboard para ver que tenemos con este binario. La primera columna nos dice la siguiente información:

![info1](/img/cutter01/info1.png)

Tenemos un formato ELF32, escrito en C. Continuamos con la siguiente columna:

![info2](/img/cutter01/info2.png)

Primero tenemos que tiene 3 File Descriptors, la base address es 0x08048000 (con el ajuste a 0), es user-space, sin canarios ni crypto. No es estático, mas abajo veremos que tiene una biblioteca de la cual depende.

![info3](/img/cutter01/info3.png)

Finalmente, en la tercera columna podemos observar que pertenece a una arquitectura i386 para Linux. Y como era de esperarse con intel, es little endian.

![info4](/img/cutter01/info4.png)

La parte inferior nos indica que la entropía es moderadamente alta y que el binario depende de libc, lo cual lo convierte en LDD como se menciono atrás.

## Entrypoints

Ok, antes de analizar la ventana de funciones, analizaremos cuantos puntos de entrada tiene este binario, para ello en la barra de menus seleccionaremos "Windows-\>Entry Points" lo cual nos abrirá una pestaña más en la parte inferior:

![entrypoints1](/img/cutter01/entrypoints1.png)

Al seleccionar "Entry Points" en la parte inferior, observaremos que nos vamos a otro panel en donde solo tenemos una entrada, la cual apunta a la dirección 0x08048360, la cual también es etiquetada como entry0 y es la posición donde actualmente nos encontramos. Un punto importante que aun no he mencionado a detalle es que, en la parte superior se  encuentra la barra de address/section, que nos marca en rojo la posición y sección en la que nos encontramos actualmente, esto es bastante cómodo cuando queremos movernos rápidamente sobre .text o otras secciones.

> **Hit:** Podemos dar click derecho sobre la barra de desplazamiento y nos abrirá un menú para seleccionar algún otra ventana que no tengamos en este momento, inclusive nos da un vistazo rápido a todas las ventanas posibles.

## Strings

Continuamos aun sin ir a la ventana de funciones, esta vez pasaremos rápido por strings. Para ello damos a la pestaña de strings que se encuentra en la parte inferior:

![strings1](/img/cutter01/strings1.png)

Como dato a analizar, las direcciones de los primeros strings son diferentes a los de los últimos. También podemos observar que aparecen algunos mensajes probablemente asociados a la ejecución.

![strings2](/img/cutter01/strings2.png)

## Funciones

Ahora si, la sección de funciones, en donde nos moveremos, renombraremos y predecimos que sucede durante la ejecución con ayuda del análisis estático.

![functions1](/img/cutter01/functions1.png)

Podemos observar rápidamente, que ya radare2 identifico correctamente la función main, que tal como recordaremos es la función que se encuentra en todo programa escrito en **C**, y que ademas tiene la función de unir como pegamento todas las partes e iniciar el programa.

Como también podemos observar por la manera en la que se presentan los nombres de las funciones, algunas empiezan por "sym.imp", indicándonos que han sido importadas. Esto se puede observar mas claramente si seleccionamos en el panel derecho la pestaña de "Imports" como aparece a continuación:

![functions and imports](/img/cutter01/functions2.png)

También podemos ver que se marca en verde la función main, esto es debido a que se encuentra explícitamente creada dentro del código como veremos en otros crackmes. En negritas siempre estará marcado aquella posición en donde nos encontremos. Para ir a una función debemos dar doble click izquierdo, por lo que clickeamos a main:

![goto-main](/img/cutter01/functions3.png)

Arriba de main, se encuentra una función llamada fcn.08048384, la cual recibe ese nombre tras no poder ser identificada como una función dinámica y se deja un nombre genérico que haga referencia a su ubicación dentro de la sección .text. Las funciones pueden ser renombradas dando click derecho y seleccionando "rename", tal como veremos un poco mas adelante. Otras opciones importantes son "xrefs" y "Add comment" las cuales veremos mas a detalle un poco mas adelante.

![functions-options](/img/cutter01/functions-options.png)

Continuemos con la siguiente parte, el Disassembly.

## Desensamblador

Ok, la pestaña donde mas pasaremos tiempo analizando las instrucciones del binario, la del desensamblado del binario. Como podemos ver gracias al panel de funciones, ya nos encontramos en la función main y podemos analizar todo el código desensamblado de esta función:

![dis-main](/img/cutter01/dis-main.png)

Lo primero que observamos es lo que sucede en la primera posición:

```assembly
(fcn) main 127
  main (int argc, char **argv, char **envp);
          ; var char *s1 @ ebp-0x18
          ; var char *s2 @ esp+0x4
          0x08048414      push ebp
```

La función tiene dos variables, la primera bien definida detrás del _Base Pointer_ por 0x18 y la otra sujeta al _Stack Pointer_ por un más 0x4.

```asm
0x08048414      push ebp
0x08048415      mov ebp, esp
0x08048417      sub esp, 0x28
0x0804841a      and esp, 0xfffffff0
```

Las siguientes instrucciones corresponden al _Stack Frame_ de main, en donde se reserva 0x28 entre el EBP y el ESP.

```asm
0x0804841d      mov eax, 0
0x08048422      add eax, 0xf
0x08048425      add eax, 0xf
0x08048428      shr eax, 4
0x0804842b      shl eax, 4
0x0804842e      sub esp, eax
0x08048430      mov dword [esp], str.IOLI_Crackme_Level_0x00
0x08048437      call sym.imp.printf
```

Las siguientes instrucciones descienden a ESP para poder cargar en el stack, la dirección de la sección rodata en donde se encuentra uno de los strings que observábamos hace varias imágenes. Posteriormente es llamada a la función printf de libc, la cual despliega el parámetro que recibe (via ESP). Este nos da un mensaje de bienvenida al nivel 00.

Otro tema interesante, es que EAX se queda inicializado con el valor de 0x10.

```asm
0x0804843c      mov dword [esp], str.Password:
0x08048443      call sym.imp.printf
```

El siguiente mensaje es como el anterior, en donde se mueve la ubicación del string "Password" al ESP y se llama a printf para que imprima dicho argumento.

```asm
0x08048448      lea eax, [s1]
0x0804844b      mov dword [s2], eax
0x0804844f      mov dword [esp], 0x804858c
0x08048456      call sym.imp.scanf
```

Continuamos con la primera variable que teníamos definida, s1. Aquí lo que hacemos es cargar la ubicación de s1 en EAX para posteriormente cargar el apuntador a s2 y subir al ESP el valor de 0x804858c, correspondiente al espacio de s1 donde se almacenara la información de scanf, función que es llamada en la ultima linea. Muy importante mencionar que en la ubicación tenemos la instrucción "and eax, 0x35320073", lo cual mantiene el almacenamiento de la variable hasta su ejecución y reserva de manera peligrosa mucho espacio para nuestro string.

```asm
0x0804845b      lea eax, [s1]
0x0804845e      mov dword [s2], str.250382
0x08048466      mov dword [esp], eax
0x08048469      call sym.imp.strcmp
```

Ahora lo que vemos a continuación, es una comparación entre s1 y s2, teniendo s1 el valor que hayamos definido en la variable y siendo s2 el valor string de 250382 (rodata).

```asm
           0x0804846e      test eax, eax
       ,=< 0x08048470      je 0x8048480
       |   0x08048472      mov dword [esp], str.Invalid_Password
       |   0x08048479      call sym.imp.printf
      ,==< 0x0804847e      jmp 0x804848c
      |`-> 0x08048480      mov dword [esp], str.Password_OK_:
      |    0x08048487      call sym.imp.printf
      `--> 0x0804848c      mov eax, 0
           0x08048491      leave
```

En el ultimo bloque que analizaremos, tenemos que en el acomulador EAX se registra el resultado de la comparación, la cual si es 0, levanta el flag de ZF, caso contrario el flag ZF se mantiene en 0. Al llegar a "JE 0x8048480", si ZF es 0, brincamos a 0x08048480, en donde imprimiremos el mensaje "Password OK :)", caso contrario que ZF sea diferente a 0, imprimiremos el mensaje "Invalid Password"

Aunque hasta este punto, ya debe ser evidente la solución a este crackme, no olvidemos revisar la grafica de la función main.

## Grafico

Una de las pestañas que nos ilustra mejor como las funciones interactúan y donde tienen saltos condicionales, es la sección de Graph, la cual luce de la siguiente manera:

![graph](/img/cutter01/graph.png)

Con un poco mas de zoom (con el scroll del mouse) en el bloque principal antes del "if":

![if](/img/cutter01/if.png)

Como podemos observar tenemos un recuadro donde se encuentra las mismas instrucciones que estábamos revisando, solo que con los comentarios adicionales que nos devuelve radare2. Adicionalmente podemos ver una linea verde y otra linea roja que descienden del JE, estas corresponden a los saltos condicionales; verde para verdadero o en nuestro crackme, ZF==0 y rojo para ZF!=0.

En caso verdadero como ya sabemos, imprime el mensaje 'bueno':

![true](/img/cutter01/true.png)

En caso falso como ya sabemos, imprime el mensaje 'malo':

![false](/img/cutter01/false.png)

## Validación

Hemos llegado a la parte final, donde validamos la conclusión a la que hemos realizado después del análisis estático, en la cual la contraseña que debemos ingresar en el crackme es 250382.

Validemos:

![validacion](/img/cutter01/validacion.png)

Excelente! Hemos resuelto el primer crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,
