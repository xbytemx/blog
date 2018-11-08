---
title: "Reversing de IOLI Crackmes con Cutter - Crackme0x03"
date: 2018-10-28T20:57:33-06:00
draft: true
tags: ["crackme","ioli","cutter","radare","reversing"]
categories: ["reversing","ELF32"]
---

Hoy toca el 4to crackme de IOLI, el cual nos presenta un reto muy parecido al anterior crackme0x02 solo que usando funciones como veremos más adelante.
<!--more-->

## Footprint

Comenzamos por obtener información sobre el crackme, siguiendo el mismo proceso con la finalidad de identificar algún cambio sobre lo hecho anteriormente en otros crackmes.

![Initial info about crackme0x03](/img/cutter04/initial.png)

Nuevamente tenemos otro ELF32 muy parecido a los anteriores.

Carguemos el archivo directo a Cutter con los valores de análisis por defecto:

![Loading crackme on Cutter](/img/cutter04/load.png)

En el dashboardno vemos mucha diferencia sobre los retos anteriores:

![Dashboard de crackme0x03](/img/cutter04/dashboard.png)


## Entrypoints

Comenzamos por analizar todos los entry points de este ejecutable:

![Entry Point de crackme0x03](/img/cutter04/entrypoint.png)

Como podemos ver, el programa unicamente cuenta con un entrypoint, **0x8048360**.


## Strings

Los strings dentro de este ejecutable son (ojo con la dirección):

![Strings de crackme0x03](/img/cutter04/strings.png)

Sobre salen los strings:

![Strings largos de crackme0x03](/img/cutter04/strings2.png)

Estos mantienen cierta longitud sobre los demás datos que se traducen. También sobre sale por su proximidad con los strings conocidos (.rodata). Veamos como son usados mas adelante.


## Imports

En la sección de imports, veremos que nuevamente llamamos a nuestras funciones de costumbre, mas una adicional strlen:

![Imports de crackme0x03](/img/cutter04/imports.png)

> Se nos advierte que tanto la función scanf, como strlen son funciones inseguras.


## Functions

Ahora toca revisar las funciones que tenemos en nuestro programa, las cuales son las siguientes:

![Funciones de crackme0x03](/img/cutter04/functions.png)

Tenemos ahora una variación con los retos anteriores, tenemos no solo sym.main y las importadas, sino ahora tenemos a sym.shift y sym.test

También dentro de las importadas tenemos ahora a sym.imp.strlen.


## Disassembly

### Función main

Comencemos a realizar el análisis estático para conocer que sucede dentro de este programa. Comenzemos por la funcion principal, main:

![Disassembly de crackme0x03](/img/cutter04/dis1.png)

> Trate de reducir un poco el tamaño de la letra para que fuera visible toda la función main.

Como vemos es bastante similar al reto anterior, solo que ya no tenemos la parte comparativa y en su lugar tenemos una llamada a la función test. Llegaremos a esto después de entender la primera parte.

Aplicando la misma metodología que hemos manejado:

![Stack Frame de crackme0x03](/img/cutter04/dis2.png)

Primero tenemos el stack frame, que como hemos visto antes nos ayuda a separar lo que las variables locales usaran posteriormente.

![Welcome de crackme0x03](/img/cutter04/dis3.png)

Continuamos con la primera llamada a printf, el cual recibe de parámetro el string "IOLI Crackme Level 0x03\\n", dando inicio al primer mensaje que vemos en pantalla.

![Ask for password de crackme0x03](/img/cutter04/dis4.png)

Continuamos con la segunda llamada a printf, la cual recibe de parámetro el string "Password: ", manteniendo el cursor sin salto de linea.

![scanf de crackme0x03](/img/cutter04/dis5.png)

La siguiente parte toma la variable local_4h y carga su posición en memoria sobre EAX, para después guardar dicho valor en local_4h_2. La siguiente parte carga en el stack pointer el valor de lo ubicado en 0x8048634. Finalmente scanf es llamado en donde usara el valor de la ubicación 0x8048634 y la ubicación en memoria de local_4h.

Traducimos esta referencia para ver que valor tiene el const chr *format.
> Recordemos que para ir a una posición debemos dar dos clicks izq sobre la misma.

![Integer](/img/cutter04/integer.png)

![1er parámetro de scanf](/img/cutter04/scanf-1.png)

Ahora sabemos que el valor que reciba se guardara como entero sobre el segundo parámetro.

