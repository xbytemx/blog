---
title: "Secuma2018 - Writeups diversos"
date: 2018-11-20T22:54:21-06:00
draft: false
tags: ["networking","crypto","forensics","stego","reversing"]
categories: ["ctf"]
---

El pasado 15 de noviembre, sec/UMA y Bitup realizaron un Capture The Flag (CTF) online y presencial en la universidad de Málaga, España. Afortunadamente pude participar de manera remota resolviendo algunos retos, de los cuales comparto la solución a dichos retos. Las categorías fueron Reversing, Web, Stego, Forensics, Crypto, Networking y Quizz.

> Omito las soluciones a los retos de Web ya que aun no han salido de manera oficial los writeups. Quizz lo omito porque google+^f.

## Reversing
### cs30 (150pts)

Se ha encontrado un ejecutable en los servidores de E-Corp tras el ataque DDoS que recibieron hace unos días. Tenemos razones para pensar que hay un miembro infiltrado de la organización criminal en ECorp. Como miembros de allsafe cybersecurity, debemos conseguir acceder al archivo ya que creemos que podría ser un mensaje para su infiltrado.

---

El primer reto nos entrega un ejecutable, el cual tras validarlo con file veremos que se trata de un ejecutable ELF64:

![file over fsociety](/img/secuma18/r1-file.png)

Tras esto, abrimos el binario en radare2 y le damos una pasada de análisis experimental.

![r2 AAAA](/img/secuma18/r1-aaaa.png)

Analicemos los imports de este binario:

Num | Vaddr      | Bind   | Type   | Name                                                                                                                                                                                                                                                           |
----|------------|--------|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--
1   | 0x00000e50 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::compare(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)const                                                                         |
2   | 0x00000e60 | GLOBAL | FUNC   | sprintf                                                                                                                                                                                                                                                        |
3   | 0x00000e70 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::c_str()const                                                                                                                                                             |
4   | 0x00000000 | WEAK   | FUNC   | __cxa_finalize                                                                                                                                                                                                                                                 |
5   | 0x00000e80 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)                                                                         |
6   | 0x00000e90 | GLOBAL | FUNC   | memset                                                                                                                                                                                                                                                         |
7   | 0x00000ea0 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string()                                                                                                                                                          |
8   | 0x00000eb0 | GLOBAL | FUNC   | memcpy                                                                                                                                                                                                                                                         |
9   | 0x00000ec0 | GLOBAL | FUNC   | __cxa_atexit                                                                                                                                                                                                                                                   |
10  | 0x00000ed0 | GLOBAL | FUNC   | sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::operator<<<char,std::char_traits<char>,std::allocator<char>>(std::basic_ostream<char,std::char_traits<char>>&,std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&) |
11  | 0x00000ee0 | GLOBAL | FUNC   | sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>>&,charconst*)                                                                                                   |
12  | 0x00000ef0 | GLOBAL | FUNC   | sym.imp.std::allocator<char>::~allocator()                                                                                                                                                                                                                     |
13  | 0x00000f00 | GLOBAL | FUNC   | __stack_chk_fail                                                                                                                                                                                                                                               |
14  | 0x00000f10 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator=(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>&&)                                                                                |
15  | 0x00000f20 | GLOBAL | FUNC   | sym.imp.std::basic_istream<char,std::char_traits<char>>&std::operator>><char,std::char_traits<char>,std::allocator<char>>(std::basic_istream<char,std::char_traits<char>>&,std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>&)      |
16  | 0x00000f30 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(charconst*,std::allocator<char>const&)                                                                                                                      |
17  | 0x00000f40 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string()                                                                                                                                                           |
18  | 0x00000f50 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::length()const                                                                                                                                                            |
19  | 0x00000f60 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::Init()                                                                                                                                                                                                                            |
20  | 0x00000000 | GLOBAL | FUNC   | __gxx_personality_v0                                                                                                                                                                                                                                           |
21  | 0x00000000 | WEAK   | NOTYPE | _ITM_deregisterTMCloneTable                                                                                                                                                                                                                                    |
22  | 0x00000f70 | GLOBAL | FUNC   | _Unwind_Resume                                                                                                                                                                                                                                                 |
23  | 0x00000f80 | GLOBAL | FUNC   | sym.imp.std::allocator<char>::allocator()                                                                                                                                                                                                                      |
24  | 0x00000000 | GLOBAL | FUNC   | __libc_start_main                                                                                                                                                                                                                                              |
25  | 0x00000000 | WEAK   | NOTYPE | __gmon_start __                                                                                                                                                                                                                                                 |
26  | 0x00000000 | WEAK   | NOTYPE | _ITM_registerTMCloneTable                                                                                                                                                                                                                                      |
27  | 0x00000000 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::~Init()                                                                                                                                                                                                                           |
4   | 0x00000000 | WEAK   | FUNC   | __cxa_finalize                                                                                                                                                                                                                                                 |
20  | 0x00000000 | GLOBAL | FUNC   | __gxx_personality_v0                                                                                                                                                                                                                                           |
21  | 0x00000000 | WEAK   | NOTYPE | _ITM_deregisterTMCloneTable                                                                                                                                                                                                                                    |
24  | 0x00000000 | GLOBAL | FUNC   | __libc_start_main                                                                                                                                                                                                                                              |
25  | 0x00000000 | WEAK   | NOTYPE | __gmon_start __                                                                                                                                                                                                                                                 |
26  | 0x00000000 | WEAK   | NOTYPE | _ITM_registerTMCloneTable                                                                                                                                                                                                                                      |
27  | 0x00000000 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::~Init()                                                                                                                                                                                                                           |


Listamos las funciones las cuales tenemos lo siguiente:

