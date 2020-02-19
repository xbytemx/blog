---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x01"
date: 2018-10-23T18:42:38-05:00
draft: false
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
---

En el crackme anterior, exploramos un poco mas sobre las propiedades de Cutter para analizar el código, pero como tal tomamos los apuntes basados en las instrucciones que observamos directamente del desensamblador. En esta entrada aprenderemos un poco mas sobre como usar el desensamblador y el grafo de brincos condicionales.
<!--more-->

## Footprint
Antes de llegar de lleno con el desensamblado del binario, analicemos si el siguiente crackme se parece al anterior. Comenzamos por un file:

![File](/img/cutter02/file.png)

En la entrada anterior, nos limitamos a usar file y mirar de lleno en Cutter, en esta ocasión, usaremos adicionalmente readelf y ldd para mirar un poco mas sobre la información que de primera mano nos da file.

![ldd](/img/cutter02/ldd.png)

![readelf-headers](/img/cutter02/readelf-headers.png)

Tenemos, la dirección de entrada, la clase, el tipo, maquina y con ldd las bibliotecas importadas.

Ahora si, carguemos el archivo en Cutter, continuando con los valores por defecto y seleccionemos la pestaña de dashboard:

![Dashboard](/img/cutter02/dashboard.png)

## Entrypoints

Podemos observar que es bastante parecido al programa anterior que analizamos, miremos los entrypoints:

![EntryPoints](/img/cutter02/entrypoint.png)

Si asociamos con la salida de readelf, observaremos la equivalencia.

## Strings

Respecto a los strings en este crackme, podemos ver que tiene una pinta parecida al del reto anterior, con la diferencia de que no aparece el string a comparar con el caso anterior, donde se asomaban los números:

![Strings](/img/cutter02/strings.png)

## Imports

Respecto a los imports, observamos que es MUY parecido casi igual a lo que teníamos en el crackme anterior.

![Imports](/img/cutter02/imports.png)

## Functions

Nuevamente entre este reto y el anterior, podemos ver que las funciones son muy parecidas, es decir, tenemos el entry0, main y 3 funciones que se importan de libc, entre ellas scanf y printf.

![Functions](/img/cutter02/functions.png)

## Disassembly

Continuando el flujo del ejercicio anterior, revisemos main con el desensamblador:

![main](/img/cutter02/main.png)

Como podemos observar por la parte de arriba, en donde se define el stack frame, radare2 nos trata de sugerir que en su análisis encuentra dos variables dentro de la función, una llamada local_4h y otra local_4h_2. local_4h, es un entero sin signo, el cual si pongo el mouse sobre el nombre de la variable en este comentario o header de la función, me indicara donde mas tenemos coincidencias:

![On-local_4h](/img/cutter02/on-local_4h.png)

Vemos que del análisis algo llamativo:

```
; var unsigned int local_4h @ ebp-0x4
; var int local_4h_2 @ esp+0x4
```

Cada una variable se encuentra en ubicaciones particulares; una 4 menos el base pointer, y la otra 4 más del stack pointer. En el ejercicio anterior pasó algo igual.

Esto se debe a que una variable esta definida y la otra solo es un valor que se usa dentro de la función.

Ok, después de la definición del stack frame, continuamos con la primera y segunda llamada a printf, en donde de manera igual al ejercicio anterior se carga en el stack la dirección del string que usará printf.

![Welcome](/img/cutter02/welcome.png)

El flujo de las instrucciones continua hasta la siguiente función "scanf", la cual usa la dirección __0x804854c__ para almacenar el tipo de dato entero. Pongamos nuevamente detalle en esta parte:

![scanf](/img/cutter02/scanf.png)

La primera parte, "lea EAX, [local_4h]", carga la dirección de memoria de la variable local_4h en EAX. Después en la siguiente instrucción, "mov [local_4h_2], EAX", esta dirección ubicada en EAX, se escribe en [local_4h_2], es decir, en esp+0x4. Finalmente, antes de llamar a scanf, se ejecuta la instrucción "MOV ESP, 0x804854c". Entonces scanf puede extraer del Stack, primero 0x804854c y después la dirección de local_4h. Si recordamos como usar la función de scanf, recordaremos que scanf recibe dos argumentos:

    scanf("%s", &variable)

El primero se trata de un string en donde se define el tamaño y tipo de dato a almacenar, mientras que el segundo argumento se trata de la posición en memoria de la variable a insertar.

