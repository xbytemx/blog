---
title: "Flare-on 2016 - Challenge 01"
date: 2019-11-02T15:10:46-06:00
description: "Las aventuras del primer reto del flare-on del 2016."
tags: ["flareon2016","reversing","writeup"]
categories: ["ctf","reversing"]
draft: true

---

El primer reto del flare-on del 2016 me dejo una enseñanza interesante acerca de base64 y su funcionamiento.
<!--more-->

# Información

Comenzamos por descargar y descomprimir el reto, el cual nos devuelve el binario `challenge1.exe`.

> Recuerden que todos los retos del 2016 los podemos descargar del siguiente enlace: [Flare-On3_Challenges.zip](http://flare-on.com/files/Flare-On3_Challenges.zip). La contraseña para descomprimirlo es **flare**.

Realizando un `file`, detect it easy (`DIE`) y `pescan`, observamos que su comportamiento es normal y que no presenta relativamente nada empaquetado.

![file and diec](/img/flareon2016-c1/file.png)

![pescan](/img/flareon2016-c1/pescan.png)

# Entrypoint

Iniciamos un proyecto en `ghidra`, cargamos el binario aceptando la opción de autodetección y ejecutamos CodeBrowser sobre el binario aplicando las opciones de análisis por defecto. Vemos que `main` no fue detectado automáticamente, pero analicemos desde entrypoint para encontrar a `main`:

![Entry](/img/flareon2016-c1/entry0.png)

Podemos ver que se inicializa el canario y que se realiza un `jmp` fuera de este bloque. Sigamos a ese bloque:

![Entry after jump](/img/flareon2016-c1/entry1.png)

Aquí podemos ver que el enlazador ha empezado con su proceso antes de llamar a `main`. Esto nos da un indicio de que `main` debe estar cerca el los `exit` de la función, por lo que nos adelantamos hasta encontrar los primeros `exit` en la función:

![Entry before exit](/img/flareon2016-c1/entry2.png)

Como podemos ver en la imagen, el primer `exit` recibe la salida de la función marcada como `FUN_00401420`, que si prestamos atención a dicha función, observaremos que recibe por el stack el contenido de EAX, EDI, ESI. Este debería ser nuestro `main` y tras buscar en Google el significado de `_Code`, comprendí que aquí es cuando el enlazador de Microsoft hace la referencia a `main` en msvc.

# Función main

Veamos que tenemos al inicio de esta función:

![main](/img/flareon2016-c1/main1.png)

Utilizando la herramienta de decompile, se genera el siguiente pseudocódigo:

```c
int main(int argc,char **argv,char **envp)

{
  int iVar1;
  undefined local_98 [128];
  char *local_18;
  char *local_14;
  HANDLE local_10;
  HANDLE local_c;
  DWORD local_8;

  local_c = GetStdHandle(0xfffffff5);
  local_10 = GetStdHandle(0xfffffff6);
  local_14 = "x2dtJEOmyjacxDemx2eczT5cVS9fVUGvWTuZWjuexjRqy24rV29q";
  WriteFile(local_c,"Enter password:\r\n",0x12,&local_8,(LPOVERLAPPED)0x0);
  ReadFile(local_10,local_98,0x80,&local_8,(LPOVERLAPPED)0x0);
  local_18 = (char *)FUN_00401260((int)local_98,local_8 - 2);
  iVar1 = _strcmp(local_18,local_14);
  if (iVar1 == 0) {
    WriteFile(local_c,"Correct!\r\n",0xb,&local_8,(LPOVERLAPPED)0x0);
  }
  else {
    WriteFile(local_c,"Wrong password\r\n",0x11,&local_8,(LPOVERLAPPED)0x0);
  }
  return 0;
}
```

Lo primero que observamos, a rasgos generales, es que ingresaremos una contraseña, esta será convertida (local_18) y finalmente comparada con el texto de local_14. Como la aproximación nos da una idea general del funcionamiento, empecemos a renombrar variables:

| # | local    | renombrada a |
|---|----------|--------------|
| 1 | local_8  | BytesWritten |
| 2 | local_c  | hOutput      |
| 3 | local_10 | hInput       |
| 4 | local_14 | secret       |
| 5 | local_18 | key          |
| 6 | local_98 | keybuffer    |
| 7 | iVar1    | resultado    |

Luego entonces:

![Main after rename variables](/img/flareon2016-c1/main2.png)

Para llegar a "Correct", resultado tiene que ser 0. La manera en que resultado es igual a 0, es que strcmp tenga dos strings idénticos, por lo que ahora debemos reversear la función `FUN_00401260`.

# Función convert-text