direccion  | nbbs | size     | nombre
-----------|------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
0x00000249 | 7    | 67->112  | fcn.00000249
0x00000e28 | 3    | 23       | sym._init
0x00000e50 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char __::compare_std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char__const__const
0x00000e60 | 1    | 6        | sym.imp.sprintf
0x00000e70 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char __::c_str__const
0x00000e80 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char __::basic_string_std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char__const
0x00000e90 | 1    | 6        | sym.imp.memset
0x00000ea0 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char __::_basic_string
0x00000eb0 | 1    | 6        | sym.imp.memcpy
0x00000ec0 | 1    | 6        | sym.imp.__cxa_atexit
0x00000ed0 | 1    | 6        | sym.std::basic_ostream_char_std::char_traits_char___std::operator___char_std::char_traits_char__std::allocator_char___std::basic_ostream_char_std::char_traits_char____std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char__const
0x00000ee0 | 1    | 6        | sym.std::basic_ostream_char_std::char_traits_char___std::operator___std::char_traits_char___std::basic_ostream_char_std::char_traits_char____charconst
0x00000ef0 | 1    | 6        | sym.std::allocator_char_::_allocator
0x00000f00 | 1    | 6        | sym.imp.__stack_chk_fail
0x00000f10 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char\__::operator__std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char
0x00000f20 | 1    | 6        | sym.std::basic_istream_char_std::char_traits_char___std::operator___char_std::char_traits_char__std::allocator_char___std::basic_istream_char_std::char_traits_char____std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char
0x00000f30 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char\__::basic_string_charconst__std::allocator_char_const
0x00000f40 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char\__::basic_string
0x00000f50 | 1    | 6        | sym.std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char\__::length__const
0x00000f60 | 1    | 6        | sym.std::ios_base::Init::Init
0x00000f70 | 1    | 6        | sym.imp._Unwind_Resume
0x00000f80 | 1    | 6        | sym.std::allocator_char_::allocator
0x00000f90 | 1    | 6        | sub.__cxa_finalize_f90
0x00000fa0 | 1    | 43       | entry0
0x00000fd0 | 4    | 50->40   | sym.deregister_tm_clones
0x00001010 | 4    | 66->57   | sym.register_tm_clones
0x00001060 | 5    | 58->51   | sym.__do_global_dtors_aux
0x000010a0 | 1    | 10       | entry1.init
0x000010aa | 4    | 65       | sym.check_std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char___std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char
0x000010eb | 3    | 522      | sym.flag
0x000012f5 | 8    | 634->498 | main
0x0000156f | 4    | 73       | sub.std::ios_base::Init._Init_56f
0x000015b8 | 1    | 21       | sym._GLOBAL__sub_I__Z5checkNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEES4
0x000015ce | 1    | 27       | sym.MD5::MD5
0x000015ea | 1    | 95       | sym.MD5::MD5_std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char__const
0x0000164a | 1    | 84       | sym.MD5::init
0x0000169e | 4    | 175      | sym.MD5::decode_unsignedint__unsignedcharconst__unsignedint
0x0000174e | 4    | 215      | sym.MD5::encode_unsignedchar__unsignedintconst__unsignedint
0x00001826 | 1    | 1021     | sym.MD5::transform_unsignedcharconst
0x0000247e | 9    | 306      | sym.MD5::update_unsignedcharconst__unsignedint
0x000025b0 | 1    | 44       | sym.MD5::update_charconst__unsignedint
0x000025dc | 8    | 279      | sym.MD5::finalize
0x000026f4 | 9    | 307->255 | sym.MD5::hexdigest_abi:cxx11___const
0x00002827 | 2    | 98->103  | sym.operator___std::ostream__MD5
0x000028a3 | 1    | 5        | loc.000028a3
0x000028af | 3    | 113      | sym.md5_std::__cxx11::basic_string_char_std::char_traits_char__std::allocator_char
0x00002920 | 4    | 73       | sym.__static_initialization_and_destruction_0_int_int
0x00002969 | 1    | 21       | sym._GLOBAL__sub_I__ZN3MD5C2Ev
0x0000297e | 1    | 33       | sym.MD5::F_unsignedint_unsignedint_unsignedint
0x000029a0 | 1    | 33       | sym.MD5::G_unsignedint_unsignedint_unsignedint
0x000029c2 | 1    | 24       | sym.MD5::H_unsignedint_unsignedint_unsignedint
0x000029da | 1    | 26       | sym.MD5::I_unsignedint_unsignedint_unsignedint
0x000029f4 | 1    | 24       | sym.MD5::rotate_left_unsignedint_int
0x00002a0c | 1    | 106      | sym.MD5::FF_unsignedint__unsignedint_unsignedint_unsignedint_unsignedint_unsignedint_unsignedint
0x00002a76 | 1    | 106      | sym.MD5::GG_unsignedint__unsignedint_unsignedint_unsignedint_unsignedint_unsignedint_unsignedint
0x00002ae0 | 1    | 89       | sym.MD5::HH_unsignedint__unsignedint_unsignedint_unsignedint_unsignedint_unsignedint_unsignedint
0x00002b39 | 1    | 3        | fcn.00002b39
0x00002b4a | 1    | 106      | sym.MD5::II_unsignedint__unsignedint_unsignedint_unsignedint_unsignedint_unsignedint_unsignedint
0x00002bc0 | 4    | 101      | sym.__libc_csu_init
0x00002c30 | 1    | 2        | sym.__libc_csu_fini
0x00002c34 | 1    | 9        | sym._fini

Tras analizar el tamaño de las funciones y reconocer que estamos tratando con un programa en C++, vamos a la función main a ver que sucede:

![pdf de main](/img/secuma18/r1-pdf-main.png)

Lo primero que resalta es el string str.8a24367a1f46c141048752f2d5bbd14b, tras recordar que varias de las funciones tienen la palabra md5 en sus nombres, esto pareciera algún indicativo hacia la flag. Continuemos sobre las instrucciones.

![alloc](/img/secuma18/r1-alloc.png)

