---
title: "Flare-on 2014 - Challenge 03: Shellolololol"
date: 2018-10-25T19:21:39-05:00
draft: false
tags: ["flareon","revisitado","flareon2014","radare2","reversing","writeup"]
categories: ["reversing","ctf"]
description: "Ahora el turno del challenge 3 del FLARE-ON 2014, este se trata de un PE32 que descarga una shellcode maliciosa, veamos que tenemos por aqui"
---

Bien comenzamos por descargar este reto, "C3.zip" (recordar que se los retos se encuentran [aquí][c2c76e1c]). En este challenge lo descomprimos con la contraseña "malware", por lo que iniciamos descomprimiendo el archivo en nuestra carpeta challenge03:

![Unzip](/img/flareon2014-c3/unzip.png)

Identificamos el ejecutable con DetectItEasy:

![Detect It Easy](/img/flareon2014-c3/die.png)

Como podemos observar, se trata de un ejecutable tipo PE32, que no se encuentra empaquetado y podemos ver que tambien importa la dll "msvcrt.dll", veamos mas de esto en CFF Explorer:

![CFF Explorer](/img/flareon2014-c3/cffexplorer-import.png)

El DLL msvcrt.dll, es La biblioteca estandar de C para Visual C++, la cual nos permite manipular strings, posiciones de memoria, llamadas a la entrada y salida estilo C, entre otras cosas.

Bien, continuamos cargando el archivo en radare2 para comenzar a analizarlo estaticamente:

![Radare2 - Functions, Imports](/img/flareon2014-c3/r2-functions.png)

Observamos las funciones importadas y 4 funciones:

    0x00401000    1 1022         fcn.00401000
    0x0040258c    1 4            fcn.0040258c
    0x00402590    1 10           fcn.00402590
    0x004025d1    1 58           fcn.004025d1

wow, 1022 instrucciones en la primera funcion, espero que tambien sepa hacer el cafe ...

Analicemos desde el entry0:

![Entry0-1](/img/flareon2014-c3/entry0.png)

![Entry0-2](/img/flareon2014-c3/entry0_1.png)

Se define el stack frame con 0x2c y posteriormente se carga la variable local_18h en el stack y se llama a fcn.004025d1

![Entry0, Funcion 1](/img/flareon2014-c3/e0_f1.png)

Veamos que hace esta funcion:

```
0x004025d1      55             push ebp
0x004025d2      8b6c2408       mov ebp, dword [arg_8h]     ; Subimos el balor de arg_8h en ebp
0x004025d6      8d44240c       lea eax, dword [arg_ch]     ; Cargamos la direccion de arg_ch en EAX
0x004025da      894500         mov dword [ebp], eax        ; Cargamos la direccion de arg_ch en EBP
0x004025dd      31c0           xor eax, eax                ; EAX = 0
0x004025df      894504         mov dword [arg_4h], eax     ; arg_4h = 0
0x004025e2      64a100000000   mov eax, dword fs:[0]       ; EAX = 0
0x004025e8      894508         mov dword [arg_8h_2], eax   ; arg_8h_2 = 0
0x004025eb      b8cc254000     mov eax, 0x4025cc           ; EAX = 0x4025cc
0x004025f0      89450c         mov dword [arg_ch_2], eax   ; arg_ch_2 = 0x4025cc
0x004025f3      b8c0254000     mov eax, 0x4025c0           ; EAX = 0x4025c0
0x004025f8      894510         mov dword [arg_10h], eax    ; arg_10h = 0x4025c0
0x004025fb      31c0           xor eax, eax                ; EAX = 0
0x004025fd      894514         mov dword [arg_14h], eax    ; arg_14h = 0
0x00402600      8d4508         lea eax, dword [arg_8h_2]   ; EAX = direccion de arg_8h_2
0x00402603      64a300000000   mov dword fs:[0], eax       ;
0x00402609      5d             pop ebp
0x0040260a      c3             ret
```

Ok, despues en ENTRY0 vemos que los calls que hacemos a las funciones de "msvcrt.dll", llamamos a nuestra mega funcion, fcn.00401000:

![Funcion loader, inicio](/img/flareon2014-c3/loader1.png)