Como en el crackme anterior, renombraremos los parámetros local_4h y local_4h_2 para que sean mas legibles mas adelante.
> Hotkey: shift + n

![2do parámetro de scanf](/img/cutter04/scanf-2.png)

Ahora que hemos realizado lo mismo del ejercicio anterior, donde determinamos el tipo de variable que estaremos usando y las renombramos, continuemos con el siguiente bloque de instrucciones:

![Operaciones matemáticas](/img/cutter04/dis6.png)

Este bloque lo resolvimos en el ejercicio anterior, el cual en pseudocódigo queda como el siguiente:

    i = 90
    j = 492
    i += j
    i *= i
    j = i

Renombramos las variables para homologar:

![i y j](/img/cutter04/dis7.png)

Terminando las instrucciones anteriores que si conocemos, sigue una serie de asociaciones:

![Load to Stack](/img/cutter04/dis9.png)

Aquí lo que sucede es:

    EAX = j
    ESP+0x4 = EAX
    EAX = num
    ESP = EAX

Básicamente colocamos en el stack dos variables para que cuando llamemos a test, las pueda retirar (parámetros). El stack debe quedarnos parecido a lo siguiente:

| dirección | valor   |
|-----------|---------|
| ESP       | num     |
| ESP+0x4   | 0x52b24 |

La instrucción siguiente es **call sym.test**, la cual carga en el stack la dirección de la siguiente instrucción dentro de la función main. Acto seguido, brincamos a sym.test que inicia en _0x0804846e_.

| dirección | valor      |
|-----------|------------|
| ESP       | 0x08048511 |
| ESP+0x4   | num        |
| ESP+0x8   | 0x52b24    |

> Cuando retornemos de esta función, las siguientes instrucciones corresponden al _return 0;_ del programa.

La idea general de como seria esto implementado en C seria como el siguiente código:

```c
int main(int argc, char **argv, char **envp){
  int num, i, j;
  printf("IOLI Crackme Level 0x03\n");
  printf("Password: ");
  scanf("%d", &num);

  i = 90;
  j = 492;
  i += j;
  i *= i;
  j = i;

  test(num, j);

  return 0;
}
```


### Función test

Analicemos lo que hace la función test, para ello damos doble click izq sobre el nombre la función para ir a ese lugar. La función completa es la siguiente:

![Función test](/img/cutter04/dis11.png)

Ahora enfocándonos en la primera parte de la función, el stack frame:

![Stack Frame - test](/img/cutter04/dis12.png)

Esta describe que se salve el EBP en el stack, después asignamos EBP = ESP. Recordar que el call sube la ubicación a ESP de la siguiente instrucción que continua después del call:

| dirección | valor      |
|-----------|------------|
| ESP       | EBP        |
| ESP+0x4   | 0x08048511 |
| ESP+0x8   | num        |
| ESP+0xc  | 0x52b24    |

Y se resta 8 direcciones al ESP. Actualizando nuestra tabla imaginaria del stack:

| dirección | valor      |
|-----------|------------|
| ESP-0x8   | EBP        |
| ESP-0x4   | 0x08048511 |
| ESP       | num        |
| ESP+0x4   | 0x52b24    |

Finalizamos el stack frame aquí en _0x08048471_, y haciendo las resoluciones, EBP = ESP-0x8. Esto es importante para analizar la siguiente parte.

![valor ingresado, valor a comparar](/img/cutter04/dis13.png)

Ahora lo que vemos es que EAX recibe el valor de arg_8h, el cual equivale a EBP+0x8, osea el valor num de main.

Después hacemos un cmp entre EAX (num) y arg_ch, el cual equivale a EBP+0xc, osea el valor 0x52b24 (2do argumento)

> Renombraremos arg_8h como num1 y arg_ch como num2.

![rename nums](/img/cutter04/dis17.png)

Esto nos hace una clara idea de que acabamos de ejecutar un _if_:

```c
if (num1 == num2)
```

Continuemos con la siguiente instrucción:

![Verdadero](/img/cutter04/dis14.png)

Si num1 es igual a num2 (**338724**), entonces subimos el string _str.Lqydolg_Sdvvzrug_ al ESP:

| dirección | valor                |
|-----------|----------------------|
| ESP-0x8   | EBP                  |
| ESP-0x4   | 0x08048511           |
| ESP       | str.Lqydolg_Sdvvzrug |
| ESP+0x4   | num                  |
| ESP+0x8   | 0x52b24              |

