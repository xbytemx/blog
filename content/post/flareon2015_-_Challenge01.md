---
title: "Flare-On 2015 - Challenge 01"
date: 2019-07-01T22:25:36-05:00
draft: false
tags: ["flareon","revisitado","flareon2015","dotnet","reversing","writeup"]
categories: ["reversing","ctf"]
---

Iniciamos con mas retos de flareon, ya que ya esta muy cerca. Ahora comenzaremos por los retos de 2 en 2 de cada año hasta terminar lo mas posibles.
<!--more-->

Comenzamos por descargar el [zip](http://www.flare-on.com/files/2015_FLAREOn_Challenges.zip) que contiene todos los challenges, para posteriormente ir a cada challenge y resolverlos.

Bien, después de descargar el archivo general y descomprimirlo, debemos tener en nuestra carpeta un archivo llamando `Flare-On_start_2015.exe`, en la carpeta `1`, el cual posteriormente lo pasamos por file y nos indica vía headers que se trata de un 'PE32+ ejecutable con GUI x86_64', por lo que probamos a ejecutarlo en un sandbox, ahí veremos que se trata de Microsoft Cabinet, que nos hace aceptar un EULA de Fireeye, por lo que extraemos su contenido usando `cabextract`:

```bash
xbytemx@laptop:~/flare-on2015/1/$ cabextract Flare-On_start_2015.exe
Extracting cabinet: Flare-On_start_2015.exe
  extracting i_am_happy_you_are_to_playing_the_flareon_challenge.exe

All done, no errors.
```

Nos despliega en pantalla que ha extraído correctamente "i_am_happy_you_are_to_playing_the_flareon_challenge.exe", el cual nuevamente lo pasamos por `file`:

```
xbytemx@laptop:~/flare-on2015/1$ file i_am_happy_you_are_to_playing_the_flareon_challenge.exe
i_am_happy_you_are_to_playing_the_flareon_challenge.exe: PE32 executable (console) Intel 80386, for MS Windows
```

El comando `file` nos indica que se trata de un PE32 ejecutable para consola, por lo que lo analizamos con DIE en busca de más información:

```
xbytemx@laptop:~/flare-on2015/1$ ~/tools/die_lin64_portable/diec.sh i_am_happy_you_are_to_playing_the_flareon_challenge.exe
PE: linker: unknown(8.0)[EXE32,console]
```

Revisando mas información con **`PEbear`** podremos ver que el binario tiene 2 secciones y que únicamente importa una biblioteca (kernel32.dll) con 8 funciones.

![sections](/img/flareon2015-c1/sections.png)

![imports](/img/flareon2015-c1/imports.png)

Carguemos este binario en radare2 para su análisis estático:

![Load and Functions](/img/flareon2015-c1/r2-afl.png)

Como podemos ver, el binario no cuenta con funciones después o antes del entrypoint, por lo que analicemos que sucede después del entry.

```
[0x00401000]> pdf
            ;-- section..text:
            ;-- eip:
┌ (fcn) entry0 149
│   entry0 ();
│           ; var int32_t var_10h @ ebp-0x10
│           ; var HANDLE var_ch @ ebp-0xc
│           ; var HANDLE hFile @ ebp-0x8
│           ; var LPDWORD lpNumberOfBytesWritten @ ebp-0x4
│           0x00401000      55             push ebp                    ; [00] -rwx section size 4096 named .text
│           0x00401001      89e5           mov ebp, esp
│           0x00401003      83ec10         sub esp, 0x10
│           0x00401006      8945f0         mov dword [var_10h], eax
│           0x00401009      6af6           push 0xfffffffffffffff6     ; DWORD nStdHandle
│           0x0040100b      ff1558204000   call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
│           0x00401011      8945f4         mov dword [var_ch], eax
│           0x00401014      6af5           push 0xfffffffffffffff5     ; DWORD nStdHandle
│           0x00401016      ff1558204000   call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
│           0x0040101c      8945f8         mov dword [hFile], eax
│           0x0040101f      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
│           0x00401021      8d45fc         lea eax, [lpNumberOfBytesWritten]
│           0x00401024      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
│           0x00401025      6a2a           push 0x2a                   ; '*' ; 42 ; DWORD nNumberOfBytesToWrite
│           0x00401027      68f2204000     push str.Let_s_start_out_easy____Enter_the_password ; 0x4020f2 ; "Let's start out easy\r\nEnter the password>" ; LPCVOID lpBuffer
│           0x0040102c      ff75f8         push dword [hFile]          ; HANDLE hFile
│           0x0040102f      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
│           0x00401035      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
│           0x00401037      8d45fc         lea eax, [lpNumberOfBytesWritten]
│           0x0040103a      50             push eax                    ; LPDWORD lpNumberOfBytesRead
│           0x0040103b      6a32           push 0x32                   ; '2' ; 50 ; DWORD nNumberOfBytesToRead
│           0x0040103d      6858214000     push 0x402158               ; 'X!@' ; LPVOID lpBuffer
│           0x00401042      ff75f4         push dword [var_ch]         ; HANDLE hFile
│           0x00401045      ff1568204000   call dword [sym.imp.kernel32.dll_ReadFile] ; 0x402068 ; BOOL ReadFile(HANDLE hFile, LPVOID lpBuffer, DWORD nNumberOfBytesToRead, LPDWORD lpNumberOfBytesRead, LPOVERLAPPED lpOverlapped)
│           0x0040104b      31c9           xor ecx, ecx
│           ; CODE XREF from entry0 (0x401061)
│       ┌─> 0x0040104d      8a8158214000   mov al, byte [ecx + 0x402158] ; [0x402158:1]=0
│       ╎   0x00401053      347d           xor al, 0x7d
│       ╎   0x00401055      3a8140214000   cmp al, byte [ecx + 0x402140] ; [0x402140:1]=31
│      ┌──< 0x0040105b      751e           jne 0x40107b
│      │╎   0x0040105d      41             inc ecx
│      │╎   0x0040105e      83f918         cmp ecx, 0x18               ; 24
│      │└─< 0x00401061      7cea           jl 0x40104d
│      │    0x00401063      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
│      │    0x00401065      8d45fc         lea eax, [lpNumberOfBytesWritten]
│      │    0x00401068      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
│      │    0x00401069      6a12           push 0x12                   ; 18 ; DWORD nNumberOfBytesToWrite
│      │    0x0040106b      681c214000     push str.You_are_success    ; 0x40211c ; "You are success\r\n" ; LPCVOID lpBuffer
│      │    0x00401070      ff75f8         push dword [hFile]          ; HANDLE hFile
│      │    0x00401073      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
│      │┌─< 0x00401079      eb16           jmp 0x401091
│      ││   ; CODE XREF from entry0 (0x40105b)
│      └──> 0x0040107b      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
│       │   0x0040107d      8d45fc         lea eax, [lpNumberOfBytesWritten]
│       │   0x00401080      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
│       │   0x00401081      6a12           push 0x12                   ; 18 ; DWORD nNumberOfBytesToWrite
│       │   0x00401083      682e214000     push str.You_are_failure    ; 0x40212e ; "You are failure\r\n" ; LPCVOID lpBuffer
│       │   0x00401088      ff75f8         push dword [hFile]          ; HANDLE hFile
│       │   0x0040108b      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
│       │   ; CODE XREF from entry0 (0x401079)
│       └─> 0x00401091      89ec           mov esp, ebp
│           0x00401093      5d             pop ebp
└           0x00401094      c3             ret
```

Podemos ver que radare2 identifico acertadamente algunas funciones con sus respectivos parámetros, así como el análisis de sus argumentos.

Analicemos lo que sucede después del stack frame para esta función:

```
0x00401006      8945f0         mov dword [var_10h], eax
0x00401009      6af6           push 0xfffffffffffffff6     ; DWORD nStdHandle
0x0040100b      ff1558204000   call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
```

Guardamos el valor original de EAX en una variable y subimos **-10** decimal con signo al stack. Llamamos a GetStdHandle con este argumento, lo cual significa que obtendremos un handle para el STDIN via CONIN$. [Véase mas aquí](https://docs.microsoft.com/en-us/windows/console/getstdhandle)

```
│           0x00401011      8945f4         mov dword [var_ch], eax
│           0x00401014      6af5           push 0xfffffffffffffff5     ; DWORD nStdHandle
│           0x00401016      ff1558204000   call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
```

El valor retornado de GetStdHandle es guardado en var_ch, mientras que ahora en lugar del valor -10, le pasaremos **-11**, lo cual significa el STDOUT o CONOUT$. De esta manera el programa ya tiene las direcciones de buffer (handles) para poder interactuar en entrada y salida con nuestra consola.

```
0x0040101c      8945f8         mov dword [hFile], eax
│           0x0040101f      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
│           0x00401021      8d45fc         lea eax, [lpNumberOfBytesWritten]
│           0x00401024      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
│           0x00401025      6a2a           push 0x2a                   ; '*' ; 42 ; DWORD nNumberOfBytesToWrite
│           0x00401027      68f2204000     push str.Let_s_start_out_easy____Enter_the_password ; 0x4020f2 ; "Let's start out easy\r\nEnter the password>" ; LPCVOID lpBuffer
│           0x0040102c      ff75f8         push dword [hFile]          ; HANDLE hFile
0x0040102f      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
```

Tras la llamada anterior, el resultado (handle de STDOUT) es guardado en hFile.

Ahora llamamos a `BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)`, lo que es equivalente a escribir en el STDOUT, el string "Let's start out easy\r\nEnter the password>" que incluye 42 caracteres. Los siguientes dos parámetros son interesantes cuando manejamos una sincronización en el handle, pero dado que se trata del STDOUT, podemos ignorarlos.

```
0x00401035      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
0x00401037      8d45fc         lea eax, [lpNumberOfBytesWritten]
0x0040103a      50             push eax                    ; LPDWORD lpNumberOfBytesRead
0x0040103b      6a32           push 0x32                   ; '2' ; 50 ; DWORD nNumberOfBytesToRead
0x0040103d      6858214000     push 0x402158               ; 'X!@' ; LPVOID lpBuffer
0x00401042      ff75f4         push dword [var_ch]         ; HANDLE hFile
0x00401045      ff1568204000   call dword [sym.imp.kernel32.dll_ReadFile] ; 0x402068 ; BOOL ReadFile(HANDLE hFile, LPVOID lpBuffer, DWORD nNumberOfBytesToRead, LPDWORD lpNumberOfBytesRead, LPOVERLAPPED lpOverlapped)
```

Después de escribir en pantalla, la siguiente función ReadFile, tomara el texto que ingresemos después del cursor hasta que encuentre un carácter de nueva linea. Importante mencionar que lo que ingresemos, se guardará en **0x402158** y el tamaño máximo de caracteres a guardar es de 50. Nuevamente las ultimas dos opciones corresponden a archivos que necesiten sincronía, que no es nuestro caso.

```
0x0040104b      31c9           xor ecx, ecx
0x0040104d      8a8158214000   mov al, byte [ecx + 0x402158] ; [0x402158:1]=0
0x00401053      347d           xor al, 0x7d
0x00401055      3a8140214000   cmp al, byte [ecx + 0x402140] ; [0x402140:1]=31
0x0040105b      751e           jne 0x40107b
0x0040105d      41             inc ecx
0x0040105e      83f918         cmp ecx, 0x18               ; 24
0x00401061      7cea           jl 0x40104d
```

Limpiamos ECX y iniciamos un loop. En este loop estaremos tomando cada carácter que escribimos anteriormente y le aplicaremos la operación XOR contra **0x7d**, posteriormente compararemos el resultado con lo que se encuentre en la ubicación 0x402140 mas el index de la letra respectiva.

Si por algún motivo, en alguna iteración las letras no son iguales (entiéndase letra como el carácter en turno ^ 0x7d), entonces el loop saldrá hacia la dirección 0x40107b. Caso de que sean iguales, incrementara el contador ECX y validara si este ya llego a 24 decimal.

Esto significa que la longitud a comparar es de 24 caracteres y la del buffer es de 50, y que cada letra puede ser reversible si aplicamos un xor 0x7d al buffer de comparación. Hagamos este paso después de terminar el análisis estático.

```
0x00401063      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
0x00401065      8d45fc         lea eax, [lpNumberOfBytesWritten]
0x00401068      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
0x00401069      6a12           push 0x12                   ; 18 ; DWORD nNumberOfBytesToWrite
0x0040106b      681c214000     push str.You_are_success    ; 0x40211c ; "You are success\r\n" ; LPCVOID lpBuffer
0x00401070      ff75f8         push dword [hFile]          ; HANDLE hFile
0x00401073      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
0x00401079      eb16           jmp 0x401091
```

Si todas las letras anteriores coincidieron, entonces se escribirá en el STDOUT el mensaje "You are success\r\n".

```
0x0040107b      6a00           push 0                      ; LPOVERLAPPED lpOverlapped
0x0040107d      8d45fc         lea eax, [lpNumberOfBytesWritten]
0x00401080      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
0x00401081      6a12           push 0x12                   ; 18 ; DWORD nNumberOfBytesToWrite
0x00401083      682e214000     push str.You_are_failure    ; 0x40212e ; "You are failure\r\n" ; LPCVOID lpBuffer
0x00401088      ff75f8         push dword [hFile]          ; HANDLE hFile
0x0040108b      ff1564204000   call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
```

Caso de que alguna letra no coincida, se realizaba el jump a `0x0040107b` y se imprime en el STDOUT el mensaje "You are failure\r\n".

Ahora que hemos analizado correctamente las instrucciones, radare2 nos permite aplicar operaciones sin realmente modificar el binario, por lo que veamos primero que hay en `0x402140`, después habilitemos el modo de cache y finalmente modifiquemos esta área del buffer con un `XOR 0x7d` para 24 caracteres:

![buffer1](/img/flareon2015-c1/buffer1.png)

La flag se revela ante nosotros:

**bunny_sl0pe@flare-on.com**

Validemos rápidamente vía WINE:

![solved](/img/flareon2015-c1/solved.png)

Flag validada.

---

Espero que les haya gustado.

Saludos,