Y se sigue asi hasta el final:

![Funcion loader, final](/img/flareon2014-c3/loader2.png)

Como vemos, lo unico que se hace es subir 8bits via EAX al stack y no lo coloca en direcciones aleatorias sino continuas. Vamos a nombrarla StackLoader para ser mas amigables:

![Rename to StackLoader](/img/flareon2014-c3/stackloader.png)

Ok para combinar todo lo que se carga en el stack, ejecutemos r2pipe para concatenar y generar lo que carga el loader:

{{< highlight python >}}
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import r2pipe, binascii

r = r2pipe.open('such_evil')
r.cmd("e anal.bb.maxsize=2048000") # Se pone el block size a 2M
r.cmd("aa")
fcn_master = r.cmdj("aflj")
fcn_payload = fcn_master[0]
asm_payload = r.cmdj("pDj %u @ %s" % (fcn_payload['realsz'], fcn_payload['offset']))

with open("payload.bin", "wb") as fout:
    for idx in range(4,1030,2):
        fout.write(chr(asm_payload[idx]["val"]))

r.quit()

{{< /highlight >}}

Ahora que tenemos el siguiente archivo "payload.bin", miremos cual es su contenido en radare2:

![XOR1](/img/flareon2014-c3/xorcode2.png)

Veamos como se ve en el modo por grafo:

![XOR2](/img/flareon2014-c3/xorcode1.png)

Ok, parece que todas las instrucciones siguen a un loop en donde se realizar un XOR del byte correspondiente con 0x66 de los 479 bytes siguientes, algo en pseudocodigo como lo siguiente:

```
index = 28;
newcode = stack;
for (Contador =  479; Contardor != 0; Contador--){
  newcode[index] = stack[index] ^ 0x66;
  index++;
}
```

Eso quiere decir que el jmp 0x31 (marcado en verde), no era un jmp...

![Previo a Loop2](/img/flareon2014-c3/preloop2.png)

Ok, entonces modificamos el codigo que usamos hace un momento para hacer el xor, descargar el payload nuevamente y analizarlo:

```python
#!/usr/bin/env python
# -*- encode: utf-8 -*-

import r2pipe, binascii

r = r2pipe.open('such_evil')
r.cmd("e anal.bb.maxsize=2048000") # Se pone el block size a 2M
r.cmd("aa")
fcn_master = r.cmdj("aflj")
fcn_payload = fcn_master[0]
asm_payload = r.cmdj("pDj %u @ %s" % (fcn_payload['realsz'], fcn_payload['offset']))

real_payload = b''
for idx in range(4,1030,2):
    real_payload += chr(asm_payload[idx]["val"])

xor_payload = b''
for idx in range(479):
    xor_payload += chr(ord(real_payload[idx+33]) ^ 0x66)

with open("payload2.bin", "wb") as fout:
    for idx in range(len(xor_payload)):
        fout.write(xor_payload[idx])

r.quit()
```

Tras ejecutar el codigo, analizamos el resultado en radare2:

![Post Loop2, XOR](/img/flareon2014-c3/postxor1.png)

Nuevamente parece que algo no esta del todo bien en el codigo, vemos los strings del binario:

![Strings of XOR1](/img/flareon2014-c3/xor1_izz.png)

El primer string, parece darnos algun tipo de mensaje:

    Num Paddr      Vaddr       Len Size Section  Type  String
    000 0x00000000 0x00000000  19  20   ()       ascii and so it beginshus

Pareciera que la cola del "hus" esta de mas, por lo que convirtamos eso que aparece como codigo, como data:

![Loop3, code](/img/flareon2014-c3/prexor2.png)

Ok, este codigo se parece un poco al anterior, vemos que ahora en lugar de ser solo un 0x66, es un string.

![Loop3, graph](/img/flareon2014-c3/loop3.png)

Analicemos el codigo:

```
0x00000010      6875730000     push 0x7375                 ; 'us'
0x00000015      6873617572     push 0x72756173             ; 'saur'
0x0000001a      686e6f7061     push 0x61706f6e             ; 'nopa'
                                                           ; En total: 'nopasaurus'
0x0000001f      89e3           mov ebx, esp                ; Salvamos el ESP en EBX (EBX = ESP)
0x00000021      e800000000     call 0x26                   ; brincamos unos ceros
0x00000026      8b3424         mov esi, dword [esp]        ; Salvamos ESP en ESI (ESI = ESP)
0x00000029      83c62d         add esi, 0x2d               ; ESI ahora vale ESP+45 (offset)
0x0000002c      89f1           mov ecx, esi                ; ECX = ESP + 45
0x0000002e      81c18c010000   add ecx, 0x18c              ; ECX = ESP + (45 + 396 = 441)
0x00000034      89d8           mov eax, ebx                ; EAX = ESP
0x00000036      83c00a         add eax, 0xa                ; EAX = ESP + 10 (longitud del string)
0x00000039      39d8           cmp eax, ebx                ; Compara EAX con EBX, para ver si reinicia el string
0x0000003b      7505           jne 0x42                    ; Sino ha acabado con el string, brinca 0x42
0x0000003d      89e3           mov ebx, esp                ; Si ya acabo el string, EBX = ESP
0x0000003f      83c304         add ebx, 4                  ; EBX = ESP + 4
0x00000042      39ce           cmp esi, ecx                ; Compara si ya acabo los 441 bytes
0x00000044      7408           je 0x4e                     ; Si acabo, se va a 0x4e, osea sale
0x00000046      8a13           mov dl, byte [ebx]          ; Si no acabo, carga en dl un byte de EBX
0x00000048      3016           xor byte [esi], dl          ; Hace XOR de DL y ESI, guardando en ESI
0x0000004a      43             inc ebx                     ; Incremento EBX (Debe alcanzar a EAX)
0x0000004b      46             inc esi                     ; Siguiente byte!
0x0000004c      ebeb           jmp 0x39                    ; De regreso al inicio del loop.
```

Nuevamente, modificamos el script que teniamos anterior para realizar el loop de xor multibyte:

```python
#!/usr/bin/env python
# -*- encode: utf-8 -*-

import r2pipe, binascii

r = r2pipe.open('such_evil')
r.cmd("e anal.bb.maxsize=2048000") # Se pone el block size a 2M
r.cmd("aa")
fcn_master = r.cmdj("aflj")
fcn_payload = fcn_master[0]
asm_payload = r.cmdj("pDj %u @ %s" % (fcn_payload['realsz'], fcn_payload['offset']))

real_payload = b''
for idx in range(4,1030,2):
    real_payload += chr(asm_payload[idx]["val"])

xor_payload = b''
for idx in range(479):
    xor_payload += chr(ord(real_payload[idx+33]) ^ 0x66)

xor2_payload = b''
secret = 'nopasaurus'
for idx in range(396):
    xor2_payload += chr(ord(xor_payload[idx + 45 + 19 + 19]) ^ ord(secret[idx%len(secret)])) # 45 de offset, 19 de texto y 19 de codigo

with open("payload3.bin", "wb") as fout:
    for idx in range(len(xor2_payload)):
        fout.write(xor2_payload[idx])

r.quit()
```

Con el codigo anexo generamos otro binario, el cual analizamos en radare2 nuevamente:

![Loop4, code](/img/flareon2014-c3/loop4-code.png)

Buscamos por strings:

![Payload4, strings](/img/flareon2014-c3/loop4-strings.png)

Ajustamos nuevamente el codigo:

![Loop4, offset](/img/flareon2014-c3/loop4-offset.png)

![Loop4, graph](/img/flareon2014-c3/loop4-graph.png)

Ooootro loop. Veamos las instrucciones a detalle:

```
0x00000031      e800000000     call 0x36                    ; Brincamos a 0x36
0x00000036      8b3424         mov esi, dword [esp]         ; ESI = ESP
0x00000039      83c61e         add esi, 0x1e                ; ESI = ESP + 30
0x0000003c      b938010000     mov ecx, 0x138               ; ECX = 312
0x00000041      83f900         cmp ecx, 0                   ; if (ECX == 0)
0x00000044      7e0e           jle 0x54                     ; Igual o menor, sal de este loop
0x00000046      8136624f6c47   xor dword [esi], 0x476c4f62  ; XOR de ESI contra 'GlOb'
0x0000004c      83c604         add esi, 4                   ; ESI = ESI + 4
0x0000004f      83e904         sub ecx, 4                   ; ECX = ECX - 4
0x00000052      ebed           jmp 0x41                     ; Regresa a inicio
```