Y ejecutamos __call sym.shift__

| dirección | valor                |
|-----------|----------------------|
| ESP-0x8   | EBP                  |
| ESP-0x4   | 0x08048511           |
| ESP       | 0x08048488           |
| ESP+0x4   | str.Lqydolg_Sdvvzrug |
| ESP+0x8   | num                  |
| ESP+0xc   | 0x52b24              |

Con esto hemos brincado a otra función, por lo que dejaremos este caso hasta aquí.

> Al retornar, unicamente ejecutaríamos el _jmp 0x08048496_.

![Falso](/img/cutter04/dis15.png)

Si num1 es diferente a num2 (**338724**), entonces subimos el string _str.Sdvvzrug_RN_ al ESP:

| dirección | valor           |
|-----------|-----------------|
| ESP-0x8   | EBP             |
| ESP-0x4   | 0x08048511      |
| ESP       | str.Sdvvzrug_RN |
| ESP+0x4   | num             |
| ESP+0x8   | 0x52b24         |

Y ejecutamos __call sym.shift__

| dirección | valor           |
|-----------|-----------------|
| ESP-0x8   | EBP             |
| ESP-0x4   | 0x08048511      |
| ESP       | 0x08048488      |
| ESP+0x4   | str.Sdvvzrug_RN |
| ESP+0x8   | num             |
| ESP+0xc   | 0x52b24         |

Con esto hemos brincado a otra función, por lo que dejaremos este caso hasta aquí.

![if completo](/img/cutter04/dis16.png)

Finalmente, la vista completa; sea como sea, llamamos a shift con diferentes strings. Uno para el caso verdadero otro para el caso falso. Después de terminar el if, retornamos a **main** con las insutrcciones *leave* y *ret*.

El codigo en C pudiera ser el siguiente:

```C
void test(int num1, unsigned int num2){
  if (num1 == num2) shift("Lqydolg#Sdvvzrug$");
  else shift("Sdvvzrug#RN$$$#=,");
}
```


### Función shift

Toca turno ahora a la función shift, que hasta donde conocemos, recibe de argumento un string. Veamos que tenemos en toda la función:

![Funcion shift](/img/cutter04/dis21.png)

Se ve mas extensa que la anterior función test. Comencemos por el stack frame:

![Stack frame - shift](/img/cutter04/dis22.png)

Tenemos gracias al análisis de la función, el argumento char s* y tres variables tipo int.

Salvamos el EBP actual en el stack y le asignamos el valor de ESP. Finalizamos con mover el ESP unas 0x98 direcciones ... 152 posiciones nada mas.

| dirección | valor           |
|-----------|-----------------|
| ESP-0xa0  | EBP.main        |
| ESP-0x9c  | 0x08048511      |
| ESP-0x98  | EBP.test        |
| ESP-0x94  | 0x08048488      |
| ESP-0x90  | str.Sdvvzrug_RN |
| ESP-0x8c  | num             |
| ESP-0x88  | 0x52b24         |
| ...       | ...             |
| ESP       | 0x00000000      |

![strlen](/img/cutter04/dis23.png)

En el siguiente bloque de instrucciones, movemos a ESP-0x7c el valor de 0, subimos a EAX el string correspondiente (EBP+0x8 aka ESP-0x90) y guardamos dicho valor en ESP (ESP vale ahora EAX, osea s)