Las primeras instrucciones se tratan de allocators que reservan la memoria para los strings que usa la funcion main. Podemos ver por ejemplo que RBP-0xc0 tiene el valor del string en cuestión que estábamos revisando en el bloque pasado. Las instrucciones son las siguientes:

![ostream](/img/secuma18/r1-printf.png)

El siguiente bloque se encarga de imprimir en pantalla el mensaje "Introduzca la contraseña: " y "\\n"

![istream](/img/secuma18/r1-scanf.png)

El siguiente se encarga de tomar el valor introducido y guardarlo en una variable tipo string ubicada en RBP-0xa0.

![reasigna](/img/secuma18/r1-asigna.png)

Esta variable es reasignada para conservar la original y trabajar con una copia.

![md5 del valor ingresado](/img/secuma18/r1-md5.png)

Como se podía intuir, se le pasa la variable con el valor ingresado a la función md5, la cual almacena el resultado en RBP-0x40

![Muevo y limpio](/img/secuma18/r1-clean.png)

Después el valor de RBP-0x40 pasa a ser de RBP-0x80, tenemos unos procesos de limpieza y imprimimos "\\n" para no llenar la pantalla.

![check](/img/secuma18/r1-check.png)

En esta parte, asigno local_c0h (string hasta arriba en main) y local_80h hacia local_40h y local_60h. Ambas se pasan a la función check. Veamos que hace esta función:

![función check](/img/secuma18/r1-pdf-check.png)

Básicamente compara sus dos argumentos y devuelvo 1 si son iguales o 0 si son diferentes, invertido el asunto. Continuamos con el siguiente bloque de instrucciones:

![función check](/img/secuma18/r1-if.png)

Empezamos por salvar el valor de EAX en EBX, tenemos un proceso de limpieza y continuamos con un test sobre BL, que es básicamente el valor 1 si el string es igual o 0 si es diferente. Si check fue 1, la siguiente parte je no brincara y nos mandara a sym.flag (tal vez aquí se genera la flag). Si el check por otra parte devuelve 0, entonces el je si se ejecutara y nos mandara un mensaje diciendo que es incorrecto.

A partir de aquí tenemos dos formas, romper el hash md5 o romper la función flag. Hagamos ambas :D

1. Cracker la hash md5

En mi caso para identificar el valor que genero el hash, use hashcat de la siguiente manera:

![función check](/img/secuma18/r1-pass.png)

2. Reversing de función flag

![función flag1](/img/secuma18/r1-flag1.png)

Lo primero que vemos al llegar a la función flag es que se guarda mucha información en el stack, desde RBP-0x60 hasta RBP-0x18, guardando muchos caracteres que son reconocibles.

![función flag1](/img/secuma18/r1-flag2.png)

Después vemos que hay  un string raro, diciendo "OnE_oRzr\\n" que pareciera ser con lo que trabaja en las siguientes funciones. Trabaja con substrings de ese string mas el resultado que se almacena en RAX.

![función flag1](/img/secuma18/r1-flag3.png)

Esto sigue hasta el final de la función, donde antes de salir se valida el canario de la muerte. También tenemos un bonito NOP haciendo bulto.

Para agilizar este resultado, decidí resolverlo por análisis dinámico, iniciando el programa y colocando el RIP sobre esta función, ya que no requiere nada previo ni devuelve algún valor.

![función flag1](/img/secuma18/r1-flag4.png)

Usamos ese NOP como breakpoint y la flag se libera ante nosotros.

__secuma18{OnE_oR_zEro}__

### brEakCORP (300pts)

If could you break the restriction, I will be so happy. I think that you can do it.

---

Para el segundo reto de reversing, tenemos nuevamente un ejecutable ELF64:

![file de brEakCORP](/img/secuma18/r2-fil2.png)

Cargamos el ejecutable en radare2 y comenzamos por el análisis experimental:

![funciones de brEakCORP](/img/secuma18/r2-load.png)

Listamos las funciones en donde no vemos otras funciones locales aparte de main.

![imports de brEakCORP](/img/secuma18/r2-imports.png)

Los imports nos dicen lo mismo de las funciones, no hay nada extraño hasta el momento. Analicemos main:

![bloque1 de brEakCORP](/img/secuma18/r2-main1.png)

Lo primero que notamos, es que hay muchos caracteres cargándose en sobre el stack, una señal clara que aquí se utilizaran para alguna función.

![bloque2 de brEakCORP](/img/secuma18/r2-main2.png)

Termina de subir cosas al stack y se guardan algunos enteros. Interesante que local_ch y local_a8h se inician en 0, mientras que local_8h se inicia en 4815159197. Después tenemos un brinco muy adelante hasta 0x400728.

![bloque3 de brEakCORP](/img/secuma18/r2-main3.png)

Lo que tenemos a continuación es probablemente un ciclo for, ya que mientras local_ch (que vale 0) sea menor igual a 26, no podrá salir del bucle.

![bloque4 de brEakCORP](/img/secuma18/r2-main4.png)

Termina nuestro bucle de operaciones matemáticas y llamamos a printf, scanf y fflush, para poner en pantalla el mensaje de "Ecorp Enter Access Code: " y tomar el valor entero que ingrese el usuario. Este se almacena en local_a8h.

![bloque5 de brEakCORP](/img/secuma18/r2-main5.png)

Continuamos con un cmp, en donde se evaluá si el valor que ingresamos es igual a local_8h.

Si es verdadero, local_18h es igual a 0 y brincamos a 0x4007a9, en donde se compara local_18h contra 26, que es la longitud del flag. Básicamente construye el flag y lo imprime en pantalla al finalizar.

En caso falso, imprime "Access_Denied" y sale de main devolviendo 0.

Continuemos ahora con el análisis dinámico.

Si nos fijamos bien, conociendo el valor de local_8h en el cmp, pudiéramos descifrar la contraseña necesaria para imprimir la flag, así que coloquemos un breakpoint ahí:

![analisis dinamico de brEakCORP](/img/secuma18/r2-dyn.png)

