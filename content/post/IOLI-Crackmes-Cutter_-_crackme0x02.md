---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x02"
date: 2018-10-27T22:54:21-06:00
draft: false
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
description: "Nuevamente a usar Cutter para resolver crackmes! En esta ocasión continuamos con la serie de crackmes de IOLI, ahora resolviendo el 3er crackme"

---

## Footprint

Comenzamos por obtener información sobre el crackme que tenemos entre manos, para ello usamos las herramientas usadas en el crackme anterior:

File:

![File over crackme0x02](/img/cutter03/file.png)

LDD:

![LDD over crackme0x02](/img/cutter03/lld.png)

ReadELF:

![readELF over crackme0x02](/img/cutter03/readelf.png)

Parece que este caso no es tan diferente a los anteriores.

Carguemos el archivo directo a Cutter con los valores de análisis por defecto:

![Dashboard de crackme0x02](/img/cutter03/dashboard.png)

También a nivel de dashboard vemos que este reto no es muy diferente a sus predecesores. Continuemos con la siguiente sección.


## Entrypoints

Comenzamos por analizar todos los entry points de este ejecutable:

![Entry Points de crackme0x02](/img/cutter03/entrypoints.png)

Como podemos ver, el entry point es el mismo que ya nos decía readelf, **0x8048330**.


## Strings

Los strings dentro de este ejecutable son (ojo con la dirección):

![Strings de crackme0x02](/img/cutter03/strings.png)

Aparentemente no sera tan sencillo esta vez, no aparece ni por error algún valor que haga referencia al password.


## Imports

En la sección de imports, veremos que nuevamente llamamos a nuestras funciones de costumbre:

![Imports de crackme0x02](/img/cutter03/imports.png)

> Nuevamente tenemos un mensaje sobre lo inseguro que es la función scanf, ya que es altamente explotable.


## Functions

Ahora toca revisar las funciones que tenemos en nuestro programa, las cuales son las siguientes:

![Funciones de crackme0x02](/img/cutter03/functions.png)

No quiero sorprender a nadie, pero esta casi idéntico a lo que teníamos en el anterior crackme... continuemos.


## Disassembly

Ahora toca el turno del Disassembly, en donde realizaremos el análisis estático para determinar que sucede dentro de este programa, vemos primero como se presenta:

![Disassembly de crackme0x02](/img/cutter03/dis1.png)

> Trate de reducir un poco el tamaño de la letra para que fuera visible toda la función main.

Aplicando la misma metodología que hemos manejado:

```
0x080483e4      push ebp
0x080483e5      mov  ebp, esp
0x080483e7      sub  esp, 0x18
0x080483ea      and  esp, 0xfffffff0
0x080483ed      mov  eax, 0
0x080483f2      add  eax, 0xf
0x080483f5      add  eax, 0xf
0x080483f8      shr  eax, 4
0x080483fb      shl  eax, 4
0x080483fe      sub  esp, eax
```

Primero tenemos el stack frame, que como hemos visto antes nos ayuda a separar lo que las variables locales usaran posteriormente.

```
0x08048400      mov  dword [esp], str.IOLI_Crackme_Level_0x02 ; [0x8048548:4]=0x494c4f49 ; "IOLI Crackme Level 0x02\n" ; const char *format
0x08048407      call sym.imp.printf ; int printf(const char *format)
```

Continuamos con la primera llamada a printf, el cual recibe de parámetro el string "IOLI Crackme Level 0x02\\n", dando inicio al primer mensaje que vemos en pantalla.

```
0x0804840c      mov  dword [esp], str.Password: ; [0x8048561:4]=0x73736150 ; "Password: " ; const char *format
0x08048413      call sym.imp.printf ; int printf(const char *format)
```

Continuamos con la segunda llamada a printf, la cual recibe de parámetro el string "Password: ", manteniendo el cursor sin salto de linea.

```
0x08048418      lea  eax, [local_4h]
0x0804841b      mov  dword [local_4h_2], eax
0x0804841f      mov  dword [esp], 0x804856c ; [0x804856c:4]=0x50006425 ; const char *format
0x08048426      call sym.imp.scanf ; int scanf(const char *format)
```

La siguiente parte toma la variable local_4h y carga su posición en memoria sobre EAX, para después guardar dicho valor en local_4h_2. La siguiente parte carga en el stack pointer el valor de lo ubicado en 0x804856c. Finalmente scanf es llamado en donde usara el valor de la ubicación 0x804856c y la ubicación en memoria de local_4h.

Comenzando por las traducciones, veamos que hay en 0x804856c:
> Recordemos que para ir a una posición debemos dar dos clicks izq sobre la misma.

![rodata](/img/cutter03/rodata-strings.png)

Hemos llegado a esta posición de .rodata, que como en el ejemplo anterior no se tradujo 100% a string, por lo que nos toca ayudar un poco.