Realizamos la llamada a [strlen](https://www.programiz.com/c-programming/library-function/string.h/strlen) vía **call sym.imp.strlen**. Si conocemos un poco esta función sabemos que debe devolver un valor natural, el cual se guarda en EAX.

![i <= len(s)](/img/cutter04/dis24.png)

Las siguientes instrucciones hacen lo siguiente; comparo si ESP-0x7c (actualmente con valor de 0) es igual o mayor al valor retornado de strlen de s. Luego tomo la decisión en el jae (jmp if great o equal unsigned), siendo verdadero brincamos, siendo falso continuamos con la siguiente instrucción.

Esto es una clara referencia a un tipo de loop que sera mas claro cuando encontremos el jmp de retorno a este punto de comparación y salida.

La comparación que tenemos entre manos es:

    if (0 < strlen(s))

Continuemos al siguiente bloque.

![incrementa el index](/img/cutter04/dis25.png)

El siguiente bloque carga ESP-0x78 en EAX, y después salva el valor de ESP-0x78 en EDX. La idea es después sumar ESP-0x7c (actualmente con valor de 0) en ESP-0x78, aquí es donde haríamos una iteración de ESP-0x78 si el valor de ESP-0x7c cambia. Después guardaríamos ESP-0x7c (actualmente con valor de 0) en EAX para el siguiente bloque.

Esto seria como el siguiente pseudocódigo para la primera pasada:

    EDX = ESP-0x78 + 0
    EAX = 0

![shift!](/img/cutter04/dis26.png)

Continuamos con las siguientes instrucciones, la comienza con la suma de ESP-0x78 (valor 0) con la ubicación del string que esta en .rodata (recordar que depende del valor verdadero o falso del if), si nos fijamos esto seria iterar la ubicación del carácter del string, algo como cuando tenemos un s\[i\] dentro de un código...

Ok, continuando con la instrucción 2, tenemos algo interesante, **movzx eax, byte \[eax\]** toma el primer byte de la ubicación de s y lo guarda en EAX. Lo demás lo llena de ceros.

Después, **sub al, 3** resta 3 al valor almacenado en EAX, por ejemplo si su valor era 0x43 (C) ahora valdrá 0x41 (A).

La siguiente instrucción salva este byte en EDX, es decir, si se convirtió a 0x41, este valor se salva en el primer byte de EDX.

![final del loop](/img/cutter04/dis27.png)

Continuamos cargando la ESP-0x7c en EAX, incrementamos el valor de ESP-0x7c (antes 0, ahora 1) y salvamos en EAX. Ejecutamos un jmp de retorno a 0x08048424 (donde empezamos a preparamos para el strlen).

Con esto finalizamos el loop, pero ... ¿que rayos paso?

Si tuviera que traducirlo a código en C seria como el siguiente:

```c
char buf[100];
int i;

for (i = 0; i < strlen(s); i++)
{
  buf[i] = (char) s[i] - 3;
}
```

Yo se que en ninguna parte se crea ese buf de ese tamaño definido ni que tampoco se hace el cast de char de operaciones en entero, pero lo agregue para darle un poco de practicidad.

![string final](/img/cutter04/dis28.png)

Después de acabar este for, salvamos la dirección ESP-0x78 (buf) en EAX. Sumamos el valor de ESP-0x7c (ahora strlen(s)-1) sobre el valor de EAX (la dirección ESP-0x78).

La ultima instrucción, **mov byte\[eax\], 0** escribe el byte 0x00 sobre la ultima ubicación para cortar el _buffer_ y de esa manera poner el carácter null. Ahora si, tenemos un string hecho y derecho en el buffer.

![printf final](/img/cutter04/dis29.png)

Bueno, tenemos un string en el buffer, retornamos ESP-0x78 a EAX con la primera instrucción, luego con la segunda guardamos ese valor (la dirección del stack) sobre ESP-0x4 y finalmente en la tercera instrucción cargamos en el stack el string "%s\\n".
> Me ahorre las vueltas a la ubicación en memoria para decodificar el string constante que recibe printf

Por supuesto esto fue un preludio para la llamada a printf, donde el primer argumento define que se recibirá un string y el segundo argumento es el string en si que hemos manejado de buffer.

Renombrando todas nuestras variables tenemos un código en C como el siguiente:

```c
void shift(char *s){
  char buf[100];
  int i;

  for (i = 0; i < strlen(s); i++) buf[i] = (char) s[i] - 3;
  printf("%s\n", buf);
}
```

Renombrando en el disassembly también nos queda de la siguiente manera:

![funcion shift final](/img/cutter04/dis20.png)

## Grafico

Los grafos de nuestras 3 funciones son:

### Función main
![Graph main](/img/cutter04/graph-main.png)

### Función test
![Graph test](/img/cutter04/graph-test.png)

### Función shift
![Graph shift](/img/cutter04/graph-shift.png)


## Validación

Hemos llegado a la parte final, donde validamos la conclusión a la que hemos realizado después del análisis estático, en la cual la contraseña que debemos ingresar en el crackme es **338724**, misma del reto anterior.

Validemos:

![Validación](/img/cutter04/validacion.png)

Excelente! Hemos resuelto el cuarto crackme de IOLI.

---
Bueno eso ha sido todo de momento, espero que les haya gustado.
Saludos,