Validemos que este sea el valor y que podamos imprimir la flag:

![analisis dinamico de brEakCORP](/img/secuma18/r2-flag.png)

__secuma18{eVeRyThInghaPPendSBYaReaSoN}__


### Strange file (500pts)

Tras estar toda la noche pwneando una máquina de HTB, encuentras un extraño archivo en el directorio /tmp de tu máquina.

---

Para este reto descargamos el archivo anexo [SimulacroySimulacion](https://secuma.bitupalicante.es/files/211afb4d195db282570420559b94a56c/SimulacroySimulacion), el cual analizamos con file:

![file de simulacro](/img/secuma18/r3-file.png)

Como podemos ver tratamos otra vez un archivo ELF64, el cual si nos llama la atención el resultado de file, nos indica que no tiene section header... uy.

![ejecución de simulacro](/img/secuma18/r3-exec1.png)

Tras ejecutar el binario, nos pide la contraseña y sencillamente nos comenta si es o no correcto.

Veamos que podemos ver por strings:

![strings de simulacro](/img/secuma18/r3-strings.png)

Vuelve a aparecer el mensaje de copyright, y también algo que parece la flag, "uma18{".

![rabin de simulacro](/img/secuma18/r3-rabin1.png)

Rabin2 nos entrega información interesante, tal como que no existen imports, no hay secciones cosa muy común con los packers. Veamos la entropía para confirmar si el binario se encuentra empaquetado. Comencemos por cargar el binario en radare2 para su análisis estático:

![r2load de simulacro](/img/secuma18/r3-r2load.png)

Nuevamente los warnings que ya hemos visto.

Veamos la entropía:

![entropy de simulacro](/img/secuma18/r3-entropy.png)

Esto nos confirma como todo se va de un lado, lo cual implica que hay un pedazo que desempaca.

Ahora que sabemos que se trata de un packer, analicemos nuevamente el string del copyright:

![strings de aaa de simulacro](/img/secuma18/r3-aaa.png)

Eso de las aaa, se parece a otro tipo de packer, busquemos en google el copyright pero si las AAA, por lo que llegamos a esta linea en el código de un packer en particular, UPX.

https://github.com/upx/upx/blob/master/src/packer.cpp#L1064

Tras identificar que estamos tratando con UPX, probemos a descomprimir el binario directamente con el comando upx:

![upx failed de simulacro](/img/secuma18/r3-upx1.png)

El binario no puede ser descomprimido... Probemos a sustituir esas fastidiosas AAA por UPX. En bless sera mas fácil esta sustitución. Guardamos el archivo como SimulacroySimulacion.upx

![bless de simulacro](/img/secuma18/r3-bless1.png)

Tratemos de identificarlo con file y upx:

![upx listed de simulacro](/img/secuma18/r3-upx2.png)

Como podemos apreciar, el archivo es ahora correctamente identificado. Procedamos a descomprimirlo:

![upx unpcacked de simulacro](/img/secuma18/r3-upx3.png)

Tras la descompresión, carguemos el ejecutable en radare2:

![load upx unpcacked de simulacro](/img/secuma18/r3-load2.png)

Como podemos ver ya no tenemos los mismos mensajes de warning o de información faltante. El único mensaje de warning que tenemos es acerca del tamaño del bloque en cierta función.

Veamos la tabla de Imports

Num | Vaddr      | Bind   | Type   | Name
----|------------|--------|--------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1   | 0x00000f30 | GLOBAL | FUNC   | isspace
2   | 0x00000f40 | GLOBAL | FUNC   | sprintf
3   | 0x00000f50 | GLOBAL | FUNC   | strstr
4   | 0x00000f60 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::compare(charconst*)const
5   | 0x00000f70 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::c_str()const
6   | 0x00000000 | WEAK   | FUNC   | __cxa_finalize
7   | 0x00000f80 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)
8   | 0x00000f90 | GLOBAL | FUNC   | memset
9   | 0x00000fa0 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string()
10  | 0x00000fb0 | GLOBAL | FUNC   | open
11  | 0x00000fc0 | GLOBAL | FUNC   | memcpy
12  | 0x00000fd0 | GLOBAL | FUNC   | __cxa_atexit
13  | 0x00000fe0 | GLOBAL | FUNC   | sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::operator<<<char,std::char_traits<char>,std::allocator<char>>(std::basic_ostream<char,std::char_traits<char>>&,std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)
14  | 0x00000ff0 | GLOBAL | FUNC   | sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>>&,charconst*)
15  | 0x00001000 | GLOBAL | FUNC   | sym.imp.std::allocator<char>::~allocator()
16  | 0x00001010 | GLOBAL | FUNC   | __stack_chk_fail
17  | 0x00001020 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator=(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>&&)
18  | 0x00001030 | GLOBAL | FUNC   | sym.imp.std::basic_istream<char,std::char_traits<char>>&std::operator>><char,std::char_traits<char>,std::allocator<char>>(std::basic_istream<char,std::char_traits<char>>&,std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>&)
19  | 0x00001040 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(charconst*,std::allocator<char>const&)
20  | 0x00001050 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string()
21  | 0x00001060 | GLOBAL | FUNC   | read
22  | 0x00001070 | GLOBAL | FUNC   | sym.imp.std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::length()const
23  | 0x00001080 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::Init()
24  | 0x00000000 | GLOBAL | FUNC   | __gxx_personality_v0
25  | 0x00000000 | WEAK   | NOTYPE | _ITM_deregisterTMCloneTable
26  | 0x00001090 | GLOBAL | FUNC   | _Unwind_Resume
27  | 0x000010a0 | GLOBAL | FUNC   | sym.imp.std::allocator<char>::allocator()
28  | 0x00000000 | GLOBAL | FUNC   | __libc_start_main
29  | 0x00000000 | WEAK   | NOTYPE | __gmon_start__
30  | 0x00000000 | WEAK   | NOTYPE | _ITM_registerTMCloneTable
31  | 0x00000000 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::~Init()
6   | 0x00000000 | WEAK   | FUNC   | __cxa_finalize
24  | 0x00000000 | GLOBAL | FUNC   | __gxx_personality_v0
25  | 0x00000000 | WEAK   | NOTYPE | _ITM_deregisterTMCloneTable
28  | 0x00000000 | GLOBAL | FUNC   | __libc_start_main
29  | 0x00000000 | WEAK   | NOTYPE | __gmon_start__
30  | 0x00000000 | WEAK   | NOTYPE | _ITM_registerTMCloneTable
31  | 0x00000000 | GLOBAL | FUNC   | sym.imp.std::ios_base::Init::~Init()