Nuevamente damos click derecho, sobre la ubicación, seleccionamos "Set to Data", ubicamos "...   *" y continuamos con la siguiente ventana donde seleccionamos tamaño 3, un solo elemento.

![Convert code to data](/img/cutter03/code-to-data.png)

![Set to Data](/img/cutter03/set-to-data.png)

Y ahora se ha revelado el primer parámetro de scanf:

![1er parámetro de scanf](/img/cutter03/scanf-1.png)

Si observamos detenidamente los parámetros que nos indica el análisis de la función, veremos algo como lo siguiente:

    ; var int local_4h @ ebp-0x4
    ; var int local_4h_2 @ esp+0x4

Esto quiere decir que local_4h y local_4h_2 son reconocidos como enteros y no solo eso, por la ubicación en memoria (o el valor real y no el alias) tenemos que local_4h es **EBP-0x4** mientras que local_4h_2 es **ESP+0x4**, esto quiere decir que las primeras dos instrucciones:

    0x08048418      lea  eax, [local_4h]
    0x0804841b      mov  dword [local_4h_2], eax

Primero guarde la dirección de EBP-0x4 en EAX y después escribí dicha dirección sobre ESP+0x4, para que acabando el primer argumento (ESP), pueda tomar el segundo del stack.

Renombremos las variables como numero (local_4h) y *numero (local_4h_2), ubicándonos en el nombre y dando click derecho seleccionaremos "rename ... (used here)":

> Hotkey: shift + n

![2do parámetro de scanf](/img/cutter03/rename-numero.png)

Agreguemos también una etiqueta sobre el valor del primer parámetro, nos ubicamos sobre la ubicación del "%d" y ejecutamos renombrar:

![Etiqueta de decimal](/img/cutter03/label.png)

Ahora que hemos realizado lo mismo del ejercicio anterior, donde determinamos el tipo de variable que estaremos usando, continuemos con el siguiente bloque de instrucciones:


```
0x0804842b      mov  dword [local_8h], 0x5a ; 'Z' ; 90
0x08048432      mov  dword [local_ch], 0x1ec ; 492
0x08048439      mov  edx, dword [local_ch]
0x0804843c      lea  eax, [local_8h]
0x0804843f      add  dword [eax], edx
0x08048441      mov  eax, dword [local_8h]
0x08048444      imul eax, dword [local_8h]
0x08048448      mov  dword [local_ch], eax
0x0804844b      mov  eax, dword [local_4h]
```
En las primeras dos instrucciones, asignamos valor a las variables local_8h y local_ch, con 90 y 492 respectivamente. Renombremos estas variables a i y j.

![int i y j](/img/cutter03/i-j.png)

    i = 90
    j = 492

Después salvamos el valor 492 en EDX, y guardamos la ubicación de memoria de "i" (ebp-0x8) en EAX.

Continuamos por sumar 0x1ec (492) a EAX (i=90)y guardarlo en EAX. EAX ahora vale 0x246.

    i += j

La siguiente parte se encarga de asignar 582 a EAX. Después multiplicamos 582 * 582, osea, 582^2, dándonos 0x52b24 o 338724. Este resultado se guarda en "i"

    i *= i

Guardamos "i" en "j":

    j = i

Finalmente cargamos numero en "i"

    i = numero

Continuemos con el siguiente bloque:


```
0x0804844e      cmp  eax, dword [local_ch]
0x08048451      jne  0x8048461
0x08048453      mov  dword [esp], str.Password_OK_: ; [0x804856f:4]=0x73736150 ; "Password OK :)\n" ; const char *format
0x0804845a      call sym.imp.printf ; int printf(const char *format)
0x0804845f      jmp  0x804846d
0x08048461      mov  dword [esp], str.Invalid_Password ; [0x804857f:4]=0x61766e49 ; "Invalid Password!\n" ; const char *format
0x08048468      call sym.imp.printf ; int printf(const char *format)
0x0804846d      mov  eax, 0
0x08048472      leave
0x08048473      ret
```

Iniciamos por comparar i==j, si es verdadero tenemos un printf("Password OK :)\\n"), si es falso llamaremos a printf("Invalid Password!\\n")

```
    if (i==j):
      printf("Password OK :)\n")
    else:
      printf("Invalid Password!\n")
```

Finalmente tenemos EAX=0 y el leave (ESP=EBP, pop EBP) de la función main.

Creo que hasta este punto el análisis estático ha bastado para poder identificar la contraseña de este crackme.


## Grafico

No puede faltar el grafo de esta función, en donde veamos a nuestro único if haciendo una pirueta:

![Graph](/img/cutter03/graph.png)


## Validación

Hemos llegado a la parte final, donde validamos la conclusión a la que hemos realizado después del análisis estático, en la cual la contraseña que debemos ingresar en el crackme es **338724**.

Validemos:

![Validación](/img/cutter03/validacion.png)

Excelente! Hemos resuelto el tercer crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,
