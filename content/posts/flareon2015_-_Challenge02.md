---
title: "Flare-on 2015 - Challenge 02"
date: 2019-07-02T00:05:30-05:00
tags: ["flareon","revisitado","flareon2015","reversing","writeup"]
categories: ["reversing","ctf"]
---

Continuamos con los challenges del 2015, en esta ocasión el reto 2 sube un poco más la dificultad. En este caso tenemos un programa que al inicio parece idéntico al anterior, solo que este tiene su propia función de cifrado que implementa un mecanismo bastante interesante de protección. Nos encontraremos con una protección que aunque continua comparando contra un bloque fijo de bytes, el proceso previo requiere información que se genera en cada iteración.

<!--more-->

En el reto anterior, descargamos y descomprimimos un archivo que nos creo unas carpetas. Dentro de la carpeta `2` podremos encontrar el archivo del challenge 2 llamado `599EA8F84AD975CFB07E0E5732C9BA14.zip`, el cual descomprimimos y nos devuelve un archivo unicamente (`very_success`). Recordemos que la contraseña para todos los retos del 2015 es **flare**.

> Nota: Todos los archivos de challenge se encuentran dentro de un zip que se descarga [aquí](http://www.flare-on.com/files/2015_FLAREOn_Challenges.zip)

![unzip](/img/flareon2015-c2/unzip.png)

Si analizamos el archivo `very_success` con el comando `file` y `pescan` tenemos lo siguiente:

![File and PEscan](/img/flareon2015-c2/file-pescan.png)

Esto significa que tenemos poca entropía (no empaquetado probablemente) y 2 secciones; `.text` que se puede automoficiar y `.data`.

Continuamos el análisis en radare. Verificamos que importa y cuantos entrypoints tiene:

![Load on Radare2](/img/flareon2015-c2/r2-load.png)

Si recordamos el reto anterior, también unicamente hacia uso de _kernel32.dll_ y algunas de sus funciones. Entre ellas podemos ver algunas conocidas como GetStdHandle, WriteFile y ReadFile, por lo que omitiré las explicaciones de estas funciones. Si tienes duda, por favor consulta el reto anterior.

Veamos con cuantas y cuales funciones cuenta este programa:

![Print Disassembly of Function entry0](/img/flareon2015-c2/r2-afl.png)

Así mismo, realice un _Print Disassembly of Function_ (pdf) sobre el entry0, por lo que vemos que al mero inicio en esta sección, ya estamos llamando a la función `section..text` que renombrare como `main`.

![Analyze Function Name to main](/img/flareon2015-c2/r2-afn.png)

## Parte 1. Main

Nos cambiamos al inicio de la función main y realizamos un `pdf` (lo pego también como texto por el tamaño de la función main):

![Print Disassembly of Function main](/img/flareon2015-c2/r2-pdf-main.png)

```
[0x004010df]> s main
[0x00401000]> pdf
;-- section..text:
/ (fcn) main 132
|   int main (int argc, char **argv, char **envp);
|           ; var int32_t var_10h @ ebp-0x10
|           ; var HANDLE var_ch @ ebp-0xc
|           ; var HANDLE hFile @ ebp-0x8
|           ; var LPDWORD lpNumberOfBytesWritten @ ebp-0x4
|           0x00401000      pop  eax ; [00] -rwx section size 4096 named .text
|           0x00401001      push ebp
|           0x00401002      mov  ebp, esp
|           0x00401004      sub  esp, 0x10
|           0x00401007      mov  dword [var_10h], eax
|           0x0040100a      push 0xfffffffffffffff6 ; DWORD nStdHandle
|           0x0040100c      call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
|           0x00401012      mov  dword [var_ch], eax
|           0x00401015      push 0xfffffffffffffff5 ; DWORD nStdHandle
|           0x00401017      call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
|           0x0040101d      mov  dword [hFile], eax
|           0x00401020      push 0 ; LPOVERLAPPED lpOverlapped
|           0x00401022      lea  eax, [lpNumberOfBytesWritten]
|           0x00401025      push eax ; LPDWORD lpNumberOfBytesWritten
|           0x00401026      push 0x43 ; 'C' ; 67 ; DWORD nNumberOfBytesToWrite
|           0x00401028      push str.You_crushed_that_last_one__Let_s_up_the_game.____Enter_the_password ; 0x4020f2 ; "You crushed that last one! Let's up the game.\r\nEnter the password>" ; LPCVOID lpBuffer
|           0x0040102d      push dword [hFile] ; HANDLE hFile
|           0x00401030      call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
|           0x00401036      push 0 ; LPOVERLAPPED lpOverlapped
|           0x00401038      lea  eax, [lpNumberOfBytesWritten]
|           0x0040103b      push eax ; LPDWORD lpNumberOfBytesRead
|           0x0040103c      push 0x32 ; '2' ; 50 ; DWORD nNumberOfBytesToRead
|           0x0040103e      push 0x402159 ; 'Y!@' ; LPVOID lpBuffer
|           0x00401043      push dword [var_ch] ; HANDLE hFile
|           0x00401046      call dword [sym.imp.kernel32.dll_ReadFile] ; 0x402068 ; BOOL ReadFile(HANDLE hFile, LPVOID lpBuffer, DWORD nNumberOfBytesToRead, LPDWORD lpNumberOfBytesRead, LPOVERLAPPED lpOverlapped)
|           0x0040104c      push 0
|           0x0040104e      lea  eax, [lpNumberOfBytesWritten]
|           0x00401051      push eax
|           0x00401052      push 0x11 ; 17
|           0x00401054      push dword [lpNumberOfBytesWritten]
|           0x00401057      push 0x402159 ; 'Y!@'
|           0x0040105c      push dword [var_10h]
|           0x0040105f      call fcn.00401084
|           0x00401064      add  esp, 0xc
|           0x00401067      test eax, eax
|       ,=< 0x00401069      je   0x401072
|       |   0x0040106b      push str.You_are_success ; 0x402135 ; "You are success\r\n"
|      ,==< 0x00401070      jmp  0x401077
|      |`-> 0x00401072      push str.You_are_failure ; 0x402147 ; "You are failure\r\n"
|      `--> 0x00401077      push dword [hFile] ; HANDLE hFile
|           0x0040107a      call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
|           0x00401080      mov  esp, ebp
|           0x00401082      pop  ebp
\           0x00401083      ret
[0x00401000]>
```

Analicemos esto por partes:

1. Stack Frame y GetStdHandle

```
0x00401000      pop  eax ; [00] -rwx section size 4096 named .text
0x00401001      push ebp
0x00401002      mov  ebp, esp
0x00401004      sub  esp, 0x10
0x00401007      mov  dword [var_10h], eax
0x0040100a      push 0xfffffffffffffff6 ; DWORD nStdHandle
0x0040100c      call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
0x00401012      mov  dword [var_ch], eax
0x00401015      push 0xfffffffffffffff5 ; DWORD nStdHandle
0x00401017      call dword [sym.imp.kernel32.dll_GetStdHandle] ; 0x402058 ; HANDLE GetStdHandle(DWORD nStdHandle)
0x0040101d      mov  dword [hFile], eax
```

La primera parte es bastante sencilla, tenemos el stack frame (0x10) y tenemos los apuntadores o handles para interactuar con el STDIN y STDOUT. Hablaremos del 'POP EAX' y el 'MOV EBP+0x10, EAX' más adelante.

2. Mensaje de bienvenida

```
0x00401020      push 0 ; LPOVERLAPPED lpOverlapped
0x00401022      lea  eax, [lpNumberOfBytesWritten]
0x00401025      push eax ; LPDWORD lpNumberOfBytesWritten
0x00401026      push 0x43 ; 'C' ; 67 ; DWORD nNumberOfBytesToWrite
0x00401028      push str.You_crushed_that_last_one__Let_s_up_the_game.____Enter_the_password ; 0x4020f2 ; "You crushed that last one! Let's up the game.\r\nEnter the password>" ; LPCVOID lpBuffer
0x0040102d      push dword [hFile] ; HANDLE hFile
0x00401030      call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
```

En esta parte llamamos a WriteFile para que muestre el mensaje de bienvenida y nos muestre en pantalla que requiere una contraseña.

3. Guardamos la contraseña

```
0x00401036      push 0 ; LPOVERLAPPED lpOverlapped
0x00401038      lea  eax, [lpNumberOfBytesWritten]
0x0040103b      push eax ; LPDWORD lpNumberOfBytesRead
0x0040103c      push 0x32 ; '2' ; 50 ; DWORD nNumberOfBytesToRead
0x0040103e      push 0x402159 ; 'Y!@' ; LPVOID lpBuffer
0x00401043      push dword [var_ch] ; HANDLE hFile
0x00401046      call dword [sym.imp.kernel32.dll_ReadFile] ; 0x402068 ; BOOL ReadFile(HANDLE hFile, LPVOID lpBuffer, DWORD nNumberOfBytesToRead, LPDWORD lpNumberOfBytesRead, LPOVERLAPPED lpOverlapped)
```

El texto ingresado con limite a 50 caracteres, es guardado en **0x402159**.

4. The Call

```
0x0040104c      push 0
0x0040104e      lea  eax, [lpNumberOfBytesWritten]
0x00401051      push eax
0x00401052      push 0x11 ; 17
0x00401054      push dword [lpNumberOfBytesWritten]
0x00401057      push 0x402159 ; 'Y!@'
0x0040105c      push dword [var_10h]
0x0040105f      call fcn.00401084
```

Ahora llamamos a la función 00401084 de la siguiente manera:

`fcn.00401084(EBP-0x10, 0x402159, lpNumberOfBytesWritten, 0x11, EBP-0x4, 0)`

Antes de entrar en esta función, veamos que mas hace main.

5. success or failure

```
      0x00401064      add  esp, 0xc
      0x00401067      test eax, eax
  ,=< 0x00401069      je   0x401072
  |   0x0040106b      push str.You_are_success ; 0x402135 ; "You are success\r\n"
 ,==< 0x00401070      jmp  0x401077
 |`-> 0x00401072      push str.You_are_failure ; 0x402147 ; "You are failure\r\n"
 `--> 0x00401077      push dword [hFile] ; HANDLE hFile
      0x0040107a      call dword [sym.imp.kernel32.dll_WriteFile] ; 0x402064 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
      0x00401080      mov  esp, ebp
      0x00401082      pop  ebp
      0x00401083      ret
```

Empezamos por un ajuste al ESP de 12, esto significa que probablemente la función anterior modifico el mismo stack. La siguiente instrucción evalúa EAX (probablemente el resultado de la función anterior), y realiza un jump si EAX es igual a 0, y sino, continua por el siguiente camino que hace un jump.

Como podemos ver por los strings identificados en la data, tenemos éxito y fracaso dependiendo del valor de EAX, mas importante aun, tenemos un push de este string conforme a lo que sea necesitado. Después subimos el valor de hFile y llamamos a WriteFile, función que conocemos por recibir una serie de parámetros conocidos.

Si aplicamos el movimiento al stack pointer de 12 (add esp, 0xc), estaríamos ajustando de manera interesante el stack, en donde ya no consideraríamos: dword [lpNumberOfBytesWritten], 0x402159, dword [var_10h]. En su lugar escribiríamos STR, hFile, quedando la pila de la siguiente manera:

| Ubicación | Contenido     |
|-----------|---------------|
| ESP +16   | 0             |
| ESP +12   | \[EBP-0x4\]   |
| ESP +8    | 0x11          |
| ESP +4    | STR_WinOrLose |
| ESP       | hFile         |

Este patro lo conocemos pues son los argumentos que recibe la función WriteFile, por lo que básicamente escribiríamos 'Éxito' o 'Fracaso' dependiendo del resultado de EAX para seleccionar el string correcto.

Ya después del 5, tenemos una serie de pasos que seguimos hasta llegar al ret, donde ajustamos el stack.

## Parte 2. Valida Password

Ahora que hemos terminado con `main`, analicemos `fcn.00401084`, que desde ahora renombraremos como `validaPassword`:

![Analyze Function Rename validaPassword](/img/flareon2015-c2/r2-afl-validapassword.png)

Veamos el contenido de esta función:

![PDF validaPassword](/img/flareon2015-c2/r2-pdf-validapassword.png)

![VV validaPassword](/img/flareon2015-c2/r2-VV-validapassword.png)

```
┌ (fcn) validaPassword 91
│   validaPassword (LPDWORD arg_10h);
│           ; arg LPDWORD arg_10h @ ebp+0x10
│           ; CALL XREF from main (0x40105f)
│           0x00401084      55             push ebp
│           0x00401085      89e5           mov ebp, esp
│           0x00401087      83ec00         sub esp, 0
│           0x0040108a      57             push edi
│           0x0040108b      56             push esi
│           0x0040108c      31db           xor ebx, ebx
│           0x0040108e      b925000000     mov ecx, 0x25
│           0x00401093      394d10         cmp dword [arg_10h], ecx    ; [0x10:4]=-1 ; 16
│       ┌─< 0x00401096      7c3f           jl 0x4010d7
│       │   0x00401098      8b750c         mov esi, dword [ebp + 0xc]  ; [0xc:4]=-1 ; 12
│       │   0x0040109b      8b7d08         mov edi, dword [ebp + 8]    ; [0x8:4]=-1 ; 8
│       │   0x0040109e      8d7c0fff       lea edi, [edi + ecx - 1]
│       │   ; CODE XREF from validaPassword (0x4010d3)
│      ┌──> 0x004010a2      6689da         mov dx, bx
│      ╎│   0x004010a5      6683e203       and dx, 3
│      ╎│   0x004010a9      66b8c701       mov ax, 0x1c7               ; 455
│      ╎│   0x004010ad      50             push eax
│      ╎│   0x004010ae      9e             sahf
│      ╎│   0x004010af      ac             lodsb al, byte [esi]
│      ╎│   0x004010b0      9c             pushfd
│      ╎│   0x004010b1      32442404       xor al, byte [esp + 4]
│      ╎│   0x004010b5      86ca           xchg dl, cl
│      ╎│   0x004010b7      d2c4           rol ah, cl
│      ╎│   0x004010b9      9d             popfd
│      ╎│   0x004010ba      10e0           adc al, ah
│      ╎│   0x004010bc      86ca           xchg dl, cl
│      ╎│   0x004010be      31d2           xor edx, edx
│      ╎│   0x004010c0      25ff000000     and eax, 0xff
│      ╎│   0x004010c5      6601c3         add bx, ax
│      ╎│   0x004010c8      ae             scasb al, byte es:[edi]
│      ╎│   0x004010c9      660f45ca       cmovne cx, dx
│      ╎│   0x004010cd      58             pop eax
│     ┌───< 0x004010ce      e307           jecxz 0x4010d7
│     │╎│   0x004010d0      83ef02         sub edi, 2
│     │└──< 0x004010d3      e2cd           loop 0x4010a2
│     │┌──< 0x004010d5      eb02           jmp 0x4010d9
│     │││   ; CODE XREFS from validaPassword (0x401096, 0x4010ce)
│     └─└─> 0x004010d7      31c0           xor eax, eax
│      │    ; CODE XREF from validaPassword (0x4010d5)
│      └──> 0x004010d9      5e             pop esi
│           0x004010da      5f             pop edi
│           0x004010db      89ec           mov esp, ebp
│           0x004010dd      5d             pop ebp
└           0x004010de      c3             ret
```

Como en la función anterior, partiremos esta función en partes para entender lo que sucede:

1. Stack Frame + Análisis inicial

```
┌ (fcn) validaPassword 91
│   validaPassword (LPDWORD arg_10h);
│           ; arg LPDWORD arg_10h @ ebp+0x10
│           ; CALL XREF from main (0x40105f)
│           0x00401084      55             push ebp
│           0x00401085      89e5           mov ebp, esp
│           0x00401087      83ec00         sub esp, 0
```

Tan pronto como iniciamos en la función establecemos un stack frame, eso es lo normal y esperado. Pero esta función realiza algo interesante y es que al momento de mover el ESP (0x00401087), lo deja en el mismo lugar, lo cual implica que no tiene variables locales.

2. Inicializador y contador

```
│           0x0040108a      57             push edi
│           0x0040108b      56             push esi
│           0x0040108c      31db           xor ebx, ebx
│           0x0040108e      b925000000     mov ecx, 0x25
│           0x00401093      394d10         cmp dword [arg_10h], ecx    ; [0x10:4]=-1 ; 16
│       ┌─< 0x00401096      7c3f           jl 0x4010d7
```

Salvamos EDI y ESI (cuales valores sean al iniciar el programa), inicializamos EBX=0 y cargamos 0x25 en ECX. Le sigue un CMP que evalúa si lo encontrado en EBP+0x10 es igual a 0x25, recordemos que CMP no es mas que una resta sin guardar el resultado, solo sirve para activar las flags. La siguiente instrucción compara el resultado de CMP, evaluado si el valor de arg_10h (EBP+0x10) es menor a 0x25.

¿Pero que es EBP+0x10? Pues se trata de algo declarado ubicaciones hacia atrás en el stack, porque en este momento EBP=ESP.

En el stack si nos vamos hacia atrás tenemos:

| Ubicación | Contenido                        |
|-----------|----------------------------------|
| EBP +16   | dword \[lpNumberOfBytesWritten\] |
| EBP +12   | 0x402159                         |
| EBP +8    | dword \[var\_10h\]               |
| EBP +4    | 0x00401064                       |
| EBP       | EBP main                         |

Entonces el valor de EBP+0x10 es la cantidad de Bytes ingresados en el WriteFile anterior. Esto significa que básicamente estaríamos evaluando si ingresamos mas de 0x25 (37) caracteres para evadir el jump.

3. Load input

```
│       ┌─< 0x00401096      7c3f           jl 0x4010d7
│       │   0x00401098      8b750c         mov esi, dword [ebp + 0xc]  ; [0xc:4]=-1 ; 12
│       │   0x0040109b      8b7d08         mov edi, dword [ebp + 8]    ; [0x8:4]=-1 ; 8
│       │   0x0040109e      8d7c0fff       lea edi, [edi + ecx - 1]
```

En el siguiente bloque, tras un false en el jump, hacemos referencia a ebp+12 para cargar el contenido de esa dirección en ESI, equivalente a 0x402159. Luego cargamos el contenido en EBP+8, equivalente a VAR_10h de main o lo que es igual a EBP-0x10 de main.

Este valor es bastante curioso, si nos ponemos a revisar el código al inicio de main, la primera instrucción es un 'POP EAX' y la consecutiva donde se usa EAX, 'MOV EBP-0x10, EAX', por lo que primero guardamos el valor inmediato del stack (ESP) y en la siguiente instrucción guardamos en esa dirección del stack (EBP-0x10) el valor de EAX, la siguiente dirección que se ejecutaría después de llamar a main, es decir, 0x004010e4. Si recordamos, en el entry0 teníamos directamente el call a main, y después unas instrucciones poco habituales, pero esto como veremos mas adelante, no se trata de instrucciones, sino de data.

La ultima instrucción, carga en EDI la dirección de ECX (0x25 al inicio) menos uno, es decir, 0x24 mas la dirección 0x004010e4, resultando en 0x00401108.

4. XOR input password

```
│       │   ; CODE XREF from validaPassword (0x4010d3)
│      ┌──> 0x004010a2      6689da         mov dx, bx
│      ╎│   0x004010a5      6683e203       and dx, 3
│      ╎│   0x004010a9      66b8c701       mov ax, 0x1c7               ; 455
│      ╎│   0x004010ad      50             push eax
│      ╎│   0x004010ae      9e             sahf
│      ╎│   0x004010af      ac             lodsb al, byte [esi]
│      ╎│   0x004010b0      9c             pushfd
│      ╎│   0x004010b1      32442404       xor al, byte [esp + 4]
```

Las primeras dos instrucciones, cargan el contenido de BX y lo filtran a 2 bits, es decir, los valores posibles de DX serian 0,1,2,3.

La 3ra instrucción, carga 0x01c7 en AX y la 4ta lo sube al stack como 0x000001c7.

La 5ta instrucción, 'sahf', carga AH en los registros EFLAGS, por lo que al subir 0x01 solo el registro CF se ve encendido.

La 6ta instrucción carga el contenido del tamaño de un byte de la dirección en ESI en AL (veremos mas adelante que se trata de la letra en turno).

La 7ma instrucción carga el contenido de EFLAGS en el stack.

La 8va instrucción ejecuta un XOR de ESP+4, es decir, de 0xc7 porque acabamos de subir el contenido de EFLAGS, y del valor o la primera letra ingresada en password que guardamos en 0x402159.

5. Pseudorandom Shift

```
│      ╎│   0x004010b5      86ca           xchg dl, cl
│      ╎│   0x004010b7      d2c4           rol ah, cl
│      ╎│   0x004010b9      9d             popfd
│      ╎│   0x004010ba      10e0           adc al, ah
│      ╎│   0x004010bc      86ca           xchg dl, cl
```

1ra instrucción: intercambiamos el valor en DL con CL, de manera que ahora CL vale BL&3 y DL vale el numero de bytes ingresados.
2da instrucción: Realiza un shift a la izquierda de AH (0x01) la cantidad ubicada en CL (BL&3). CL puede ir de 0 a 3 en sus valores.

La siguiente instrucción (3ra) baja el ultimo valor en el stack (los registros EFLAGS) hacia EFLAGS, regresando consigo lo ultimo que se llego a modificar vía 0x01 (CF=1).

La 4ta instrucción, suma AL + AH + CF y lo almacena en AL. Como sabemos, AL tiene el valor de la letra actual, AH tiene un shift a la izquierda denotado por BL&3 y el CF es igual a 1.

La 5ta instrucción intercambia de nuevo DL y CL, por lo que ahora CL vale el contador de numero de bytes ingresados (mayor igual a 37) y DL el valor de BL&3, ya que no se modificaron estos valores y solo trabajamos sobre EAX.

6. Compara contra data local

```
│      ╎│   0x004010be      31d2           xor edx, edx
│      ╎│   0x004010c0      25ff000000     and eax, 0xff
│      ╎│   0x004010c5      6601c3         add bx, ax
│      ╎│   0x004010c8      ae             scasb al, byte es:[edi]
│      ╎│   0x004010c9      660f45ca       cmovne cx, dx
│      ╎│   0x004010cd      58             pop eax
```

1ra instrucción, limpia EDX que antes valía BL&3, ahora vale 0.
2da instrucción, filtra EAX via AND y nada mas deja a EAX con el valor de AL.
3ra instrucción, suma BX (antes valia 0) y AX (letra actual).

4ta instrucción, compara si el 1er byte de EDI (dirección 0x004010e4 + lpNumberOfBytesWritten -1) es igual a AL (después del XOR, Shift y suma con AH+1). Si son iguales, incrementa EDI (siguiente letra) y decrementa ECX (siguiente valor a comprar).

5ta instrucción, si la instrucción anterior modifico a ZF=0, mueve DX (0x0) en CX.

6ta instrucción, guarda el ultimo elemento ingresado en el stack en EAX, en este caso 0x01c7.

7. Win or Lose

```
│     ┌───< 0x004010ce      e307           jecxz 0x4010d7
│     │╎│   0x004010d0      83ef02         sub edi, 2
│     │└──< 0x004010d3      e2cd           loop 0x4010a2
│     │┌──< 0x004010d5      eb02           jmp 0x4010d9
│     │││   ; CODE XREFS from validaPassword (0x401096, 0x4010ce)
│     └─└─> 0x004010d7      31c0           xor eax, eax
│      │    ; CODE XREF from validaPassword (0x4010d5)
│      └──> 0x004010d9      5e             pop esi
│           0x004010da      5f             pop edi
│           0x004010db      89ec           mov esp, ebp
│           0x004010dd      5d             pop ebp
└           0x004010de      c3             ret
```

En el ultimo bloque, 1ra instrucción, brincamos si CX es igual a 0 a la instrucción que nos haria igual a 0 EAX (mensaje de FAIL en main), pero si ECX es diferente de 0, restamos 2 a la direccion en EDI y ejecutamos un loop a 0x4010a2, regresando al inicio.

Este loop se rompe cuando ECX = 1, porque loop va decrementando en -1 a ECX. Cuando ECX se hace 0 por loop, se brinca a la salida de la función sin modificar EAX, lo que nos da el win en main.

## Solver, decoder

Ahora que hemos analizado por completo la función validaPassword, tenemos en mano una idea mas detallada de como cada valor ingresado es evaluado y como podemos llegar hasta el mensaje de WIN.

Lo primero es extraer el contenido el buffer "oculto":

```
[0x00401000]> px 37 @ 0x004010e4
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x004010e4  afaa adeb aeaa eca4 baaf aeaa 8ac0 a7b0  ................
0x004010f4  bc9a baa5 a5ba afb8 9db8 f9ae 9dab b4bc  ................
0x00401104  b6b3 909a a8                             .....
[0x00401000]> pcp 37 @ 0x004010e4
import struct
buf = struct.pack ("37B", *[
0xaf,0xaa,0xad,0xeb,0xae,0xaa,0xec,0xa4,0xba,0xaf,0xae,
0xaa,0x8a,0xc0,0xa7,0xb0,0xbc,0x9a,0xba,0xa5,0xa5,0xba,
0xaf,0xb8,0x9d,0xb8,0xf9,0xae,0x9d,0xab,0xb4,0xbc,0xb6,
0xb3,0x90,0x9a,0xa8])
```

> Por conveniencia, usaremos python3 para resolver este reto.

Este buffer invierte el orden, por lo que empezamos desde el ultimo hasta el primero (`buf[::-1]`).

Si seguimos el orden inverso de las instrucciones, lo primero seria la parte de ADC, en donde guardamos en AL el valor de AL+AH+1, ya que EDI es comparado contra AL. Así que:

> EDI = AL_preADC + AH + 1
> EDI -AH -1 = AL_preADC

Pero AL_preADC se encuentra protegida por el XOR de 0xc7, por lo que si:

> AL_preADC = AL_original ^ 0xc7,

Entonces:

> (EDI - AH - 1) ^ 0xc7 = AL_original

Ahora debemos despejar AH para poder realizar la ecuación `xD`.

> AH = shiftl(AH_preShift, CL)

Si recordamos antes de eso, AH_preShift era 0x01, por lo que:

> AH = shiftl(1, CL)

Entonces, CL se había cambiado con DL, y DL valía BL&3.

BL es una variable muy interesante, porque en al inicio de la primera iteración BX=0, pero al final de la primera iteración BX=BX+AX. Esto significa que con cada iteración de letra, BX sera la suma de todas las anteriores a esa.

A este valor suma, le vamos a aplicar un AND contra 3 para unicamente tomar los primeros 2 bits, por lo que el shift puede ser hasta 3 ubicaciones.

El valor de AH entonces, puede ser de 1,2,4,8. Dicho de otra manera: 0x0001, 0x0010, 0x0100, 0x1000.

Una representación de lo anterior en python3 seria lo siguiente:

```python
import struct
buf = struct.pack ("37B", *[
0xaf,0xaa,0xad,0xeb,0xae,0xaa,0xec,0xa4,0xba,0xaf,0xae,
0xaa,0x8a,0xc0,0xa7,0xb0,0xbc,0x9a,0xba,0xa5,0xa5,0xba,
0xaf,0xb8,0x9d,0xb8,0xf9,0xae,0x9d,0xab,0xb4,0xbc,0xb6,
0xb3,0x90,0x9a,0xa8])
buf = buf[::-1]

flag=''
shift = 0
l_acomulada = 0


for d in range(len(buf)):
    l = (buf[d] - ((1 << shift) & 0xff) - 1) ^ 0xc7
    flag += chr(l)

    l_acomulada += buf[d]
    shift = l_acomulada & 3

print(flag)
```

Tras ejecutar el código, la flag aparecerá ante nosotros:

```
xbytemx@laptop:~/flare-on2015/2$ python3 decode.py
a_Little_b1t_harder_plez@flare-on.com
```

Validamos ingresando esta flag en el binario:

![flag](/img/flareon2015-c2/flag.png)

Con esto, hemos resuelto el segundo reto del Flare-On 2015.

**flag: a_Little_b1t_harder_plez@flare-on.com**

---

Espero que te haya gustado.

Saludos,