También tenemos Exports...

Num | Paddr      | Vaddr      | Bind   | Type   | Size | Name
----|------------|------------|--------|--------|------|---------------------------------------------------------------------------------------------
054 | 0x000028c2 | 0x000028c2 | GLOBAL | FUNC   | 279  | MD5::finalize()
055 | 0x00001930 | 0x00001930 | GLOBAL | FUNC   | 84   | MD5::init()
058 | 0x00002764 | 0x00002764 | GLOBAL | FUNC   | 306  | MD5::update(unsignedcharconst*,unsignedint)
060 | ---------- | 0x00204060 | GLOBAL | NOTYPE | 0    | _edata
061 | 0x000018b4 | 0x000018b4 | GLOBAL | FUNC   | 27   | MD5::MD5()
064 | 0x00002f20 | 0x00002f20 | GLOBAL | OBJ    | 4    | _IO_stdin_used
069 | 0x00001571 | 0x00001571 | GLOBAL | FUNC   | 740  | main
071 | 0x00004008 | 0x00204008 | GLOBAL | OBJ    | 0    | __dso_handle
072 | 0x000011ca | 0x000011ca | GLOBAL | FUNC   | 224  | check(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>)
075 | 0x00002f14 | 0x00002f14 | GLOBAL | FUNC   | 0    | _fini
084 | 0x000018d0 | 0x000018d0 | GLOBAL | FUNC   | 95   | MD5::MD5(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)
085 | 0x000010c0 | 0x000010c0 | GLOBAL | FUNC   | 43   | _start
087 | 0x000012aa | 0x000012aa | GLOBAL | FUNC   | 323  | flag(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>)
090 | 0x00000f00 | 0x00000f00 | GLOBAL | FUNC   | 0    | _init
091 | ---------- | 0x00204060 | GLOBAL | OBJ    | 0    | __TMC_END__
094 | ---------- | 0x00204060 | GLOBAL | OBJ    | 272  | _ZSt4cout@@GLIBCXX_3.4
096 | 0x00001b0c | 0x00001b0c | GLOBAL | FUNC   | 3160 | MD5::transform(unsignedcharconst*)
099 | 0x00004000 | 0x00204000 | GLOBAL | NOTYPE | 0    | __data_start
100 | 0x000013ed | 0x000013ed | GLOBAL | FUNC   | 388  | debuggerIsAttached()
101 | 0x00002896 | 0x00002896 | GLOBAL | FUNC   | 44   | MD5::update(charconst*,unsignedint)
102 | ---------- | 0x002042a0 | GLOBAL | NOTYPE | 0    | _end
107 | ---------- | 0x00204060 | GLOBAL | NOTYPE | 0    | __bss_start
108 | 0x00002b95 | 0x00002b95 | GLOBAL | FUNC   | 113  | md5(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>)
110 | 0x00002ea0 | 0x00002ea0 | GLOBAL | FUNC   | 101  | __libc_csu_init
111 | 0x000029da | 0x000029da | GLOBAL | FUNC   | 307  | MD5::hexdigest[abi:cxx11]()const
112 | 0x00001984 | 0x00001984 | GLOBAL | FUNC   | 175  | MD5::decode(unsignedint*,unsignedcharconst*,unsignedint)
114 | 0x00001a34 | 0x00001a34 | GLOBAL | FUNC   | 215  | MD5::encode(unsignedchar*,unsignedintconst*,unsignedint)
118 | 0x00002f10 | 0x00002f10 | GLOBAL | FUNC   | 2    | __libc_csu_fini
119 | ---------- | 0x00204180 | GLOBAL | OBJ    | 280  | _ZSt3cin@@GLIBCXX_3.4
124 | 0x000018b4 | 0x000018b4 | GLOBAL | FUNC   | 27   | MD5::MD5()
125 | 0x00002b0d | 0x00002b0d | GLOBAL | FUNC   | 136  | operator<<(std::ostream&,MD5)
126 | 0x000018d0 | 0x000018d0 | GLOBAL | FUNC   | 95   | MD5::MD5(std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>const&)

De la tabla de imports concluimos que tenemos muchas de las funciones del challenge1, por lo que básicamente trabajaremos con strings y md5 nuevamente.

Analicemos main:

![main1 de simulacro](/img/secuma18/r3-main1.png)

Interesante, en este caso tenemos un mecanismo de protección para que el programa no pueda ser debuggeado, adicional al stack canary que hemos visto en los demás programas.

![main2 de simulacro](/img/secuma18/r3-main2.png)

La primera parte se encarga de definir algunos strings, entre ellos "QwertyPassword" y "s3cuma18{TheRockOrLoL?}". Yo se que este ultimo parece flag, pero no camina como flag, no dice secuma18, dice s3cuma18.

![main3 de simulacro](/img/secuma18/r3-main3.png)

Después el siguiente bloque se encarga de imprimir el mensaje "Introduzca la contraseña: " y el mensaje "\\n", continuamos por tomar el valor y almacenarlo como string en local_a0h. Imprimimos un "\\n" después de tomar el valor string para darle padding.

![main4 de simulacro](/img/secuma18/r3-main4.png)