Aplicamos el mismo caso del ejemplo anterior en el codigo:

```python
#!/usr/bin/env python
# -*- encode: utf-8 -*-

import r2pipe, binascii

r = r2pipe.open('such_evil')
r.cmd("e anal.bb.maxsize=2048000") # Se pone el block size a 2M
r.cmd("aa")
fcn_master = r.cmdj("aflj")
fcn_payload = fcn_master[0]
asm_payload = r.cmdj("pDj %u @ %s" % (fcn_payload['realsz'], fcn_payload['offset']))

real_payload = b''
for idx in range(4,1030,2):
    real_payload += chr(asm_payload[idx]["val"])

xor_payload = b''
for idx in range(479):
    xor_payload += chr(ord(real_payload[idx+33]) ^ 0x66)

xor2_payload = b''
secret = 'nopasaurus'
for idx in range(396):
    xor2_payload += chr(ord(xor_payload[idx + 45 + 19 + 19]) ^ ord(secret[idx%len(secret)]))

xor3_payload = b''
secret2 = 'bOlG:'
for idx in range(312):
    xor3_payload += chr(ord(xor2_payload[idx + 84]) ^ ord(secret2[idx%len(secret2)]))

with open("payload4.bin", "wb") as fout:
    for idx in range(len(xor3_payload)):
        fout.write(xor3_payload[idx])

r.quit()
```

Ok, iniciamos por analizar payload4.bin:

![Loop5, Init](/img/flareon2014-c3/loop5-init.png)

Como podemos observar, hay algo parecido a un texto en las primeras lineas, tal como los anteriores...

Veamos mejor las instrucciones:

![Loop5, Code1](/img/flareon2014-c3/loop5-code1.png)

![Loop5, Code2](/img/flareon2014-c3/loop5-code2.png)

Nuevamente otro loop, solo que esta vez la llave es mas grande. Apliquemos una adecuacion al codigo que ya tenemos:

```python
#!/usr/bin/env python
# -*- encode: utf-8 -*-

import r2pipe, binascii

r = r2pipe.open('such_evil')
r.cmd("e anal.bb.maxsize=2048000") # Se pone el block size a 2M
r.cmd("aa")
fcn_master = r.cmdj("aflj")
fcn_payload = fcn_master[0]
asm_payload = r.cmdj("pDj %u @ %s" % (fcn_payload['realsz'], fcn_payload['offset']))

real_payload = b''
for idx in range(4,1030,2):
    real_payload += chr(asm_payload[idx]["val"])
xor_payload = b''
for idx in range(479):
    xor_payload += chr(ord(real_payload[idx+33]) ^ 0x66)

xor2_payload = b''
secret = 'nopasaurus'
for idx in range(396):
    xor2_payload += chr(ord(xor_payload[idx+45+19+19]) ^ ord(secret[idx%len(secret)]))

xor3_payload = b''
secret2 = 'bOlG'
for idx in range(312):
    xor3_payload += chr(ord(xor2_payload[idx +84]) ^ ord(secret2[idx%len(secret2)]))

xor4_payload = b''
secret3 = 'omg is it almost over?!?'
for idx in range(81):
    xor4_payload += chr(ord(xor3_payload[idx +93+5]) ^ ord(secret3[idx%len(secret3)]))

with open("payload5.bin", "wb") as fout:
    for idx in range(len(xor4_payload)):
        fout.write(xor4_payload[idx])

r.quit()
```

Tras generar payload5.bin, lo analizamos con r2:

![Loop6, Init](/img/flareon2014-c3/loop6-init.png)

Finalmente, como pudimos ver en la ventana anterior, la bandera ya se asoma, solo basta ajustar code->data con _Cd 29_, podemos observar la bandera:

![Final](/img/flareon2014-c3/final.png)


### **flag: such.5h311010101@flare-on.com**

---

Espero que les haya gustado.

Saludos,


[c2c76e1c]: http://www.flare-on.com/ "Flare-On 2014"