Ahora que hemos comprendido un poco mejor los argumentos que recibe scanf, podemos decir que local_4h se trata de una variable entera, mientras que local_4h_2 se trata del apuntador. Para hacer mas fácil la lectura de las instrucciones, renombraremos ambas variables, primero local_4h como "numero" y después local_4h_2 como \*numero. Para ello damos click derecho sobre la variable y seleccionamos "Rename 'local_4h' (used here)":

![rename_local_4h](/img/cutter02/numero.png)

![rename_local_4h](/img/cutter02/numero2.png)

![rename_local_4h](/img/cutter02/numero3.png)

Continuamos con el pointer:

![rename_local_4h_2](/img/cutter02/ptr-numero.png)

Ya las instrucciones se ven mas legibles. Aunque continuamos con la duda del primer argumento de scanf, "0x804854c".

Visitaremos esta dirección para entender mejor de que se trata. Para ello, tenemos dos opciones, podemos dar click derecho y copiar la dirección "0x804854c", pegarla en la barra de búsqueda y dar enter o dar doble click sobre la dirección y Cutter nos llevara ahí:

![Follow](/img/cutter02/follow.png)

Al llegar a esta sección, la sección de .rodata, veremos que dentro hay varios strings que se han utilizado por printf, tales como "IOLI Crackme Level 0x01\n" ó "Password OK :)\n", pero en medio de ellos, se encuentra una instrucción "and eax, 0x6e490064":

![rodata](/img/cutter02/rodata.png)

Dice el dicho, "Si camina como pato, grazna como pato y nada como pato, entonces es pato", aquí probaremos a convertir la instrucción que aparece a string, para ver si logramos visualizar el argumento tipo string que recibe scanf.

Procedemos a dar click derecho sobre la dirección de la instrucción, ponemos el cursor sobre "set to data" y seleccionamos "...   \*".

![Convert to Data](/img/cutter02/convertodata.png)

Nos aparecerá una nueva ventana preguntando de que tamaño es y cuantos elementos tiene, seleccionamos que la longitud es de 3 y es un solo elemento:

![bytes](/img/cutter02/bytes.png)

Ahora veremos que la presentación ha cambiando revelando el argumento, "%d":

![decimal](/img/cutter02/decimal.png)

Para analizar posteriormente o tener una referencia en cualquier otro punto, agregaremos una flag a esta dirección. Para ello damos click derecho y posteriormente "Add flag", ahí seleccionaremos 3 como el tamaño y le pondremos de nombre "d":

![Add Flag](/img/cutter02/flag1.png)

![Add Flag Name](/img/cutter02/flag2.png)

Seleccionaremos main de la ventana de funciones para regresar, dándole dos clicks. Al regresar, veremos la nueva referencia a ese flag que acabamos de marcar:

![main with flag](/img/cutter02/main-flag.png)

Ahora sabemos que scanf recibe "%d" y &numero como variables, por lo que continuamos con la siguiente parte:

![cmp](/img/cutter02/cmp1.png)

Vemos que de manera similar a lo que teníamos en el reto anterior, se da una instrucción cmp entre la variable numero y el valor 0x149a. Como soy un poco torpe para convertir hexadecimal a decimal, cambiaremos la base de este dato a decimal. Para ello nos posicionamos sobre el valor y damos click derecho, ponemos el mouse sobre "Set Immediate Base to..." y seleccionamos "Decimal":

![Decimal](/img/cutter02/base10.png)

Ahora ya sabemos que numero se compara contra el valor 5274.

La siguiente instrucción evaluá si el valor de numero fue igual a 5274, siendo esto verdadero, brinca y se imprime "Password OK(:", caso falso, continua y imprime "Invalid Password!"

Finalmente, se mueve 0 al acumulador y se retorna la función para ambos casos.

## Grafico

Veamos como lucen las instrucciones en el grafo:

![Graph](/img/cutter02/graph.png)

Muy similar al crackme anterior donde nada mas hay una "evaluación luego brinco". Un mensaje bueno, un mensaje malo y solo una comparación.

## Validación

Hemos llegado a la parte final, donde validamos la conclusión a la que hemos realizado después del análisis estático, en la cual la contraseña que debemos ingresar en el crackme es 5274.

Validemos:

![validacion](/img/cutter02/validacion.png)

Excelente! Hemos resuelto el segundo crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,