El valor ingresado lo pasamos como parámetro a la función md5, que genera el hash del valor ingresado y lo almacena en RBP-0x60.

![main5 de simulacro](/img/secuma18/r3-main5.png)

Preparamos el hash que hemos generado y lo enviamos a la función check. Ojo que el challenge1 enviaba ambos parámetros, en este caso solo enviamos el hash.

Continuemos en un momento mas con la función check:

![main6 de simulacro](/img/secuma18/r3-main6.png)

Si check retorna 1, bricamos el jump, si check retorna 0, se ejecuta el jump.

![main7 de simulacro](/img/secuma18/r3-main7.png)

Caso verdadero, o que check retorne 1, llamamos a flag y al retornar, limpiamos y terminamos el programa.

![main8 de simulacro](/img/secuma18/r3-main8.png)

Caso falso desplegamos el valor de incorrecto, limpiamos y terminamos el programa.

Analicemos check:

![check1 de simulacro](/img/secuma18/r3-check1.png)

Apenas iniciamos la función, observamos como se carga una serie de caracteres dentro del stack, iniciando desde RBP-0x30 hasta RBP-0x11, osea 32 caracteres.

![check2 de simulacro](/img/secuma18/r3-check2.png)

Continuamos por cargar el parámetro que recibió la función check y el inicio del string y hacemos un compare. Esto básicamente es lo que sucedía en el reto1, donde comparamos directamente los strings a fin de validar si la pass convertida a hash de md5 es igual a la que esta hardcodeada.

![check3 de simulacro](/img/secuma18/r3-check3.png)

Después que termina la comparación se prueba si es igual a 0 el retorno de compare, si es igual a 0, check devuelve 1, si es 1, check devuelve 0. Esto hace sentido a lo que veíamos que sucedía en main.

Con esto nos queda claro que para identificar la contraseña y pasar el if (sin modificar el binario claro), que nos llevara a la función flag, debemos romper el hash. Para ello primero extraemos el hash el cual es:

      b960720998179dc8ddcf01201fd0dcab

Utilizando hashcat para atacarlo por diccionario:

![hashcat de simulacro](/img/secuma18/r3-hashcat.png)

Ejecutamos ahora el binario e ingresamos la contraseña correcta:

![run de simulacro](/img/secuma18/r3-run.png)

Nos aparace un mensaje que indica que debemos hacer un xor del string "thematrixhasyou" y el hexadecimal 57654c6c436f4d113c0a390931005a390436430b2331.

![flag de simulacro](/img/secuma18/r3-flag.png)

Tras resolver el xor la password se revela.

__secuma18{WeLlCoMeToThEr3AlW0rLD}__


## Stego
### Discographyde (150pts)

When I hack...

There is always a tone for each key, there is always a melody that reminds you of that moment of tension, adrenaline ... everything.

And that makes us unable to avoid hiding our darkest secrets in the deepest of those songs.

---

Comenzamos por descargar el archivo [macquayle.7z](https://mega.nz/#!R8EFkIRT!Umqy3VM0CwxkpnkUMzitIOYclXrMSJ0NpG5txz1xpQM), el cual contiene 2 archivos, el primero es impenetrable.sd2, que se compone de tema de la serie honor de este CTF y el segundo es un archivo oculto llamado ".hint". Iniciamos por impenetrable.sd2

Tras realizar un file y un exiftool notaremos que no hay nada de especial:

![file de impenetrable](/img/secuma18/s1-file.png)

![exiftool de impenetrable](/img/secuma18/s1-exiftool.png)

Luego con sox genere un rapido espectrograma via _sox impenetrable.sd2 -n spectrogram_

![spectrogram de impenetrable via sox](/img/secuma18/s1-spectrogram.png)

El espectrograma no nos revela ningún texto legible o algo que podamos identificar rápido a simplemente vista.

![spectrogram de impenetrable via spek](/img/secuma18/s1-spectrogram2.png)

Trate de imprimirlo con otra herramienta diferente, que en este caso fue spek, pero tampoco pude visualizar algo particular. Aquí fue cuando un pequeño hit de que existía un archivo dentro del audio, lo cual hace sentido viendo la estructura del espectrograma, que es bastante cuadrado, sobre todo esos picos. Esto es una clara señal de un hide por LSB.

Si recordamos, al inicio se nos dieron dos archivos, un wav y un txt, si leemos el contenido del txt, tenemos lo siguiente:

    66736f63696574793030

Que si lo convertimos de hexa a string seria:

![pass de impenetrable](/img/secuma18/s1-xxd.png)

Probamos este texto como la contraseña y usamos steghide para extraer archivos del wav:

![pass de impenetrable](/img/secuma18/s1-flag.png)

__secuma18{Hide_your_criminal_notes}__

### What is poetry? (300pts)

Dices, mientras clavas tu pupila azul en mi pupila.

---

Iniciamos este reto por descargar el archivo What_is_poetry.7z de la pagina del reto el cual al descomprimir, nos entregara un archivo llamado "tyrell.jpg". Si analizamos este archivo con file y exiftool tendremos lo siguiente:

![file de tyrell](/img/secuma18/s2-file.png)

Hasta aquí nada extraño, veamos si hay strings dentro del archivo:

![strings de tyrell](/img/secuma18/s2-strings.png)

Strings nos revela 2 cosas importantes, la primera parte es que tenemos un archivo dentro de esta imagen, y el segundo es una clave morse que esta hasta mero abajo.

Convirtamos esa clave morse a texto:

![morse2ascii de tyrell](/img/secuma18/s2-morse.png)

Ahora usemos _intensito_ como contraseña de steghide:

![steghide extract de tyrell](/img/secuma18/s2-extract.png)

El contenido nos manda a una url de una pagina publica donde tenemos los scripts de televisión. Ahí encontraremos el script del piloto de Mr robot, el archivo "Mr_Robot_1x01_-_Pilot.pdf".

Al abrir este archivo encontraremos muchos diálogos incluidos los que tiene Tyrell y Elliot cuando se conocen. Tomando como referencia "last_word", osea el nombre del archivo, nos vamos hasta la ultima pagina del script:

![ultima pagina del script](/img/secuma18/s2-flag.png)

Aquí podemos ver que la ultima palabra del documento es _BLACK_, la cual resulta ser nuestra flag.

__secuma18{BLACK}__


### 0ld_m3m0r13s (500pts)

My father and I were very close. He was an example to follow for me.

But ... one day I failed him. And from that day he stopped talking to me and looking at me, even the night he died. He did not say anything to me.

I remember ... he always told me that people try hard to hide their fears and shadows, but with a little light, everything is discovered.

---

Tras descargar el archivo r3m3mb3rme.jpg, verificaremos con file y exiftool en busca de información del archivo:

![file de r3m3mb3rme](/img/secuma18/s3-file.png)

Parece fácilmente identificable que tenemos una imagen dentro de otra imagen vía thumbnails.

Continuamos analizado son vía strings:

![strings de r3m3mb3rme](/img/secuma18/s3-strings.png)

Strings nos revela que tenemos una imagen con steghide entre manos, pero también aparece la palabra flag.txt dentro de los strings reconocibles. Hagamos un binwalk para tener mas detalle:

![binwalk de r3m3mb3rme](/img/secuma18/s3-binwalk.png)

Con binwalk logramos extraer un poco mas de información y vemos que tenemos dos archivos reconocibles por sus headers dentro de la imagen, flag.txt y A037.zip.

Tratemos de leer flag.txt y abrir A037.zip:

![7z de r3m3mb3rme](/img/secuma18/s3-7z.png)

Como podemos ver, el archivo flag.txt se encuentra dentro del zip, pero como no tenemos la contraseña no podemos ver su contenido.

Muchas horas despues y un hit inadvertido, la imagen r3m3mb3rme.jpg tiene un pequeño texto dentro de la televisión que dice: _toseeyouagain_

Usando dicho texto como pass del zip:

![flag de r3m3mb3rme](/img/secuma18/s3-flag.png)

__secuma18{m4g1c_curv3s}__


## Forensics
### Your time's running out... (150pts)

![jigsaw joker de mr robot](/img/secuma18/f1-maxresdefault.jpg)

Your computer files have been encrypted. Your photos, videos, documents, etc... But, don´t worry! I have not deleted them, yet. You have 10 hours to decypt this file to get the flag and win!

---

El reto nos entrega un archivo cifrado llamado flag.txt.fun, el cual al leerlo no podemos identificar ni tenemos ningún indicio.


Tras leer la descripción del reto, vemos que esto se parece mucho a un ataque de un ransomware, por lo que buscamos ransomware fun extensión en google nos lleva a un ransomware llamado jigsaw, el cual tiene un mensaje de estafa muy parecido al que tenemos en el reto:

![jigsaw ransom](/img/secuma18/f1-jigsaw-ransom.png)

Tras buscar si existe un decryptor gratuito para este ransom, me tope con el siguiente:

[jigsaw-decrypter](https://www.bleepingcomputer.com/download/jigsaw-decrypter/)

El cual también esta recomendado por INCIBE.

Cargamos el archivo fun y el decryptor en una maquina virtual para realizar la extracción:

![load de jigsaw](/img/secuma18/f1-vm1.png)

Ejecutamos el decryptor sobre la carpeta donde tenemos el archivo fun:

![decrypt de jigsaw](/img/secuma18/f1-vm2.png)

Tras realizar el decrypt del archivo fun, podemos ver claramente la flag:

![flag de jigsaw](/img/secuma18/f1-flag.png)

__secuma18{D3scifr4ndo_4ndo}__

### Rummage (300pts)

Tras una incidencia de seguridad sucedida en Allsafe Cybersecurity hemos detectado que hay comportamientos anómalos en algunos ordenadores. Después de una ardua investigación, los forenses se han hecho con una copia de la memoria RAM de uno de ellos para finalmente llegar a la conclusión de que el culpable se encuentra tras el **registro** de la configuración del usuario. ¿Crees que podrás encontrarlo?

---

Tras descargar el archivo [Rummage .7z](https://mega.nz/#!1okE0SSL!UbiMVRAk54McAW_SxgMCrQG5G00HeUl0DOb6YR-SAt8) y descomprimirlo, tendremos el archivo Rummage.raw. Sabemos por la descripción del reto que tenemos entre manos un archivo de volcado de memoria, por lo que comenzamos a identificar la versión del sistema operativo con volatility:

![imageinfo de rummage](/img/secuma18/f2-imageinfo.png)

Como podemos ver por la identificación del perfil, debemos tener entre manos un Win7 x86 con SP1. Sabemos de antemano que estaremos trabajando con registros, por lo que iniciamos por hacer el dump a archivo de todos los archivos de registros de la imagen:

![dumpregistry de rummage](/img/secuma18/f2-dumpregister.png)

Ahora si procesamos los registros con hivexml en un for, podemos grepear strings de interés, como **secuma**:

![flag de rummage](/img/secuma18/f2-flag.png)

Bingo, hemos encontrado la flag dentro de los registros:

__secuma18{Tyr3ll_W3llick!!}__


## Crypto
### Cl4s1c P4r4n01d (150pts)

"Tenemos que ayudar a la agente DiPierro, ha conseguido interceptar 3 mensajes de algunos usuarios, pero cada uno usa un cifrado distinto. ¿Puedes ayudarla? Se encontró un texto sin cifrar que decía: La union es la clave."

BAnkS: ONSWG5LNMEYTQ62BIJ2WO===

irVInG: Cf_V3j3eDhah_n

ROboT: _K1qryi3}

---

Para este reto, tenemos 3 partes y una llave. Cada texto cifrado viene acompañado de un nombre extraño: BAnkS, irVInG, ROboT...

Si tomamos las mayúsculas de estas palabras tenemos:
BAS, VIG, ROT.

Estando un poco mas familiarizados con algoritmos de criptográfica clásicos (tal como el nombre del reto lo indica), podemos relacionar dichas palabras a BASE, Vigenère y rotation.

El primero se trata de un tipo de encoding, el cual siendo puras mayúsculas me llamo la atención, puesto que el encoding mas famoso, base64, hace uso de mayúsculas y minúsculas para la representación de datos. Esto me llevo a probar al hermano menor, base32:

![base32 de classic](/img/secuma18/c1-base32.png)

Excelente, ya tenemos la primera parte, _secuma18{ABug_.

El siguiente, que se trata de un Vigenère, requiere de una llave, aquí es cuando la palabra "union" entra en juego (la union es la clave)

![vigenere de classic](/img/secuma18/c1-vg.png)

Aunque el output no es claramente visible, realizando los ajustes de mayusculas minusculas y agregando caracteres numeros y de signos tenemos _Is_N3v3rJust_a_

El ultimo bloque de rotación fue el que más me costo trabajo, ya que rot tiene varias presentaciones, tenemos rot13 que es el mas conocido, pero también tenemos rot18, rot5, rot47, cada uno con sus características y los signos incluidos en las rotaciones. El primer delimitador fue "}" y "_", ya que la rotación no los debía incluir, pues forman parte de la flag, ahora solo quedaban los números, mayúsculas y minúsculas. Al probarlo ningún mensaje entendible sobresalió, pero al intentar con rot24:

![rot24 de classic](/img/secuma18/c1-rot24.png)

La ultima parte es _\_M1stak3}_, por lo que uniendo todo tenemos:

__secuma18{ABugIs_N3v3rJust_a_M1stak3}__


### Fucks0ci3ty (300pts)

![Darlene](/img/secuma18/c2-Darlene.jpg)

I think she has to be the key to all this...

---

Nos entregan un archivo llamado "thisnotasecret.txt" con contenido encodeado en base64, al pasarlo a base64 a un archivo tenemos un file tipo openssl cifrado con una pass con salt.

![file de Fucks0ci3ty](/img/secuma18/c2-file.png)

Tras revisar arduamente el archivo imagen de Darlene, no encontré ninguna prueba o evidencia que me indique que algún tipo de llave se encontraba dentro del archivo, por lo que literal empece a usar el nombre del archivo Darlene y darlene para tratar de romper el archivo cifrado.

Para ello, trate de ser mas amplio (gracias al hit del autor) y no solo usar AES, que es el cipher mas utilizado para cifrar documentos, sino aplicarle a todos. Para esta tarea cree rápidamente un for para usar todos los parámetros posibles de ciphers y este fue el resultado:

![file de Fucks0ci3ty](/img/secuma18/c2-flag.png)

__secuma18{N0b0dY_MaKe_R00tk1ts_L1K3_m3}__


## Networking
### Botnet (50pts)

Uno de nuestros sensores IDS ha detectado tráfico anómalo en las redes de Allsafe Cybersecurity. Tras un análisis exhaustivo de los logs, el equipo de forense se ha dado cuenta de que nuestro router forma parte de una famosa botnet y que el vector de infección final viene determinado por un downloader que hace unas peticiones para descargarse el cliente y poder comunicarse con el C&C con la clave secreta dentro del fichero descargado.

En esta misión necesitamos encontrar el fichero para poder continuar con las investigaciones y denunciarlo ante las autoridades pertinentes.

---

Uno de los retos de mayor puntaje y tal vez una de mis categorías favoritas. Iniciamos descargando el pcapng.

![capinfos de botnet](/img/secuma18/n1-capinfos.png)

Empezamos por tomar información sobre el pcapng que tenemos, el cual nos da algunas de sus características de como fue capturado y que capturo. Tenemos 350 paquetes.

![udp de botnet](/img/secuma18/n1-udp.png)

Comenzamos por analizar si tenemos trafico extraño sobre UDP, de manera resumida tenemos que no.

![icmp de botnet](/img/secuma18/n1-icmp.png)

Replicamos el mismo procedimiento sobre el trafico ICMP, no encontrando rastros de data extraña o anormal.

![trafico http de botnet](/img/secuma18/n1-http.png)

Sobre el trafico http, el primer paquete se ve sospechoso, ya que el archivo 7z que descarga, tiene el nombre de una botnet muy famosa, _mirai_.

![detalle del p1 de http de botnet](/img/secuma18/n1-http1.png)

Al analizar el contenido de este primer paquete filtrado, tenemos el host y el path por lo que podemos replicar lo que la maquina infectada estuvo realizando.

![wget de botnet](/img/secuma18/n1-wget.png)

Descargamos el archivo.

![7z fallido de mirai botnet](/img/secuma18/n1-7z1.png)

Tras intentar descomprimirlo, nos topamos con que esta protegido por contraseña. Intentemos romper esta contraseña, con suerte y esta en un diccionario.

![generacion de hash de 7z de botnet](/img/secuma18/n1-7z-hash.png)

Generamos el hash con el script de perl 7z2hashcat.

![hashcat del mirai botnet](/img/secuma18/n1-hashcat.png)

Ingresamos el hash sobre hascat con las opciones indicadas en la documentacion de 7z2hashcat, como pueden ver en mi caso ya la rompi, por lo que me fui directo al show.

![extraccion exitosa de botnet](/img/secuma18/n1-7z2.png)

Extraemos el archivo flag.txt de mirai.7z

![flag de botnet](/img/secuma18/n1-flag.png)

Hemos conseguido la ultima flag de esta entrada. Como nota, el flag no esta en utf-8, ojo.

__secuma18{3vil_C0rp¡!}__


## Agradecimientos
Quiero agradecer a los organizadores de este evento por permitir que personas externas podamos participar. Gracias por hacer una mejor comunidad.
