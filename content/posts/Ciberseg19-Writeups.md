---
title: "Ciberseg19 - Writeups"
date: 2019-01-24T12:54:52-06:00
draft: false
tags: ["web","crypto","forensics","exploiting","reversing"]
categories: ["ctf"]
---

Como previo a las Jornadas de Seguridad y Ciberdefensa 2019 de la Universidad de Alcalá, se realizó un Capture The Flag (CTF) abierto al público pero con el enfoque principal a estudiantes universitarios de la UAH. La duración del CTF fue del viernes 18 a las 10:00hrs hasta el miércoles 23 a las 14:00hrs, UTC+1.

Los retos estuvieron divididos en 5 categorías con una área de puntuación adicional para los que reportaran problemas en la plataforma.

Los retos que llegue a resolver son los siguientes:

<!--more-->

# Crypto

## Complutum message (15pts)

PbzcyhgvHeovfHavirefvgnf

---

### Solution

Ejecutando Caesar Cipher:

``` text
xbytemx@laptop:~/ctf-uah/crypto$ echo "PbzcyhgvHeovfHavirefvgnf" | tr '[A-Za-z]' '[N-ZA-Mn-za-m]'
ComplutiUrbisUniversitas
```

### Flag

flag{ComplutiUrbisUniversitas}

## Alien message from XXXI century (50pts)

Hemos recibido un mensaje alienígena!! ¿Nos puedes ayudar a entenderlo?


sha256 - alienMessage.png: 852e54e06fc4eed951ede96998fef12169c0e94cbfe6eecf20eb621eb1068748

---

### Solution

Para resolver este reto, usaremos https://images.google.com, en donde buscaremos la imagen y el nombre del archivo "alien message".

![Google image search](/img/ciberseg19/writeup2-google-images.png)

Como podemos ver, tenemos una coincidencia en la esquina derecha inferior, por lo que seguimos el link hasta la pagina fuente de la imagen:

> http://www.computableminds.com/post/Futurama/criptography/code/encrypt/decrypt/

Como podemos observar, esta imagen es resultado de la critografia usada en mensajes durante la serie de futurama (guiño con el nombre del reto), por lo que usamos simplemente la siguiente imagen de conversion para descifrar la contraseña.

> http://www.computableminds.com/img/futurama/futurama-lenguaje-1.gif

``` text
______ _______ ___

planet express uah

```

### Flag

flag{planetexpressuah}

## Clásica (75pts)

En la criptografía, como en la cerveza. Siempre viene bien alguna clásica:

sa okhbx dpejaja gc uad ei wlk tau becufwielhtkfaf effi uw sa okhbx pvmxauan ne zlm sgygjigxtbv oszvfaa fo chdohs jw btkvimdce xrhvn fo cbudls p ubyc loghmpe jw llcloedce eillscxaypfrxv hhsq k drqpqmesysg waurv gprkpcc on zlm rsmwjigxtbv oszvfaa a drrv ei gfdvr fyrngp ewgwjtq lrvomerkw f cworcr nshvjhdq nefwbge ggy sw caors wyrnl y deea hrymcairky ea epge jm hrqwa rv mmkvjv ahbugdes gff aopys sopvecwz dg vúphop psj hyipmicdmiw zfnrgnirquiw jgu lgfaqxse plhblq kghd z qeclh sqwof pvc gcszieys rq ushf hhrc va phsziqs f pcba yd dvmglvgtkfvd qsv vkv hgwof gfgmuako ootru fwxv tvnkdo ilhirvjl lc plnj fw lrurapnbrhsw ulw zop nof fpwej ibe uo lyhwer dmf bkon crs iwf vllgqaplpr siyhnkjaod fp zzsqe c va sdcvmts ke nk cruwidr

---

### Solution

Para resolver este reto, primero identificamos que se trata de vigeneré cipher, por lo que usando la siguiente herramienta online, podemos hacer un ataque de fuerza bruta por analisis estadistico. Es importante mencionar que al saber que el CTF esta hecho en Español, probablemente el texto cifrado se encuentre en español. Esto tambien se puede determinar con la frecuencia de algunas palabras. Sin mas apliquemos el brute forcer:

https://www.guballa.de/vigenere-solver

![vinegere solver](/img/ciberseg19/vigenere-solver.png)

Texto final:

la mahou clasica es una de sus mas representativas bebe de la mahou original de mil ochocientos noventa de cuando se utilizaba tapon de corcho y cuya botella se elaboraba artesanalmente paso a denominarse mahou clasica en mil novecientos noventa y tres de color dorado aspecto brillante y cuerpo moderado destaca por su sabor suave y buen equilibrio en boca su aroma es ligero afrutado con tonos florales de lúpulo los principales ingredientes son levadura lupulo agua y malta somos muy clasicos en todo para la cerveza y para la criptografia por eso hemos decidido meter este bonito vigenere la flag es hackandbeers que son dos cosas que se llevan muy bien por eso delegacion organizaba el viaje a la fabrica de la cerveza

key: hackandbeers


### Flag

flag{hackandbeers}

## Cryptography is not steganography (150pts)

¡Que quede claro! ¡Ocultar no es cifrar!

Video: https://drive.google.com/open?id=13eUUK7Lk9XDgSwKPArxX54cxIMSSglOd

sha256-harder.mp4: a472ad60a557b3d7bc2c33f74f8ba17c85faf0022e306150f61d41111fb83a1c

---

### Solution

Después de prestar atención al video harder.mp4, notaremos varias cosas:
- El audio continua después de que el video corta
- El video mp4, contiene un stream de audio y otro de video

Volviendo a observar el video veremos algo interesante, en la esquina superior izquierda vemos algo que aparece como luces de un faro.

Genere el siguiente programa en python para extraer el valor del pixel durante todo el video:

``` python
#!/usr/bin/env python

import cv2, re
from PIL import Image

vc = cv2.VideoCapture("harder.mp4")

while(True):
    success, image = vc.read()
    if not success: break
    print image[0, 0]

vc.release()
cv2.destroyAllWindows()
```

La salida del programe es:

``` text
xbytemx@laptop:~/ctf-uah/crypto$ python stegnotcrypt.py
[242 251 252]
[ 3 12 13]
[ 3 12 13]
[242 251 252]
[242 251 252]
[ 3 12 13]
[ 3 12 13]
[242 251 252]
[242 251 252]
[ 3 12 13]
[ 3 12 13]
[242 251 252]
[ 1 10 11]
[ 1 10 11]
[241 250 251]
[242 251 252]
[240 251 252]
[ 0  9 10]
[ 0  9 10]
[239 250 251]
[239 250 251]
[239 250 251]
[239 250 251]
[ 0 10 11]
[240 251 252]
[ 0 10 11]
[ 0 10 11]
[240 251 252]
[241 250 251]
[1 8 9]
[1 8 9]
[1 8 9]
[236 243 244]
[0 5 6]
[0 5 6]
[0 5 6]
[0 8 9]
[234 244 243]
[0 8 9]
[0 8 9]
[241 249 253]
[0 8 9]
[0 8 9]
[242 250 254]
[242 250 254]
[0 8 9]
[242 250 254]
[243 249 253]
[243 249 253]
[0 8 9]
[0 8 9]
[243 249 253]
[243 249 253]
[243 249 253]
[243 249 253]
[0 4 8]
[243 249 253]
[0 1 5]
[0 1 5]
[243 249 253]
[0 0 5]
[0 0 5]
[0 0 5]
[244 247 252]
[244 247 252]
[0 0 5]
[0 0 5]
[245 248 253]
[244 247 252]
[245 248 253]
[0 2 6]
[0 2 6]
[242 248 252]
[0 0 4]
[0 0 4]
[243 249 253]
[0 0 4]
[243 249 253]
[243 249 253]
[0 0 4]
[243 249 253]
[0 0 4]
[0 0 4]
[243 249 253]
[0 0 6]
[0 0 6]
[0 0 6]
[242 249 255]
[242 249 255]
[0 0 6]
[0 0 6]
[242 249 255]
[242 249 255]
[0 1 7]
[0 1 7]
[0 1 7]
[234 241 247]
[0 2 8]
[235 243 247]
[0 0 5]
[0 0 5]
[0 0 5]
[0 0 5]
[0 0 5]
[234 245 246]
[0 0 5]
[0 0 5]
[0 0 5]
[239 250 251]
[239 250 251]
[241 250 251]
[241 250 251]
[239 250 251]
[0 1 5]
[0 1 5]
[241 250 251]
[0 1 5]
[239 250 251]
[239 250 251]
[0 1 5]
[234 243 244]
[0 2 6]
[0 2 6]
[0 2 6]
[0 2 6]
[238 248 252]
[238 248 252]
[238 248 252]
[239 247 251]
[0 0 4]
[0 0 4]
[239 247 251]
[238 248 252]
[0 4 8]
[238 248 252]
[0 3 7]
[238 248 252]
[0 2 6]
[0 2 6]
[240 248 252]
[0 2 6]
[0 2 6]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[241 249 253]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
[0 0 0]
```

Se me hizo muy interesante que los valores fueran solo dos, altos y bajos, como binario (inclusive los beep de los pixeles), por lo que fije 200 como punto de corte y tome la primera columna de colores como margen, aunque pudo haber sido cualquiera como vimos en la imagen. El código ajustado quedó de la siguiente manera:

``` python
#!/usr/bin/env python

import cv2, re
from PIL import Image

vc = cv2.VideoCapture("harder.mp4")

bm =  ""

while(True):
    success, image = vc.read()
    if not success: break
    #print image[0, 0]

    if(image[0, 0][0] < 200):
        bm += '1'
    else:
        bm += '0'


print [bm[i:i+8] for i in range(0, len(bm), 8)]
vc.release()
cv2.destroyAllWindows()
```

Y la salida de este programa es la siguiente:

``` text
xbytemx@laptop:~/ctf-uah/crypto$ python stegnotcrypt.py
['01100110', '01101100', '01100001', '01100111', '01111011', '01100100', '01100001', '01101110', '01100011', '01101001', '01101110', '01100111', '01011111', '01110000', '01101001', '01111000', '01100101', '01101100', '00000000', '01111111', '11111111', '11111111', '1111']
```

Ahora convirtamos estos strings con base binaria a ASCII, sustituyendo el print por un for de la siguiente manera:

``` python
flag = ""
for l in [bm[i:i+8] for i in range(0, len(bm), 8)]:
    flag += chr(int(l, 2))

print flag
```

Después de ejecutar el código veremos la flag del reto:

``` text
xbytemx@laptop:~/ctf-uah/crypto$ python stegnotcrypt.py
flag{dancing_pixel
```

El código completo es el siguiente:

``` python
#!/usr/bin/env python

import cv2, re
from PIL import Image

vc = cv2.VideoCapture("harder.mp4")

bm =  ""

while(True):
    success, image = vc.read()
    if not success: break
    #print image[0, 0]

    if(image[0, 0][0] < 200):
        bm += '1'
    else:
        bm += '0'

flag = ""
for l in [bm[i:i+8] for i in range(0, len(bm), 8)]:
    flag += chr(int(l, 2))

print flag

vc.release()
cv2.destroyAllWindows()

```

Desafortunadamente ya no llegue a ingresar esta flag en el sistema, pero me divertí un rato con opencv.

### Flag

flag{dancing\_pixel}

## To be XOR not to be (200pts)

Han censurado la emisión de video, pero a ver si conseguimos el nombre del personaje que aperece en él.

Link: https://drive.google.com/open?id=1MWcJtoBXbDTiD-CB8LAxSGGAUPhiXFkQ


sha256.result.mp4: 75d835ac1d030ac1639d37df508cf666c44b1fbfec7c889fed888c19ddc795e9

---

### Solution

Comenzamos por reproducir el video:

``` text
xbytemx@laptop:~/ctf-uah/crypto$ mpv result.mp4
Playing: result.mp4
 (+) Video --vid=1 (*) (h264 480x194 25.000fps)
File tags:
 Artist: Shakespeare
 Album:
 Comment:
 Date: 0
 Genre:
 Title: ToBxor!toB
 Track: 0
VO: [gpu] 480x194 yuv444p
V: 00:00:01 / 00:02:01 (1%)


Exiting... (Quit)
```

Como podemos observar, parece que hay algún tipo de ruido o interferencia que nos impide ver claramente que sucede en el video.

Después de intentar varias cosas, llego al fin un hit caído desde twitter. En el nos entregaban un fragmento con el nombre de pista.png

Tome ese PNG y después de jugar un poco con [forensically](https://29a.ch/photo-forensics/#error-level-analysis), logre ajustar el error y la escala del mismo hasta lograr observar un mensaje dentro de la imagen:

![result with ela](/img/ciberseg19/result-with-ela.png)

Un poco de google nos ayudara a llegar a la obra [El mercader de Venecia](https://es.wikiquote.org/wiki/El_mercader_de_Venecia) donde el dialogo pertenece al personaje Shylock.

No quedándome en esta parte, y ajustando el código de [ewencp](https://gist.githubusercontent.com/ewencp/3356622/raw/674e24a63924954e5e0ab758a9479c8914d03528/ela.py), implemente este siguiente código en python para recuperar el video completo lo mejor posible:

``` python
#!/usr/bin/env python

from contextlib import closing
from videosequence import VideoSequence
from PIL import Image, ImageChops, ImageEnhance
from images2gif import writeGif

with closing(VideoSequence("result.mp4")) as frames:
    newframes = []
    for idx,frame in enumerate(frames):
        resaved = "result/result_{:04d}_low.jpg".format(idx)
        ela = "result/result_{:04d}_ela.jpg".format(idx)

        frame.save(resaved, 'JPEG', quality=5)

        resaved_im = Image.open(resaved)
        ela_im = ImageChops.difference(frame, resaved_im)
        extrema = ela_im.getextrema()
        scale = 10

        ela_im = ImageEnhance.Brightness(ela_im).enhance(scale)
        ela_im.save(ela)

        newframes.append(ela_im.copy())
        frame.close()
        ela_im.close()

    writeGif("result-with-ela.gif",newframes)

```

### Flag

flag{shylock}

# Forensics

## Exfiltration files (25pts)

Hemos interceptado esta foto parece que oculta algún mensaje.

sha256: bf9f66b6d341570b4d2247453a20f6e13f1f135c3432a8d474560da6c7d3e7bf

---

### Solution

Ejecutamos un strings sobre el archivo y la flag aparecerá como layer:

``` text
xbytemx@laptop:~/ctf-uah/forense$ strings imagen.jpg | grep flag
" id="W5M0MpCehiHzreSzNTczkc9d"?> <x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="Adobe XMP Core 5.5-c021 79.155772, 2014/01/13-19:44:00        "> <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"> <rdf:Description rdf:about="" xmlns:xmp="http://ns.adobe.com/xap/1.0/" xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/" xmlns:stEvt="http://ns.adobe.com/xap/1.0/sType/ResourceEvent#" xmlns:photoshop="http://ns.adobe.com/photoshop/1.0/" xmlns:dc="http://purl.org/dc/elements/1.1/" xmp:CreatorTool="Adobe Photoshop CC 2014 (Windows)" xmp:CreateDate="2019-01-17T13:03:34+01:00" xmp:MetadataDate="2019-01-17T13:03:34+01:00" xmp:ModifyDate="2019-01-17T13:03:34+01:00" xmpMM:InstanceID="xmp.iid:8e9e2553-4ecd-7e42-9548-16647b7dda9e" xmpMM:DocumentID="adobe:docid:photoshop:b305dd2d-1a4f-11e9-897e-ce2ac433bf61" xmpMM:OriginalDocumentID="xmp.did:22aa09e6-3955-454e-9039-9b9d8e04376c" photoshop:ColorMode="3" photoshop:ICCProfile="Adobe RGB (1998)" dc:format="image/jpeg"> <xmpMM:History> <rdf:Seq> <rdf:li stEvt:action="created" stEvt:instanceID="xmp.iid:22aa09e6-3955-454e-9039-9b9d8e04376c" stEvt:when="2019-01-17T13:03:34+01:00" stEvt:softwareAgent="Adobe Photoshop CC 2014 (Windows)"/> <rdf:li stEvt:action="saved" stEvt:instanceID="xmp.iid:8e9e2553-4ecd-7e42-9548-16647b7dda9e" stEvt:when="2019-01-17T13:03:34+01:00" stEvt:softwareAgent="Adobe Photoshop CC 2014 (Windows)" stEvt:changed="/"/> </rdf:Seq> </xmpMM:History> <photoshop:TextLayers> <rdf:Bag> <rdf:li photoshop:LayerName="flag{RapidoYFurioso}" photoshop:LayerText="flag{RapidoYFurioso}"/> </rdf:Bag> </photoshop:TextLayers> </rdf:Description> </rdf:RDF> </x:xmpmeta>

```

### Flag

flag{RapidoYFurioso}


## Break the gesture (50pts)

sha256-gestures.key: 6e34680b9240ab502ffc51375da535586803b52e1ee72a0257e2b0e1ae1eac39

### Solution

Después de googlear, gesture.key y break, tenemos un resultado acerca del Pattern Lock de los celulares con SO Android. Usando apcl rompemos el patrón de bloqueo:

``` text
xbytemx@laptop:~/ctf-uah/forense$ python ~/git/androidpatternlock/aplc.py gesture.key

################################
# Android Pattern Lock Cracker #
#             v0.2             #
# ---------------------------- #
#  Written by Chema Garcia     #
#     http://safetybits.net    #
#     chema@safetybits.net     #
#          @sch3m4             #
################################

[i] Taken from: http://forensics.spreitzenbarth.de/2012/02/28/cracking-the-pattern-lock-on-android/

[:D] The pattern has been FOUND!!! => 210345876

[+] Gesture:

  -----  -----  -----
  | 3 |  | 2 |  | 1 |
  -----  -----  -----
  -----  -----  -----
  | 4 |  | 5 |  | 6 |
  -----  -----  -----
  -----  -----  -----
  | 9 |  | 8 |  | 7 |
  -----  -----  -----

It took: 2.6252 seconds

```

### Flag

flag{210345876}

## Pcap (250pts)

Creo que se han llevado algo. ¿Me puedes decir el nombre del archivo?

URL PCAP: https://drive.google.com/open?id=1VC1TYYb01fJPbjMBhKmsB9VgQtQ3-xXM

SHA256 captura.pcap: ae4b4ef361dbbeb7ab7409df4eaca935856d5cda6976b59d9882cbddea508a36

---

### Solution

Primero buscamos rápidamente en ICMP y UDP vía wireshark por comandos extraños. Al no encontrar nada inusual, buscamos por frame contains la palabra "flag", pero tampoco tuvimos éxito. Continué buscando métodos POST para subir archivos y me encontré con uno:

``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura.pcap -Y 'http.request.method == "POST"'
35391 144.991672 192.168.1.135 → 198.54.116.21 HTTP 2118 POST /download.php HTTP/1.1  (application/x-www-form-urlencoded)
```

Si desplegamos el contenido de este paquete veremos que se sube un texto encodeado en base64:


``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura.pcap -Y 'http.request.method == "POST"' -T fields -e urlencoded-form.key
MzcgN0EgQkMgQUYgMjcgMUMgMDAgMDQgNjUgM0QgNzcgOEMgNTggMEYgMDAgMDAgMDAgMDAgMDAgMDAgNkEgMDAgMDAgMDAgMDAgMDAgMDAgMDAgOEQgNDYgRkYgQjkgRTAgNEIgQjggMEYgNTAgNUQgMDAgMjYgMTIgNDcgMTEgQTggQzQgMDggNTQgMzEgMjYgMDMgMDUgNzUgMDQgQzEgNjEgNDMgMTUgOEEgNDAgNkIgMDEgNzIgQjggNTcgN0IgOUIgNTMgRjAgMzUgMzQgQTEgNkYgRjAgMTcgOEYgN0MgRjIgMUEgN0IgQjQgMDUgMjEgQUQgMDIgNjAgQ0EgOUIgNDQgQ0IgNUYgNTkgNzQgNkMgRTYgMTIgN0MgOEQgMzQgNEIgNkUgQTQgNUEgNzkgN0YgNTIgNEQgRDAgMEQgMDQgQkQgNjEgMTEgRkMgMkIgRTIgNzYgODQgREMgMDkgMjYgNUMgNjYgQUMgNTAgMjcgRDcgQzEgRjMgQzggMzAgOUQgQTAgMUEgRTkgODAgODIgRUQgNTQgNUMgOTIgRDAgRTkgNEUgNTMgRkUgRDQgOEIgODUgMjQgQ0UgRUEgNDUgNEQgQTYgRDEgQTEgMzIgQkEgMjUgQzUgNzUgMkIgRUMgOTcgMTQgNkQgNjMgNzMgRjUgNjAgNTkgNzMgM0IgOUUgNjUgQ0YgRTMgOEMgNDcgRkMgNjIgQUYgNEUgNzIgQzQgMkQgMUMgQzggNjIgRkYgNkQgQzYgNTkgRkUgOTUgREIgQ0QgMDAgMTYgNDcgMzcgOTAgN0IgRkYgNEYgNDIgQkMgNUQgMUYgMkEgQzcgRjcgMDYgNTggMUYgRDggMTUgRDYgMDYgNzkgQTUgRjUgRDYgQjMgRDQgREQgN0IgRTAgOEUgQ0EgRUIgNzcgNzAgOUMgMzAgOEEgNjQgOTkgNTIgNTAgRTggODMgNTUgQkIgQ0EgMDEgRkEgNzMgQ0MgRUIgNTggMTYgNjggRDEgNTggNTYgQkMgMUEgNDIgRDMgMkEgQTQgRDEgNjIgNTAgN0IgMTIgMkMgRkYgMzAgNDggQjIgQzQgQUQgQTEgN0MgODMgRjMgN0EgRTcgNjAgQzIgM0YgM0IgRDYgMUMgQjAgNzMgMzEgREYgRTIgMDYgNUMgMzcgQTAgMzIgMEQgRkUgNUUgRjMgMUUgRDQgNkUgRjUgRkUgMjUgQTMgMzggNzAgNUQgMTUgOTYgQzYgNkUgNzggMzEgNjcgQzkgOTAgQ0YgNUMgRDEgNEUgMkYgRTIgQkUgMzggMEUgMzcgMDcgRjIgQjMgNDAgQzYgM0YgQjIgNDEgNkEgNEYgMTYgNUEgNjEgQTEgNTEgNjQgM0YgNEYgMDQgOTMgN0IgMUQgRjggMjAgREEgNDcgQzggREYgRjIgRjAgMjggMEUgRDMgQjYgN0YgRTkgQ0MgNTAgNjggQTUgNDUgNjggMzAgMDkgQUYgN0EgRjIgQ0IgQ0YgMTYgNzIgMzEgOEMgMDggOTYgNEYgMzQgOUYgODIgOEEgODAgN0QgOTUgM0YgNTAgMjIgMDIgQkYgNTAgOTEgQzcgNjkgMEQgQkYgQUMgMjcgMzMgMkYgQzIgNzggNzcgNUQgNkEgNUEgNzcgQzMgNTAgOUYgNzggNTcgMUEgNTcgQTQgODYgMTAgMzEgQjIgMjAgQTEgMjkgM0IgNzQgNUUgMTcgRTYgRDEgRTIgQkQgNTEgN0QgMEYgODcgQjYgMkYgQTkgOTMgMjMgMEIgODcgQzUgODIgOEQgN0MgMEYgRkEgREEgNDMgMUYgMDggMDkgOEMgRjggNjYgMjAgNkQgNDggRjAgNjIgQ0QgMDIgNUUgRDIgMUYgMTAgNTEgRjcgNEYgNEYgNjQgMDMgQjggNjIgMTUgQzAgQkYgOTYgRTUgQjcgQUEgMjkgM0EgMTUgM0YgQzAgMzkgODkgNTAgMjIgQjcgNEUgQUUgOUEgRkIgQ0EgODMgRkUgOTcgREMgMzcgRUUgQTUgQjMgMDUgRUYgREEgRDIgQkUgRDggNkIgMDIgQjQgRkUgNDcgREUgRUEgNTIgMzAgQ0QgQkIgMjIgOTggODEgQTUgMTUgQTIgRkYgMzAgNTggRDMgNzEgOEIgMTAgRDMgNTEgMDcgREYgRjYgNzMgMDQgRUEgNTkgMDMgN0YgREIgQkUgREUgMTggRjIgMTIgMEIgRDIgMTcgMkEgOUMgODEgNTcgMTIgQ0UgNjEgMUYgRkEgRTMgNzcgMTQgQUMgREQgRUUgMDMgRjkgNTAgMTkgOUYgQTkgMEEgMkYgQkMgMjAgMUUgQjkgNzEgMDQgODAgQjcgOUUgN0QgQTggRjAgM0IgRTggRDAgQjggMDkgN0EgMTkgQUYgM0UgQTQgRTggRDkgRTIgNEIgRkEgNjAgRUUgODIgQTQgQTIgQTYgQTEgMjggRDcgM0IgOUIgRTQgNzAgRTYgNkQgQkYgODYgRDMgRTMgRTIgREEgQjUgQUIgNTIgREMgRjUgNDEgM0QgOTEgRDcgM0EgNTMgRkYgNTQgMjQgRDQgNzIgNkQgQkQgMjMgNTMgOUQgQ0MgQzEgQjEgMDMgNTQgQzkgQkMgNjAgOEMgNDQgRDMgNTkgODYgNkYgODMgRTYgNDIgRkIgNTYgMDcgM0MgQ0QgQ0YgQzMgRUQgODYgNzQgQTcgRjQgNjEgMjUgNkEgRjYgMjMgNTggNzggNTAgMDkgNTAgNEQgNzAgQkMgNkEgOUEgRjAgMzEgREMgNTYgMjYgRTUgQTEgN0YgN0UgRTQgQkEgOTcgMTggRTYgRjMgNTMgRTMgOEMgNkUgNDMgMjUgNTYgNEUgODUgREIgOUMgQUYgMUMgNTYgMzEgOEIgMzIgRjMgRUEgRjkgOTUgMzQgQ0IgRkUgNEQgNEIgRTMgM0EgMDYgNDcgNzYgQ0EgREUgNDMgMjMgQkUgOUMgN0YgNjYgODkgQzQgQjIgQjUgN0IgMDcgQjUgRjUgRkIgNTggQzcgODIgMzcgMDIgRjcgMDggNEYgRTAgMTEgQjUgMDMgNTAgMEYgOUIgN0QgMTkgNTggQTYgRDYgNjcgMDYgNDkgN0EgMTkgNDIgRTIgMzcgQTUgRjEgRTggOEQgNDUgNzUgMjcgODYgQzQgRTIgM0IgOEUgQjMgNUEgNzEgMzEgNzggOTMgRDIgRDEgNDIgMkYgQjYgODMgNzQgQzggNTUgMzIgNjYgOTMgQzkgMkYgQkUgQ0YgOEEgMzcgNEIgNEIgNUYgMzcgRDUgN0QgMDMgNkUgQUEgNzIgODYgNEYgMzUgMDEgRTggRTcgREYgQ0IgNjUgOUMgMkEgNUEgRkEgRjcgQzUgOEQgNjQgNjQgNTYgQTQgNDIgMjAgREEgQTggMkUgRUIgNjMgODcgOEMgRTYgMzggQUUgRUMgQjEgNDIgNTEgMzQgMTkgNTcgQzUgQkUgNTcgQTcgN0IgOEIgRTkgMzggRjYgQjUgQTcgNEQgRjYgMzcgRDAgRDkgRTYgNEIgRjggRjIgQjEgN0QgNTcgMkUgMjIgNTIgQTUgNkYgQTcgMEQgQUIgM0MgRTggRUQgRUEgRjkgNjAgMDMgQzUgRDEgNDkgODQgNjUgOUEgQTMgOTEgOTQgRjEgQjMgRkQgMEIgQjQgQTggRUEgOUYgNkYgMEMgMjQgMTggOTkgNDcgNkMgNzggMkYgRTMgMEIgRUEgRUEgMkUgODQgNTIgNjEgOTQgNEIgOEIgQTkgODYgRkYgM0IgOTQgNDMgNEUgQ0EgOEQgMzggRTQgQUQgM0YgMkQgRTUgQUUgNDAgQkYgRDkgQUEgMzYgMDUgMEYgMDMgMUMgNDkgODEgMzYgRUYgNkYgRTAgMkIgNkMgMjkgNDEgRDAgOEMgNUYgOUQgMjQgOTEgODQgQkUgRUMgOEMgNzAgRkMgNDggNTUgQzAgQzAgRUIgQjYgNzggOTYgMzQgQkQgRDQgNjUgQTEgRjIgMjggRDUgMEQgMEYgOUIgRTMgODggQTkgODUgREQgOTkgRDIgNkMgQzkgRjQgNzYgQzYgMkMgREYgQjggMUIgOUIgMEIgODMgMzAgOEUgN0EgMTIgODggNDIgN0IgOUYgQjQgMjQgRTAgMzAgMUMgQkQgRDUgMUIgQTggMDkgQTUgMUUgMzggMTUgNTUgQ0UgODkgRUEgMjggMUEgNjggOEUgNUQgQzggMTQgODggMTQgMTEgQTQgRTggOEYgQjAgMDcgNzIgMUIgOUYgOUIgM0YgOEMgNzEgNDcgRkYgQzkgMDAgNEEgQjUgREQgRUQgNkMgOTQgN0IgNDggMkYgRDMgQjQgQkYgQ0UgQTAgODQgNTEgOTAgOTUgMzMgMTYgOTggRDIgODcgOEIgN0YgQzMgODUgQzMgNzggRjUgM0IgNkQgMkQgQkQgMzEgNkUgQkYgNjYgMzMgQzkgQTEgNjIgQTcgRjYgRkQgMTUgQkIgNUIgOUEgRkUgMDkgNDEgREYgMjUgOTMgREEgMjggODAgQzIgQ0MgMTAgNDcgQTQgRTkgOEYgMzYgRDkgMUMgNDggNEYgRDkgMzcgQzUgODYgQUYgNTcgQTAgOTcgRTIgMjYgRTcgRjMgQ0EgQTMgQjggQ0QgNTIgNzAgMUIgQTMgNTEgMzUgODMgQzEgQzcgRUYgNkYgQzcgOTUgODQgMzEgRjkgQ0UgMjYgNTIgMjggNEUgNDkgM0IgMTYgRDEgOTcgMjAgQzAgNUEgQjUgQjEgOEQgMzkgMkUgRTIgNjYgMTAgOTUgRDQgQUUgQUIgQ0YgNDYgRTIgRTcgNUUgMDUgRUMgQjYgRDAgQ0IgNkQgRTggQjMgRTEgQTEgQTIgMTYgOEIgRDMgQzMgMEMgMTMgQjcgNjggOTYgRjAgMDAgNjYgQzggNUMgQTkgOUMgQkUgQjUgMzUgMEYgMkQgNDQgNUEgQzcgMkMgMjggOTIgQ0YgRjkgQ0MgMEYgODIgMUYgOTEgMzcgQTYgMTIgOEIgNzMgMjQgMkQgMEEgQjAgOTAgMTUgMjIgOUUgRjMgMkMgRkIgNTkgMDUgODcgNDcgMDEgNkIgOTMgOEQgQzEgMUIgODMgN0YgQUQgQjMgRDkgNEQgREIgMzggQjcgNTkgRTMgOEQgNzkgMTkgNzcgMzUgNzIgOTMgRTggQUIgRTEgRjIgRTkgNTAgQUEgQjQgNTMgRDMgNjggRjMgQjQgNTggMzQgODUgQTkgMUIgRTAgODYgOEIgMTIgMzQgNUIgMTIgMDAgN0UgMTAgNkIgRDkgNjAgQkEgRjAgMkMgNDMgNTIgOTkgNTggMDMgOUIgMTYgMjMgRDcgM0EgRDcgNUIgQ0QgOUEgQzggNkUgNjAgQTUgNUYgN0IgRjYgOTkgMjEgMDkgQkIgRDAgRDggNTAgQUEgRUIgNjEgOUMgQjkgMUYgRDcgMEIgN0IgRjggREMgQ0QgMzUgMTcgODEgNjkgRTggOTYgQzkgNjcgOEMgN0EgNkQgOTYgRjAgODggNkEgRjIgRjcgNDYgRTAgMzUgRjggRkQgQUYgMUUgM0YgNDEgNEEgMDggNzkgRDggOUEgMDggMjYgRjkgNjkgOEUgNjEgREEgQzQgQTIgM0QgNDkgN0UgNDggQUQgMjAgNEMgNzYgQzggRDQgOUQgOUUgNDYgMDcgQUMgRDYgRDMgNDAgMzQgN0IgRjkgODIgRTggMjcgRTEgQkUgMUYgRjAgRTAgN0EgQjYgRTEgMkIgQjMgOTggMDMgOEQgOTcgNjUgMDQgNEEgQ0IgRjggQ0IgNkIgNkEgQTkgNTkgQTYgMTIgODcgMDcgMDAgODkgRkYgNTEgNkQgRjYgM0MgOTIgNEIgRUUgODEgRjcgMzYgNkQgMUIgMUQgMDAgQ0QgNjIgQjYgMkQgN0IgQkYgODAgMkIgNTIgMEEgQTEgRkUgMEIgOTggREMgMzkgODcgNzEgRjggNDQgREMgNTYgQkYgNEEgRDkgMEYgQ0EgNzggNkEgRTcgRjAgOUQgNDkgNEQgMDcgMzYgQzQgNEQgRjkgRjYgM0EgRkEgRUEgRjQgMTkgNEUgMDkgNEQgMDkgMDkgNUQgMkYgQzQgQjkgRkUgNEYgRjIgRDggQjIgMUYgRTEgNTEgMEMgN0IgQ0IgMUUgRTMgNTkgNkYgMUEgQkIgRDkgODcgOUEgODggMTggMzcgMEUgRUMgNzIgNkYgMDcgREYgMTAgQzUgQzEgRUYgQkIgOEYgNDIgNjUgNjMgQUMgMzQgRkUgQzcgMUQgRkYgODEgNzIgMkYgRDUgMzYgRTcgOEUgMzkgMDQgQzAgMDcgMTMgRjggNDMgRjUgRjEgRDIgMEMgRTEgNEUgN0IgODEgRDggNEMgRUEgNDcgNTEgN0IgQjYgNDcgQzUgRDkgQjIgN0IgRTggODIgQ0QgM0IgNTcgMzMgQkUgQkMgOTQgOTAgMjIgOUYgODYgMTMgNDMgMTcgQkUgRUUgN0QgOUUgRjkgQUUgQkYgM0MgMTEgRjEgNDUgQ0UgM0QgN0QgMEQgMDUgOUQgQUUgNEQgMTIgODkgNzEgNzcgRkYgNzAgOEIgQTkgQTkgNEQgQTggRUQgRUIgQjcgRUQgMTAgMDEgMEUgQ0IgRkUgMjAgQkEgMTIgRUUgNjEgMjAgNzIgODYgNkEgQjYgQkEgOUIgQTcgNDMgNzIgQ0UgOEUgQzcgRTEgMUUgQ0QgRjkgMTggRjYgM0UgNjYgOEIgMDUgNjEgOEIgRkYgMTIgNkYgMzUgMjUgRTAgQ0UgQjYgRjkgMzIgQUUgNUIgQjMgMkUgQzEgQUYgQUUgOEMgMDcgRkEgQjYgQ0IgMUEgOTIgMEQgNzIgMkUgMzggNEQgMTAgQjcgOUMgMjkgQjAgNzEgNzEgRUIgNzEgQzYgMDkgQzEgMjMgQkIgREYgREQgQTMgRDggQUUgOTggNDcgOEUgQ0MgMjggRTYgODkgNUEgNUQgOEMgNDYgRkQgRkYgRUUgQ0MgNDEgOEEgMkUgOEIgRUEgMkQgOTkgNDMgNjggRTQgMUMgNjMgODQgMEUgQkEgQjAgNUEgNkUgM0MgQzQgRjMgNDkgNzYgMzMgQzQgQjMgOTcgMEMgRjMgM0IgQTQgMkEgRUIgMUIgOUQgQTIgNDkgQTMgREIgMkYgNDIgRjcgNUYgMUIgQkYgNUIgMTAgOTUgOTYgNjQgNzYgRTAgMkQgNTAgODQgNkYgRDggQUUgRUEgMjkgNDcgQkMgMDAgRjAgODAgNDMgQTAgMUMgQ0MgQzMgRDkgQzkgNzggN0UgRkMgRTAgODkgODYgN0QgMEMgMUMgNDcgNEQgOUYgOTUgOTQgMUIgRTYgNTkgNDAgMjQgNzUgREUgQUQgREQgQUYgODQgODEgODkgNjAgQ0MgODUgREYgOTkgMTUgODAgMjUgMTYgQzMgMTIgNUIgMzIgNjggNEYgMEYgMjQgQUQgOTggRDAgNjcgMjQgNjYgQjUgQ0EgNUIgODcgMkEgNTAgMkMgNDAgMjUgNjIgRTMgNDAgMjYgNjYgRUUgMkQgMzkgRTAgQjQgRkQgRkMgNEQgODMgOTAgMjAgRkEgNDMgREYgNzIgOTcgQzcgMDEgNTIgODggOUUgRDQgNUYgNTUgMTkgQzAgMUIgNEYgQjYgN0MgMzggQzggMDkgRjMgRUMgOEEgN0MgMTIgQzMgOEMgQTcgNzcgMTIgRTYgODMgN0UgODMgQTggM0UgM0UgRDQgQkUgQzkgQzAgMzMgMDggMEYgNTQgRTAgMUYgMzIgOTIgREEgOEIgNTAgQUMgRDkgMDEgMzUgRkEgNTEgRDAgNDMgQkMgMjYgMjkgNjAgRkUgRDkgRTkgODYgRjkgM0EgRkYgNkMgMkEgMzkgRkIgMTMgMEUgQjUgNDQgMjMgMzkgNjYgNDUgRDkgQjggQUYgREEgM0IgQzMgRTAgRTAgQjIgNEMgQjEgQTEgODIgNUEgRTEgMjUgNTEgRDggODYgNUIgMTcgOTIgQzMgREEgMjQgNzcgMTQgOEYgMUYgQjAgQUUgRUUgMDcgNTcgQ0UgMzEgRDEgQjAgRDEgQTggNDQgOTcgRDAgRkYgRUIgRTEgMjIgMDQgNEUgODcgRkIgREMgRDAgMUQgNjUgRkIgRDIgQzkgRUUgQjEgNjMgOTcgQjEgN0EgQ0QgMzEgMjIgREUgOTUgQUIgNUMgQzggMzQgNTYgRDQgQjMgRjIgQkUgQjAgQjMgN0UgODggQUMgMzIgRDggQTAgRkEgRjggN0QgMjUgODYgOTQgQzIgMjcgMzcgMEUgMDkgQzEgMEIgMjggMEUgNjYgOTkgNkEgRTMgNjcgODIgNDYgNDcgQzEgRDMgQTUgNUIgRkEgNzAgNDQgMUYgQ0MgMDggQjQgQTEgOEEgQzQgQjQgRjkgNjMgMDAgNjMgNzcgNUMgRUUgQTQgNDggMkIgQzcgMzkgMUUgOUMgMDYgQ0IgOUEgMjUgQjIgM0MgNEIgNkQgQ0IgNEMgMDQgMzAgQ0QgNEEgNTYgMDUgN0IgNjEgQzIgNjcgQjQgMkEgRTggRDMgNDIgRDUgNjIgRDUgOUMgRkMgNjggNEQgNjUgOTcgMTkgMTkgMEUgMzggQzIgNEUgMDcgMkYgMjYgRDAgOTIgQjUgNTAgRjUgMDIgOEUgREQgREUgNzQgN0YgQ0MgRDcgMTkgNEEgRDIgNzEgODEgOEMgQTggMUEgMjcgQjQgNDQgQkYgMEUgMjUgNDAgNUEgNDcgQzIgQjUgNTYgNDEgN0IgRTMgM0YgNTkgNDMgQjcgRkIgMDQgQUIgQjYgNkIgQzkgNkEgMTQgQjUgRTYgOUQgNUEgOTcgOUQgOEEgMjAgNkUgNkMgOEQgREIgQjggRTYgREQgN0IgRDMgODggMDAgMjkgMzcgNTMgOEMgNEIgMEEgODAgMUEgQkEgNTkgRjEgRTUgQUUgRUYgQTMgRjcgQkEgRkYgRUQgQkQgOUIgOUEgNzMgMkQgNEEgMTMgMjIgQzYgRDIgRUMgQUYgNzMgNjAgRjcgREIgRjEgQ0IgMUEgNTAgMTIgODEgN0YgM0UgQTggNDcgQzcgNDggMUIgNjkgQjkgNzIgRjUgQkMgQjQgRDcgOUEgQzcgMzcgNzIgN0MgRUQgNTQgNkQgOTcgQTQgRUYgMUYgNjkgQzggMjAgMEUgM0IgOEEgQ0EgNTkgMjQgOTQgQTcgNDYgN0YgOUYgQjUgQUEgNzEgNkUgQTIgODUgNUYgNDkgQkMgMjYgMzEgQkMgQkIgNjkgQjEgRkUgRjAgNjYgMjAgREUgRkIgNUIgMUMgQTMgRUMgRkYgNjggQ0QgRTAgMzYgNjUgNjIgNTcgRUEgM0MgREIgQjQgQTkgODkgNzIgNUUgRjggREUgMzMgQjMgMDIgNTUgQkEgNjEgREMgMUQgRTkgNDQgMEYgODIgNzYgQUMgODAgNTMgOUIgQ0MgOEQgODQgM0MgMjMgODkgRTYgM0UgNjAgOEUgRTYgREEgRTggQ0YgMkYgQUYgMzYgREYgQTYgNUIgMEUgQTkgMjUgN0YgODUgNTggNEIgNkEgMEMgOTMgMDQgMTMgQzQgNjkgMzYgMkUgNjggQTUgMjIgQTcgMUMgMDEgMjEgMkYgRjkgNUIgQ0MgRDMgNDEgQjEgRjMgNUUgRjggNUUgODggOEMgRUIgNUIgNUMgRDAgNzQgNDIgRDIgODQgQ0EgQjEgNzkgMzYgNUQgQTMgRUIgNTIgQTEgQjYgNjcgQjkgRjUgNUYgMjcgM0UgMEYgQkEgNUIgNEUgRkMgQzggMzQgMUEgQkEgQUUgQUMgMTAgMzcgQzMgNEMgQ0QgOTkgOEYgMzcgRUIgOTggNkYgQzMgQkQgMDAgMkIgMjkgODggRjggODEgMTUgQzYgNzIgRTQgOTcgODMgOUMgRDggNTggOUIgNDcgQzEgNTQgNDQgNjAgNjQgRTQgMjIgMDggNkIgODAgRkUgQTkgMUIgMEMgMEUgM0MgNUMgRkYgNDIgRDUgMEEgQzggMEIgRjMgMTcgNTUgQTggMzMgMjIgNjMgNzkgMEQgMUEgODAgNDcgQjMgMTQgNjYgRjMgNjUgNzEgNDcgNDUgMTUgNUYgODcgMEUgNzQgMDcgMzYgRjUgQTggRDYgNjggQTcgQjggMjggOEIgMEYgMjUgQzkgQkEgNkMgRkMgRDIgODAgQzggMTYgNzIgQjMgMEUgMjEgRjggRkQgRkMgOEIgQkYgMDAgMDYgNTEgMzYgMjcgNzcgMzAgQTUgREMgQzEgRTcgNDYgMjIgQ0EgMUQgMjggOUMgMzYgMjIgQ0IgMzMgNTcgRTMgRTAgMzIgMUIgMzMgREQgMDQgMjMgOEQgOUYgQTYgOTIgQUEgQzMgQjIgMDAgNjAgNjIgNTIgRjkgMUUgMTQgMTYgRTMgMTMgNzcgNDYgRkUgOEEgRjAgMkQgOUMgQjggODkgRTcgOUQgM0EgMDkgMTMgQzUgMDggQTggMkEgNjEgQkIgQUYgQTkgNDQgM0UgQjAgNTYgMTYgMEEgRjggOUYgNTIgRkMgRTUgRkUgQzcgOTIgN0IgNEQgQUEgM0MgRTcgMEIgRjIgNTggQUEgRUMgNEQgOEQgRTUgN0IgRDIgNjcgRjAgODMgNEYgMDggRjMgRTcgQjQgNEMgOEIgODQgOTMgNkEgQjYgRjcgNTIgOTAgQTUgRjcgOUQgNTkgQUIgMjYgRjcgQjAgMjYgRDYgNDYgRTkgNEQgRjcgRkYgODggMjMgRUMgQkMgMkEgMzYgRDUgOEQgNEQgQTQgRDkgNzQgOTMgQ0MgRUYgRDQgRjMgNUIgMTMgODUgNDQgMEUgMjEgRUIgRDkgMjYgODUgMUIgODkgNDYgRTMgMjYgNTAgQzcgNTAgQ0YgRkIgNDIgMjEgRTMgODUgMkMgNDUgMEYgOEMgRUMgOEYgNkYgNDEgRjAgN0MgQzcgQkIgMEMgMEEgNjggMTMgRjEgNDQgRTcgOUMgNEUgQ0EgNTUgNDEgREYgNDAgN0MgRjYgMTIgODUgMTkgMzQgQzEgQTkgNzUgQkYgQjAgOEYgNzIgRDYgODYgQTMgNjMgNjcgNjggNkIgM0MgREEgQkMgNTkgQkQgM0IgMEMgQjQgNDkgNjcgRkYgMDggRkEgRDAgQTMgOTYgOUEgNDcgRUIgRkIgNjkgMjQgOUQgRTMgMjggMTMgQTkgMjQgNjcgQUEgOTAgQzcgNDQgQjQgMzEgNkQgNDUgRUMgMjUgMDIgQzQgNjIgNzMgMkYgRkEgNDcgMjggQkYgQzAgMjggQkIgNUIgNDAgNTcgN0IgOTQgQzggQzAgOEYgRDUgNTEgQzcgODIgQTUgQTQgMjEgNTcgNjcgNEMgOTQgNzggMjYgREMgQjcgQzQgQTUgNkYgQjggMTEgQjUgMTIgNzcgNDYgNkQgNTggMTIgMTMgNTMgMTkgOUIgMzYgNkIgMTMgMDAgQTEgMkMgRTEgOEMgMTUgODYgMkYgNTMgOUEgMTQgNDAgQUEgODAgMjAgQjQgMjYgOUYgRTggMUQgRTEgMzcgN0QgRkYgQjAgN0UgREYgRkUgQkYgM0YgNkUgNjkgMjEgMTcgRTEgNTQgQUIgNTIgM0UgRjIgM0YgNjMgNEMgMjQgQTkgQzQgMUUgMUUgOUUgNjEgMTcgM0IgRjggMjggMzggRDUgRjYgQ0QgNDQgMzkgQzQgRTcgNTEgNjcgRTkgNkQgMTkgNzYgQzIgNjIgQTMgOTggRUEgNEIgMDMgRkUgN0IgMEQgMDggNjEgRDAgNzQgODMgMzAgOEIgQjggMUQgNDUgN0EgNjAgMEMgNDcgNjcgNzkgOTkgOTMgNjkgMzAgODUgMEYgNTIgQUQgQUEgMjAgQkYgNjMgMzkgRDUgQ0EgNUUgNTcgMDMgMkUgMUUgOTAgNjAgQTcgMzMgNkIgMzIgMTkgNUYgRUEgMDEgNzEgQ0YgMjAgNzggOUEgNUYgQjQgQTggOUYgNDUgQTMgMEMgMzMgMjQgODMgMkYgQjYgREYgRkQgNUIgNjMgNTMgRkUgMjMgRDUgQzIgNzIgMTEgRkUgMjYgQUMgMEEgOEUgNUQgM0MgMzMgNTcgMDkgMkEgRjkgOTAgQkIgMUQgREYgMTYgMkEgREMgNkIgRjMgM0IgODUgNDIgOTYgRjkgQ0EgRDkgNzUgREMgQjEgNUMgQkUgODkgNTEgNTEgRTQgQ0IgQjggMjIgNDAgOTIgRkYgMDYgRDQgNjggODYgQTAgN0QgRkEgOUMgNEQgOEQgMjYgREYgQUQgMTEgMkMgQjAgNjAgRjggNUEgQkUgRjEgNDQgNUUgOUMgMjggN0EgMkEgNzIgQ0UgRUMgQkEgMjAgMUIgRDQgOTAgNjUgQ0EgQzcgQzQgMDkgRUIgRTEgQzYgMEIgQjMgNTUgOTggRDkgRUEgMkEgQUEgRUYgRTYgNjQgRTkgODMgMkIgNEYgQzIgMjMgRTIgM0IgNjkgNjIgRjIgMzkgOEIgMEUgMjYgRjcgQzkgOTUgRDIgQjEgMDEgOEQgQzkgQUEgQ0MgOTggMDUgNDcgMjUgN0EgNjggOUQgRUQgRjUgOTkgMzEgRkEgNUEgRDEgQ0YgQjkgREQgNkUgOUYgREQgNTEgM0QgQjUgQ0MgRkIgNkIgMDIgOUQgRTEgMTkgNDUgMkIgM0EgRDkgMTQgREIgMzcgNUMgMzggNEYgMzcgRDYgNzcgNjAgN0QgQjAgOTMgNEIgNDUgRjQgOUMgQTYgQzQgRDMgN0YgNjcgQ0YgRTggMzAgODYgNzkgQTUgQkQgMkUgMUMgNDMgQTAgMDMgNDYgNDQgRTYgNkIgOTUgODQgQTAgNjUgM0YgRDMgODkgRDcgN0EgMDMgNkEgQ0EgOUEgOEYgNTEgMjAgN0YgNUYgRkMgRTkgNjEgMDUgOEUgOTggRDMgRkQgQ0YgMzcgODQgRkUgRUMgMTkgN0UgNDEgQjQgRUQgMUMgOUUgOTkgMzMgMkUgRTkgOUYgODggQTEgRUMgRkQgMjkgOEQgNDMgRDcgNDUgQ0QgNzQgNEEgMEYgM0UgRjkgNDYgQjIgMTMgNEYgMUMgODQgMTQgMDcgODIgQjcgRUQgM0MgNTkgNDkgMkMgRDEgMTcgNzQgQjAgNjQgQjQgNjkgNzUgMjAgNEMgMjIgMTIgNzkgNTQgNTggNzUgNkYgOEEgNUMgRjcgQTYgNzAgODYgQ0QgMDAgNjIgMTQgQjcgOEUgMzEgMDIgOEYgNzcgMTMgQjYgNDUgQzAgOUUgQkQgNTggRUUgMDMgQjkgOEUgNTggM0QgNjggM0MgMzQgNEMgNTEgRkMgNEEgM0EgRjggRjYgMEEgMTkgNkMgQ0UgRjEgMDAgMTUgNkYgQTkgRDggMTAgRUMgMUQgMkQgODYgRDMgMUUgMEIgMEEgNEYgMTcgODMgQTIgNDcgMDkgN0MgRDUgRkQgMTkgQUEgOUUgMDEgMkUgRTMgOTQgNEUgMUMgOEMgRDQgNUUgMUUgRkEgN0QgMDMgOUMgNzkgREEgMzUgQ0EgQTUgNDIgQzcgNTAgRkQgRTYgMkEgMjQgMDEgODMgRTcgOUYgRTcgQjQgRDQgQUMgMTEgMTcgNUUgQUQgNTcgNTYgMDggNzkgNjQgREQgQzQgQTcgRkUgODkgN0YgNTkgQ0YgMEIgMDggQkIgQzEgN0MgNzkgNjcgMjQgQTAgQzYgOUYgMUUgRUEgOEQgNjUgOTEgQzYgRjkgQzggQTUgRDYgNTEgOEEgNDQgREEgQkQgNDMgMjMgNEQgNkMgNEMgM0UgRjMgMUEgMUUgN0EgMjEgQTEgNTAgQkIgRjAgQTggRkEgQzggRkEgMjggMzUgM0UgMEEgNkYgRkMgOUMgQ0IgMzUgMTggRTggQjkgNDkgRDMgQkUgOEIgREIgNzQgREQgRTYgQTEgMDQgRkYgQ0IgODggMDkgNkUgOUMgOTcgNzkgQTkgQTkgQjUgMTggNDYgMzAgQjYgRkUgNjggQ0IgODIgOTggRkIgQUIgNzIgMkEgRjkgNDIgMTggMUUgNzAgMjEgMEEgRjcgMzAgOEYgMDggODAgODQgRTQgMUEgOEYgODAgNDkgOEYgMjEgNUMgOEMgREIgQTYgQ0QgRjkgQTggOTAgMDYgNzkgNjcgNDQgODMgRDMgQjcgQTcgN0IgMjUgNzUgOTEgQTYgRDYgMzAgRTYgOEMgNkMgQ0QgNDcgMDkgMkIgMTEgQzYgQzcgMjEgN0UgODcgQTMgN0YgNDMgMDMgMzUgNUUgMTQgMzggRjkgMkEgOUEgQTkgODggRUYgM0MgQzIgMkMgODIgNEMgRkEgNkMgMDcgREUgMkEgQUEgOEEgQzcgNjQgRDAgRTIgRTAgNzYgN0YgN0EgNkYgOTEgREUgNjIgRjYgNjQgOTYgNzkgNzggODMgMTUgQjIgRkUgNDQgMDggRUIgMTQgNkEgRDMgMzkgNkMgMEMgRDUgRDkgNkUgNjcgRTEgNTAgOUIgMUQgQ0YgRDAgMUEgMzAgMEUgQTcgOEQgMTggNUQgNkMgMDggMEUgRTYgQzYgRDggRkQgRDkgMjYgMjcgM0IgODQgRUYgRDcgNTkgQkQgNzQgRUEgNDggMDMgMDYgQTYgMjMgQTkgMjkgQkMgQzQgQkMgQUYgMDggODQgQTYgRDAgQjYgMjUgOTcgNjYgMTEgQ0MgNjcgNjIgMzQgODcgOUUgODMgRjAgMEMgQTkgMUEgQjkgNTAgOTMgN0EgMjcgMkYgQjkgRUYgRkYgMzkgNTYgNUEgM0QgQ0EgMUIgN0QgQUIgRDQgQjMgRDUgNTkgQUUgRkEgMTcgNkIgQTcgQjEgMEIgNjUgOTMgODggNjAgODEgREMgNjkgMTggMkYgODEgRDQgN0QgOTEgQTUgNzUgNTEgMTAgNzUgMEIgMkYgMDUgQjEgRUMgRDggODMgNDAgMDUgMEUgQzUgQTIgRjkgNzkgODIgODUgQjggOTkgM0MgMDggNjEgRDkgNzkgMTQgQTAgMEQgNkYgMzQgQjggMDUgM0QgNEQgMzIgOEQgOEYgOTggOEUgNUMgN0YgQkMgQjcgNjkgQUIgNDkgQzUgQjEgQjAgNDQgMzkgRTMgMkEgRDEgNDAgRjAgN0IgQTAgQzUgQTggRDMgNjkgQTkgMDIgQ0IgNTEgREMgRjEgNjUgMjMgQjUgOEUgODYgRkQgMDkgMzkgRUQgMEQgRUMgRUYgQjQgQzEgRDYgMjggNjYgMDIgNTAgMjEgOEQgNUUgNzYgMDcgMkQgNDMgOTYgRDEgQUIgMjMgNzUgQTcgRjAgOUEgNDAgNUIgNjggNEIgRDAgMUUgRTUgMzAgMzUgNzIgMDMgNDEgOUMgQTAgQUQgMjggMTEgNTAgRjggQkUgREEgRTIgODQgNTEgRDAgQjEgMDIgMEUgM0UgMjcgQzMgNzEgRDcgRUYgM0UgMzcgRTQgNjkgQUEgNEYgMkMgNTQgOUUgOEMgNDggNjEgMUIgNjggODUgMTEgRkMgQ0YgNDQgNEUgOEMgNzIgNUQgNTAgQzMgMDIgMUYgMjQgOUYgOEYgNzIgMDcgNzEgNzAgQTYgQUYgNkEgQjcgNUIgQzEgRUQgMjIgQzggQjMgM0QgNDQgM0MgRDMgMjQgRUQgNzYgMTQgREEgMzggQkMgMDAgMDEgMDQgMDYgMDAgMDEgMDkgOEYgNTggMDAgMDcgMEIgMDEgMDAgMDEgMjEgMjEgMDEgMDUgMEMgQzAgQjkgNEIgMDAgMDggMEEgMDEgRDEgQTUgQzAgNjIgMDAgMDAgMDUgMDEgMTkgMDkgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMDAgMTEgMjMgMDAgNjMgMDAgNzIgMDAgNjUgMDAgNjQgMDAgNjkgMDAgNzQgMDAgNUYgMDAgNjMgMDAgNjEgMDAgNzIgMDAgNjQgMDAgNzMgMDAgMkUgMDAgNzQgMDAgNzggMDAgNzQgMDAgMDAgMDAgMTkgMDAgMTQgMEEgMDEgMDAgOUIgMkQgQTIgQ0YgQUMgODEgRDQgMDEgMTUgMDYgMDEgMDAgMjAgMDAgMDAgMDAgMDAgMDA
```

Ahora decodificaremos este base64, y tendremos lo siguiente:

``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura.pcap -Y 'http.request.method == "POST"' -T fields -e urlencoded-form.key | sed 's/$/=/' | base64 -d
37 7A BC AF 27 1C 00 04 65 3D 77 8C 58 0F 00 00 00 00 00 00 6A 00 00 00 00 00 00 00 8D 46 FF B9 E0 4B B8 0F 50 5D 00 26 12 47 11 A8 C4 08 54 31 26 03 05 75 04 C1 61 43 15 8A 40 6B 01 72 B8 57 7B 9B 53 F0 35 34 A1 6F F0 17 8F 7C F2 1A 7B B4 05 21 AD 02 60 CA 9B 44 CB 5F 59 74 6C E6 12 7C 8D 34 4B 6E A4 5A 79 7F 52 4D D0 0D 04 BD 61 11 FC 2B E2 76 84 DC 09 26 5C 66 AC 50 27 D7 C1 F3 C8 30 9D A0 1A E9 80 82 ED 54 5C 92 D0 E9 4E 53 FE D4 8B 85 24 CE EA 45 4D A6 D1 A1 32 BA 25 C5 75 2B EC 97 14 6D 63 73 F5 60 59 73 3B 9E 65 CF E3 8C 47 FC 62 AF 4E 72 C4 2D 1C C8 62 FF 6D C6 59 FE 95 DB CD 00 16 47 37 90 7B FF 4F 42 BC 5D 1F 2A C7 F7 06 58 1F D8 15 D6 06 79 A5 F5 D6 B3 D4 DD 7B E0 8E CA EB 77 70 9C 30 8A 64 99 52 50 E8 83 55 BB CA 01 FA 73 CC EB 58 16 68 D1 58 56 BC 1A 42 D3 2A A4 D1 62 50 7B 12 2C FF 30 48 B2 C4 AD A1 7C 83 F3 7A E7 60 C2 3F 3B D6 1C B0 73 31 DF E2 06 5C 37 A0 32 0D FE 5E F3 1E D4 6E F5 FE 25 A3 38 70 5D 15 96 C6 6E 78 31 67 C9 90 CF 5C D1 4E 2F E2 BE 38 0E 37 07 F2 B3 40 C6 3F B2 41 6A 4F 16 5A 61 A1 51 64 3F 4F 04 93 7B 1D F8 20 DA 47 C8 DF F2 F0 28 0E D3 B6 7F E9 CC 50 68 A5 45 68 30 09 AF 7A F2 CB CF 16 72 31 8C 08 96 4F 34 9F 82 8A 80 7D 95 3F 50 22 02 BF 50 91 C7 69 0D BF AC 27 33 2F C2 78 77 5D 6A 5A 77 C3 50 9F 78 57 1A 57 A4 86 10 31 B2 20 A1 29 3B 74 5E 17 E6 D1 E2 BD 51 7D 0F 87 B6 2F A9 93 23 0B 87 C5 82 8D 7C 0F FA DA 43 1F 08 09 8C F8 66 20 6D 48 F0 62 CD 02 5E D2 1F 10 51 F7 4F 4F 64 03 B8 62 15 C0 BF 96 E5 B7 AA 29 3A 15 3F C0 39 89 50 22 B7 4E AE 9A FB CA 83 FE 97 DC 37 EE A5 B3 05 EF DA D2 BE D8 6B 02 B4 FE 47 DE EA 52 30 CD BB 22 98 81 A5 15 A2 FF 30 58 D3 71 8B 10 D3 51 07 DF F6 73 04 EA 59 03 7F DB BE DE 18 F2 12 0B D2 17 2A 9C 81 57 12 CE 61 1F FA E3 77 14 AC DD EE 03 F9 50 19 9F A9 0A 2F BC 20 1E B9 71 04 80 B7 9E 7D A8 F0 3B E8 D0 B8 09 7A 19 AF 3E A4 E8 D9 E2 4B FA 60 EE 82 A4 A2 A6 A1 28 D7 3B 9B E4 70 E6 6D BF 86 D3 E3 E2 DA B5 AB 52 DC F5 41 3D 91 D7 3A 53 FF 54 24 D4 72 6D BD 23 53 9D CC C1 B1 03 54 C9 BC 60 8C 44 D3 59 86 6F 83 E6 42 FB 56 07 3C CD CF C3 ED 86 74 A7 F4 61 25 6A F6 23 58 78 50 09 50 4D 70 BC 6A 9A F0 31 DC 56 26 E5 A1 7F 7E E4 BA 97 18 E6 F3 53 E3 8C 6E 43 25 56 4E 85 DB 9C AF 1C 56 31 8B 32 F3 EA F9 95 34 CB FE 4D 4B E3 3A 06 47 76 CA DE 43 23 BE 9C 7F 66 89 C4 B2 B5 7B 07 B5 F5 FB 58 C7 82 37 02 F7 08 4F E0 11 B5 03 50 0F 9B 7D 19 58 A6 D6 67 06 49 7A 19 42 E2 37 A5 F1 E8 8D 45 75 27 86 C4 E2 3B 8E B3 5A 71 31 78 93 D2 D1 42 2F B6 83 74 C8 55 32 66 93 C9 2F BE CF 8A 37 4B 4B 5F 37 D5 7D 03 6E AA 72 86 4F 35 01 E8 E7 DF CB 65 9C 2A 5A FA F7 C5 8D 64 64 56 A4 42 20 DA A8 2E EB 63 87 8C E6 38 AE EC B1 42 51 34 19 57 C5 BE 57 A7 7B 8B E9 38 F6 B5 A7 4D F6 37 D0 D9 E6 4B F8 F2 B1 7D 57 2E 22 52 A5 6F A7 0D AB 3C E8 ED EA F9 60 03 C5 D1 49 84 65 9A A3 91 94 F1 B3 FD 0B B4 A8 EA 9F 6F 0C 24 18 99 47 6C 78 2F E3 0B EA EA 2E 84 52 61 94 4B 8B A9 86 FF 3B 94 43 4E CA 8D 38 E4 AD 3F 2D E5 AE 40 BF D9 AA 36 05 0F 03 1C 49 81 36 EF 6F E0 2B 6C 29 41 D0 8C 5F 9D 24 91 84 BE EC 8C 70 FC 48 55 C0 C0 EB B6 78 96 34 BD D4 65 A1 F2 28 D5 0D 0F 9B E3 88 A9 85 DD 99 D2 6C C9 F4 76 C6 2C DF B8 1B 9B 0B 83 30 8E 7A 12 88 42 7B 9F B4 24 E0 30 1C BD D5 1B A8 09 A5 1E 38 15 55 CE 89 EA 28 1A 68 8E 5D C8 14 88 14 11 A4 E8 8F B0 07 72 1B 9F 9B 3F 8C 71 47 FF C9 00 4A B5 DD ED 6C 94 7B 48 2F D3 B4 BF CE A0 84 51 90 95 33 16 98 D2 87 8B 7F C3 85 C3 78 F5 3B 6D 2D BD 31 6E BF 66 33 C9 A1 62 A7 F6 FD 15 BB 5B 9A FE 09 41 DF 25 93 DA 28 80 C2 CC 10 47 A4 E9 8F 36 D9 1C 48 4F D9 37 C5 86 AF 57 A0 97 E2 26 E7 F3 CA A3 B8 CD 52 70 1B A3 51 35 83 C1 C7 EF 6F C7 95 84 31 F9 CE 26 52 28 4E 49 3B 16 D1 97 20 C0 5A B5 B1 8D 39 2E E2 66 10 95 D4 AE AB CF 46 E2 E7 5E 05 EC B6 D0 CB 6D E8 B3 E1 A1 A2 16 8B D3 C3 0C 13 B7 68 96 F0 00 66 C8 5C A9 9C BE B5 35 0F 2D 44 5A C7 2C 28 92 CF F9 CC 0F 82 1F 91 37 A6 12 8B 73 24 2D 0A B0 90 15 22 9E F3 2C FB 59 05 87 47 01 6B 93 8D C1 1B 83 7F AD B3 D9 4D DB 38 B7 59 E3 8D 79 19 77 35 72 93 E8 AB E1 F2 E9 50 AA B4 53 D3 68 F3 B4 58 34 85 A9 1B E0 86 8B 12 34 5B 12 00 7E 10 6B D9 60 BA F0 2C 43 52 99 58 03 9B 16 23 D7 3A D7 5B CD 9A C8 6E 60 A5 5F 7B F6 99 21 09 BB D0 D8 50 AA EB 61 9C B9 1F D7 0B 7B F8 DC CD 35 17 81 69 E8 96 C9 67 8C 7A 6D 96 F0 88 6A F2 F7 46 E0 35 F8 FD AF 1E 3F 41 4A 08 79 D8 9A 08 26 F9 69 8E 61 DA C4 A2 3D 49 7E 48 AD 20 4C 76 C8 D4 9D 9E 46 07 AC D6 D3 40 34 7B F9 82 E8 27 E1 BE 1F F0 E0 7A B6 E1 2B B3 98 03 8D 97 65 04 4A CB F8 CB 6B 6A A9 59 A6 12 87 07 00 89 FF 51 6D F6 3C 92 4B EE 81 F7 36 6D 1B 1D 00 CD 62 B6 2D 7B BF 80 2B 52 0A A1 FE 0B 98 DC 39 87 71 F8 44 DC 56 BF 4A D9 0F CA 78 6A E7 F0 9D 49 4D 07 36 C4 4D F9 F6 3A FA EA F4 19 4E 09 4D 09 09 5D 2F C4 B9 FE 4F F2 D8 B2 1F E1 51 0C 7B CB 1E E3 59 6F 1A BB D9 87 9A 88 18 37 0E EC 72 6F 07 DF 10 C5 C1 EF BB 8F 42 65 63 AC 34 FE C7 1D FF 81 72 2F D5 36 E7 8E 39 04 C0 07 13 F8 43 F5 F1 D2 0C E1 4E 7B 81 D8 4C EA 47 51 7B B6 47 C5 D9 B2 7B E8 82 CD 3B 57 33 BE BC 94 90 22 9F 86 13 43 17 BE EE 7D 9E F9 AE BF 3C 11 F1 45 CE 3D 7D 0D 05 9D AE 4D 12 89 71 77 FF 70 8B A9 A9 4D A8 ED EB B7 ED 10 01 0E CB FE 20 BA 12 EE 61 20 72 86 6A B6 BA 9B A7 43 72 CE 8E C7 E1 1E CD F9 18 F6 3E 66 8B 05 61 8B FF 12 6F 35 25 E0 CE B6 F9 32 AE 5B B3 2E C1 AF AE 8C 07 FA B6 CB 1A 92 0D 72 2E 38 4D 10 B7 9C 29 B0 71 71 EB 71 C6 09 C1 23 BB DF DD A3 D8 AE 98 47 8E CC 28 E6 89 5A 5D 8C 46 FD FF EE CC 41 8A 2E 8B EA 2D 99 43 68 E4 1C 63 84 0E BA B0 5A 6E 3C C4 F3 49 76 33 C4 B3 97 0C F3 3B A4 2A EB 1B 9D A2 49 A3 DB 2F 42 F7 5F 1B BF 5B 10 95 96 64 76 E0 2D 50 84 6F D8 AE EA 29 47 BC 00 F0 80 43 A0 1C CC C3 D9 C9 78 7E FC E0 89 86 7D 0C 1C 47 4D 9F 95 94 1B E6 59 40 24 75 DE AD DD AF 84 81 89 60 CC 85 DF 99 15 80 25 16 C3 12 5B 32 68 4F 0F 24 AD 98 D0 67 24 66 B5 CA 5B 87 2A 50 2C 40 25 62 E3 40 26 66 EE 2D 39 E0 B4 FD FC 4D 83 90 20 FA 43 DF 72 97 C7 01 52 88 9E D4 5F 55 19 C0 1B 4F B6 7C 38 C8 09 F3 EC 8A 7C 12 C3 8C A7 77 12 E6 83 7E 83 A8 3E 3E D4 BE C9 C0 33 08 0F 54 E0 1F 32 92 DA 8B 50 AC D9 01 35 FA 51 D0 43 BC 26 29 60 FE D9 E9 86 F9 3A FF 6C 2A 39 FB 13 0E B5 44 23 39 66 45 D9 B8 AF DA 3B C3 E0 E0 B2 4C B1 A1 82 5A E1 25 51 D8 86 5B 17 92 C3 DA 24 77 14 8F 1F B0 AE EE 07 57 CE 31 D1 B0 D1 A8 44 97 D0 FF EB E1 22 04 4E 87 FB DC D0 1D 65 FB D2 C9 EE B1 63 97 B1 7A CD 31 22 DE 95 AB 5C C8 34 56 D4 B3 F2 BE B0 B3 7E 88 AC 32 D8 A0 FA F8 7D 25 86 94 C2 27 37 0E 09 C1 0B 28 0E 66 99 6A E3 67 82 46 47 C1 D3 A5 5B FA 70 44 1F CC 08 B4 A1 8A C4 B4 F9 63 00 63 77 5C EE A4 48 2B C7 39 1E 9C 06 CB 9A 25 B2 3C 4B 6D CB 4C 04 30 CD 4A 56 05 7B 61 C2 67 B4 2A E8 D3 42 D5 62 D5 9C FC 68 4D 65 97 19 19 0E 38 C2 4E 07 2F 26 D0 92 B5 50 F5 02 8E DD DE 74 7F CC D7 19 4A D2 71 81 8C A8 1A 27 B4 44 BF 0E 25 40 5A 47 C2 B5 56 41 7B E3 3F 59 43 B7 FB 04 AB B6 6B C9 6A 14 B5 E6 9D 5A 97 9D 8A 20 6E 6C 8D DB B8 E6 DD 7B D3 88 00 29 37 53 8C 4B 0A 80 1A BA 59 F1 E5 AE EF A3 F7 BA FF ED BD 9B 9A 73 2D 4A 13 22 C6 D2 EC AF 73 60 F7 DB F1 CB 1A 50 12 81 7F 3E A8 47 C7 48 1B 69 B9 72 F5 BC B4 D7 9A C7 37 72 7C ED 54 6D 97 A4 EF 1F 69 C8 20 0E 3B 8A CA 59 24 94 A7 46 7F 9F B5 AA 71 6E A2 85 5F 49 BC 26 31 BC BB 69 B1 FE F0 66 20 DE FB 5B 1C A3 EC FF 68 CD E0 36 65 62 57 EA 3C DB B4 A9 89 72 5E F8 DE 33 B3 02 55 BA 61 DC 1D E9 44 0F 82 76 AC 80 53 9B CC 8D 84 3C 23 89 E6 3E 60 8E E6 DA E8 CF 2F AF 36 DF A6 5B 0E A9 25 7F 85 58 4B 6A 0C 93 04 13 C4 69 36 2E 68 A5 22 A7 1C 01 21 2F F9 5B CC D3 41 B1 F3 5E F8 5E 88 8C EB 5B 5C D0 74 42 D2 84 CA B1 79 36 5D A3 EB 52 A1 B6 67 B9 F5 5F 27 3E 0F BA 5B 4E FC C8 34 1A BA AE AC 10 37 C3 4C CD 99 8F 37 EB 98 6F C3 BD 00 2B 29 88 F8 81 15 C6 72 E4 97 83 9C D8 58 9B 47 C1 54 44 60 64 E4 22 08 6B 80 FE A9 1B 0C 0E 3C 5C FF 42 D5 0A C8 0B F3 17 55 A8 33 22 63 79 0D 1A 80 47 B3 14 66 F3 65 71 47 45 15 5F 87 0E 74 07 36 F5 A8 D6 68 A7 B8 28 8B 0F 25 C9 BA 6C FC D2 80 C8 16 72 B3 0E 21 F8 FD FC 8B BF 00 06 51 36 27 77 30 A5 DC C1 E7 46 22 CA 1D 28 9C 36 22 CB 33 57 E3 E0 32 1B 33 DD 04 23 8D 9F A6 92 AA C3 B2 00 60 62 52 F9 1E 14 16 E3 13 77 46 FE 8A F0 2D 9C B8 89 E7 9D 3A 09 13 C5 08 A8 2A 61 BB AF A9 44 3E B0 56 16 0A F8 9F 52 FC E5 FE C7 92 7B 4D AA 3C E7 0B F2 58 AA EC 4D 8D E5 7B D2 67 F0 83 4F 08 F3 E7 B4 4C 8B 84 93 6A B6 F7 52 90 A5 F7 9D 59 AB 26 F7 B0 26 D6 46 E9 4D F7 FF 88 23 EC BC 2A 36 D5 8D 4D A4 D9 74 93 CC EF D4 F3 5B 13 85 44 0E 21 EB D9 26 85 1B 89 46 E3 26 50 C7 50 CF FB 42 21 E3 85 2C 45 0F 8C EC 8F 6F 41 F0 7C C7 BB 0C 0A 68 13 F1 44 E7 9C 4E CA 55 41 DF 40 7C F6 12 85 19 34 C1 A9 75 BF B0 8F 72 D6 86 A3 63 67 68 6B 3C DA BC 59 BD 3B 0C B4 49 67 FF 08 FA D0 A3 96 9A 47 EB FB 69 24 9D E3 28 13 A9 24 67 AA 90 C7 44 B4 31 6D 45 EC 25 02 C4 62 73 2F FA 47 28 BF C0 28 BB 5B 40 57 7B 94 C8 C0 8F D5 51 C7 82 A5 A4 21 57 67 4C 94 78 26 DC B7 C4 A5 6F B8 11 B5 12 77 46 6D 58 12 13 53 19 9B 36 6B 13 00 A1 2C E1 8C 15 86 2F 53 9A 14 40 AA 80 20 B4 26 9F E8 1D E1 37 7D FF B0 7E DF FE BF 3F 6E 69 21 17 E1 54 AB 52 3E F2 3F 63 4C 24 A9 C4 1E 1E 9E 61 17 3B F8 28 38 D5 F6 CD 44 39 C4 E7 51 67 E9 6D 19 76 C2 62 A3 98 EA 4B 03 FE 7B 0D 08 61 D0 74 83 30 8B B8 1D 45 7A 60 0C 47 67 79 99 93 69 30 85 0F 52 AD AA 20 BF 63 39 D5 CA 5E 57 03 2E 1E 90 60 A7 33 6B 32 19 5F EA 01 71 CF 20 78 9A 5F B4 A8 9F 45 A3 0C 33 24 83 2F B6 DF FD 5B 63 53 FE 23 D5 C2 72 11 FE 26 AC 0A 8E 5D 3C 33 57 09 2A F9 90 BB 1D DF 16 2A DC 6B F3 3B 85 42 96 F9 CA D9 75 DC B1 5C BE 89 51 51 E4 CB B8 22 40 92 FF 06 D4 68 86 A0 7D FA 9C 4D 8D 26 DF AD 11 2C B0 60 F8 5A BE F1 44 5E 9C 28 7A 2A 72 CE EC BA 20 1B D4 90 65 CA C7 C4 09 EB E1 C6 0B B3 55 98 D9 EA 2A AA EF E6 64 E9 83 2B 4F C2 23 E2 3B 69 62 F2 39 8B 0E 26 F7 C9 95 D2 B1 01 8D C9 AA CC 98 05 47 25 7A 68 9D ED F5 99 31 FA 5A D1 CF B9 DD 6E 9F DD 51 3D B5 CC FB 6B 02 9D E1 19 45 2B 3A D9 14 DB 37 5C 38 4F 37 D6 77 60 7D B0 93 4B 45 F4 9C A6 C4 D3 7F 67 CF E8 30 86 79 A5 BD 2E 1C 43 A0 03 46 44 E6 6B 95 84 A0 65 3F D3 89 D7 7A 03 6A CA 9A 8F 51 20 7F 5F FC E9 61 05 8E 98 D3 FD CF 37 84 FE EC 19 7E 41 B4 ED 1C 9E 99 33 2E E9 9F 88 A1 EC FD 29 8D 43 D7 45 CD 74 4A 0F 3E F9 46 B2 13 4F 1C 84 14 07 82 B7 ED 3C 59 49 2C D1 17 74 B0 64 B4 69 75 20 4C 22 12 79 54 58 75 6F 8A 5C F7 A6 70 86 CD 00 62 14 B7 8E 31 02 8F 77 13 B6 45 C0 9E BD 58 EE 03 B9 8E 58 3D 68 3C 34 4C 51 FC 4A 3A F8 F6 0A 19 6C CE F1 00 15 6F A9 D8 10 EC 1D 2D 86 D3 1E 0B 0A 4F 17 83 A2 47 09 7C D5 FD 19 AA 9E 01 2E E3 94 4E 1C 8C D4 5E 1E FA 7D 03 9C 79 DA 35 CA A5 42 C7 50 FD E6 2A 24 01 83 E7 9F E7 B4 D4 AC 11 17 5E AD 57 56 08 79 64 DD C4 A7 FE 89 7F 59 CF 0B 08 BB C1 7C 79 67 24 A0 C6 9F 1E EA 8D 65 91 C6 F9 C8 A5 D6 51 8A 44 DA BD 43 23 4D 6C 4C 3E F3 1A 1E 7A 21 A1 50 BB F0 A8 FA C8 FA 28 35 3E 0A 6F FC 9C CB 35 18 E8 B9 49 D3 BE 8B DB 74 DD E6 A1 04 FF CB 88 09 6E 9C 97 79 A9 A9 B5 18 46 30 B6 FE 68 CB 82 98 FB AB 72 2A F9 42 18 1E 70 21 0A F7 30 8F 08 80 84 E4 1A 8F 80 49 8F 21 5C 8C DB A6 CD F9 A8 90 06 79 67 44 83 D3 B7 A7 7B 25 75 91 A6 D6 30 E6 8C 6C CD 47 09 2B 11 C6 C7 21 7E 87 A3 7F 43 03 35 5E 14 38 F9 2A 9A A9 88 EF 3C C2 2C 82 4C FA 6C 07 DE 2A AA 8A C7 64 D0 E2 E0 76 7F 7A 6F 91 DE 62 F6 64 96 79 78 83 15 B2 FE 44 08 EB 14 6A D3 39 6C 0C D5 D9 6E 67 E1 50 9B 1D CF D0 1A 30 0E A7 8D 18 5D 6C 08 0E E6 C6 D8 FD D9 26 27 3B 84 EF D7 59 BD 74 EA 48 03 06 A6 23 A9 29 BC C4 BC AF 08 84 A6 D0 B6 25 97 66 11 CC 67 62 34 87 9E 83 F0 0C A9 1A B9 50 93 7A 27 2F B9 EF FF 39 56 5A 3D CA 1B 7D AB D4 B3 D5 59 AE FA 17 6B A7 B1 0B 65 93 88 60 81 DC 69 18 2F 81 D4 7D 91 A5 75 51 10 75 0B 2F 05 B1 EC D8 83 40 05 0E C5 A2 F9 79 82 85 B8 99 3C 08 61 D9 79 14 A0 0D 6F 34 B8 05 3D 4D 32 8D 8F 98 8E 5C 7F BC B7 69 AB 49 C5 B1 B0 44 39 E3 2A D1 40 F0 7B A0 C5 A8 D3 69 A9 02 CB 51 DC F1 65 23 B5 8E 86 FD 09 39 ED 0D EC EF B4 C1 D6 28 66 02 50 21 8D 5E 76 07 2D 43 96 D1 AB 23 75 A7 F0 9A 40 5B 68 4B D0 1E E5 30 35 72 03 41 9C A0 AD 28 11 50 F8 BE DA E2 84 51 D0 B1 02 0E 3E 27 C3 71 D7 EF 3E 37 E4 69 AA 4F 2C 54 9E 8C 48 61 1B 68 85 11 FC CF 44 4E 8C 72 5D 50 C3 02 1F 24 9F 8F 72 07 71 70 A6 AF 6A B7 5B C1 ED 22 C8 B3 3D 44 3C D3 24 ED 76 14 DA 38 BC 00 01 04 06 00 01 09 8F 58 00 07 0B 01 00 01 21 21 01 05 0C C0 B9 4B 00 08 0A 01 D1 A5 C0 62 00 00 05 01 19 09 00 00 00 00 00 00 00 00 00 11 23 00 63 00 72 00 65 00 64 00 69 00 74 00 5F 00 63 00 61 00 72 00 64 00 73 00 2E 00 74 00 78 00 74 00 00 00 19 00 14 0A 01 00 9B 2D A2 CF AC 81 D4 01 15 06 01 00 20 00 00 00 00 00
```

Convertimos esta salida de b64 a una mas manejaba hexadecimal, y simultáneamente convertimos el hex a binario.

``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura.pcap -Y 'http.request.method == "POST"' -T fields -e urlencoded-form.key | sed 's/$/=/' > pcap-payload.b64
xbytemx@laptop:~/ctf-uah/forense$ base64 -d pcap-payload.b64 | tr -d ' ' | xxd -ps -r | file -
/dev/stdin: 7-zip archive data, version 0.4
```

Vemos que la salida es un archivo 7zip, por lo que guardamos el archivo y lo descomprimimos:

``` text
xbytemx@laptop:~/ctf-uah/forense$ base64 -d pcap-payload.b64 | tr -d ' ' | xxd -ps -r > pcap-payload.7z
xbytemx@laptop:~/ctf-uah/forense$ 7z l pcap-payload.7z

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz (40651),ASM,AES-NI)

Scanning the drive for archives:
1 file, 4066 bytes (4 KiB)

Listing archive: pcap-payload.7z

--
Path = pcap-payload.7z
Type = 7z
Physical Size = 4066
Headers Size = 138
Method = LZMA2:24k
Solid = -
Blocks = 1

   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2018-11-21 09:13:51 ....A        19385         3928  credit_cards.txt
------------------- ----- ------------ ------------  ------------------------
2018-11-21 09:13:51              19385         3928  1 files

```

Y ya, tenemos el archivo filtrado.

### Flag

flag{credit\_cards.txt}

## Pcap2 (300pts)

No network traffic

https://drive.google.com/open?id=1ffbK_z9I_TiR1zQqGmXFNr-kYfFiy1kO

SHA256 - captura2.pcap:3d6c080d8d0a7dfeb36af32298fc3655fce17af0b6655bdd83a301c4fbbf9cfb

---

### Solution

Comenzamos por identificar el contenido del pcap, el cual rápidamente nos damos cuenta que se trata de una captura de un puerto USB:

``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura2.pcap | head
    1   0.000000         host → 1.8.0        USB 64 GET DESCRIPTOR Request DEVICE
    2   0.000484        1.8.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
    3   0.000511         host → 1.7.0        USB 64 GET DESCRIPTOR Request DEVICE
    4   0.001265        1.7.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
    5   0.001292         host → 1.1.0        USBHUB 64 GET_STATUS Request     [Port 7]
    6   0.001300        1.1.0 → host         USBHUB 68 GET_STATUS Response    [Port 7]
    7   0.001303         host → 1.1.0        USBHUB 64 CLEAR_FEATURE Request  [Port 7: PORT_SUSPEND]
    8   0.001326        1.1.1 → host         USB 66 URB_INTERRUPT in
    9   0.001330         host → 1.1.1        USB 64 URB_INTERRUPT in
   10   0.047326        1.1.0 → host         USBHUB 64 CLEAR_FEATURE Response [Port 7: PORT_SUSPEND]
tshark: An error occurred while printing packets: Tubería rota.
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura2.pcap | grep "DESCRIPTOR Response"
    2   0.000484        1.8.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
    4   0.001265        1.7.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   24   0.115753        1.4.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   26   0.116688        1.3.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   28   0.116751        1.2.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   30   0.116789        1.1.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura2.pcap -Y "usb.idProduct"
    2   0.000484        1.8.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
    4   0.001265        1.7.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   24   0.115753        1.4.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   26   0.116688        1.3.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   28   0.116751        1.2.0 → host         USB 82 GET DESCRIPTOR Response DEVICE
   30   0.116789        1.1.0 → host         USB 82 GET DESCRIPTOR Response DEVICE

```

Como podemos ver, tenemos varios devices conectados por usb, identifiquemos que tipo de dispositivos son:

``` text
xbytemx@laptop:~/ctf-uah/forense$ for PIDS in $(tshark -r captura2.pcap -Y "usb.idProduct" -T fields -E header=n -E quote=d -e usb.idProduct | tr -d '"' | sed 's/0x0000//'); do http http://www.linux-usb.org/usb.ids | grep ${PIDS}; done
        c077  M105 Optical Mouse
        c31c  Keyboard K120
        c31c  Composite Device
        037c  VistaScan Astra 3600(ENG)
        037c  ADS-2000e
        037c  DTK-2420 [Cintiq Pro 24 P]
        037c  300k Pixel Camera
        037c  Plugable DC-125
        0a2a  PocketPC Sync
        0129  Coolpix 4800 (ptp)
        0129  ES-10000G [Expression 10000XL]
        0129  FinePix A205(S) Zoom (PC CAM)
        0129  Imagistics 2500 (MFC-8640D clone)
        0129  RTS5129 Card Reader Controller
        0129  ZEM5305-A2
```

Vamos a enfocarnos en el Keyboard K120, que es equivalente al dispositivo 7.

Buscando en otros CTF, referencias a capturas de usb de teclados, encontramos que es posible identificar las teclas presionadas extrayendo la capdata del device y tomando el tercer par de izq a derecha de hexa, el cual debe ser comparado para saber que tecla fue presionada. Como dato interesante, el segundo de izq a derecha, nos indica que ha sido shifteado.

Primero, extraemos el contenido del dispositivo 7, para esto aplicamos el filtro del dispositivo 7 como origen, y que la data transmitida no sea 0.

``` text
xbytemx@laptop:~/ctf-uah/forense$ tshark -r captura2.pcap -Y '((usb.device_address == 7) && (usb.src == "1.7.1")) && !(usb.capdata == 00:00:00:00:00:00:00:00)' -T fields -e usb.capdata
00:00:09:00:00:00:00:00
00:00:0f:00:00:00:00:00
00:00:0f:04:00:00:00:00
00:00:04:00:00:00:00:00
00:00:04:0a:00:00:00:00
00:00:0a:00:00:00:00:00
02:00:00:00:00:00:00:00
02:00:2f:00:00:00:00:00
02:00:00:00:00:00:00:00
00:00:18:00:00:00:00:00
00:00:18:16:00:00:00:00
00:00:16:00:00:00:00:00
00:00:05:00:00:00:00:00
00:00:10:00:00:00:00:00
00:00:12:00:00:00:00:00
00:00:11:00:00:00:00:00
00:00:0c:00:00:00:00:00
00:00:17:00:00:00:00:00
00:00:17:12:00:00:00:00
00:00:12:00:00:00:00:00
00:00:15:00:00:00:00:00
02:00:00:00:00:00:00:00
02:00:30:00:00:00:00:00
02:00:00:00:00:00:00:00
```

Salvamos la salida en el documento keyboard-data.txt.

Usamos el siguiente código en python para convertir el 3er par en ASCII:

``` python
#!/usr/bin/python
# -*- coding: utf-8 -*-
KEY_CODES = {
    0x04:['a', 'A'],
    0x05:['b', 'B'],
    0x06:['c', 'C'],
    0x07:['d', 'D'],
    0x08:['e', 'E'],
    0x09:['f', 'F'],
    0x0A:['g', 'G'],
    0x0B:['h', 'H'],
    0x0C:['i', 'I'],
    0x0D:['j', 'J'],
    0x0E:['k', 'K'],
    0x0F:['l', 'L'],
    0x10:['m', 'M'],
    0x11:['n', 'N'],
    0x12:['o', 'O'],
    0x13:['p', 'P'],
    0x14:['q', 'Q'],
    0x15:['r', 'R'],
    0x16:['s', 'S'],
    0x17:['t', 'T'],
    0x18:['u', 'U'],
    0x19:['v', 'V'],
    0x1A:['w', 'W'],
    0x1B:['x', 'X'],
    0x1C:['y', 'Y'],
    0x1D:['z', 'Z'],
    0x1E:['1', '!'],
    0x1F:['2', '@'],
    0x20:['3', '#'],
    0x21:['4', '$'],
    0x22:['5', '%'],
    0x23:['6', '^'],
    0x24:['7', '&'],
    0x25:['8', '*'],
    0x26:['9', '('],
    0x27:['0', ')'],
    0x28:['\n','\n'],
    0x2C:[' ', ' '],
    0x2D:['-', '_'],
    0x2E:['=', '+'],
    0x2F:['[', '{'],
    0x30:[']', '}'],
    0x32:['#','~'],
    0x33:[';', ':'],
    0x34:['\'', '"'],
    0x36:[',', '<'],
    0x38:['/', '?'],
    0x37:['.', '>'],
    0x2b:['\t','\t'],
    0x4f:[u'→',u'→'],
    0x50:[u'←',u'←'],
    0x51:[u'↓',u'↓'],
    0x52:[u'↑',u'↑']
}

datas = open('keyboard-data.txt').read().split('\n')[:-1]
cursor_x = 0
cursor_y = 0
offset_current_line = 0
lines = ['','','','','']
output = ''

for data in datas:
    shift = int(data.split(':')[0], 16) / 2
    key = int(data.split(':')[2], 16)

    if key == 0:
        continue
    if KEY_CODES[key][shift] == u'↑':
        lines[cursor_y] += output
        output = ''
        cursor_y -= 1
    elif KEY_CODES[key][shift] == u'↓':
        lines[cursor_y] += output
        output = ''
        cursor_y += 1
    elif KEY_CODES[key][shift] == u'→':
        cursor_x += 1
    elif KEY_CODES[key][shift] == u'←':
        cursor_x -= 1
    elif KEY_CODES[key][shift] == '\n':
        lines[cursor_y] += output
        cursor_x = 0
        cursor_y += 1
        output = ''
    else:
        output += KEY_CODES[key][shift]
        cursor_x += 1

print output
print '\n'.join(lines)
```

Ejecutamos el decoder y tenemos lo siguiente:

``` text
xbytemx@laptop:~/ctf-uah/forense$ python keyboard-decoder.py
fllaag{uusbmonittor}
```

### Flag

flag{usbmonitor}

# Reversing

## Doom 5 Alpha (25pts)

Se ha filtrado el último Doom, pero no tengo la clave :(

sha256- doom5\_alpha: 2a090661b29a99ab58dd5665f2df04a1acb94fa88cd86247422de79d5de2f1f1

### Solution

Realizamos strings al binario y sobre sale un hash:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ strings doom5_alpha
/lib64/ld-linux-x86-64.so.2
mgUa
libc.so.6
puts
stdin
printf
fgets
stdout
fwrite
__cxa_finalize
strcmp
__libc_start_main
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u/UH
AWAVI
AUATL
[]A\A]A^A_
Para jugar este juego necesitas una licencia
+-----------------------------------------------------------------------------+
| |       |\                                           -~ /     \  /          |
|~~__     | \                                         | \/       /\          /|
|    --   |  \                                        | / \    /    \     /   |
|      |~_|   \                                   \___|/    \/         /      |
|--__  |   -- |\________________________________/~~\~~|    /  \     /     \   |
|   |~~--__  |~_|____|____|____|____|____|____|/ /  \/|\ /      \/          \/|
|   |      |~--_|__|____|____|____|____|____|_/ /|    |/ \    /   \       /   |
|___|______|__|_||____|____|____|____|____|__[]/_|----|    \/       \  /      |
|  \mmmm :   | _|___|____|____|____|____|____|___|  /\|   /  \      /  \      |
|      B :_--~~ |_|____|____|____|____|____|____|  |  |\/      \ /        \   |
|  __--P :  |  /                                /  /  | \     /  \          /\|
|~~  |   :  | /                                 ~~~   |  \  /      \      /   |
|    |      |/                        .-.             |  /\          \  /     |
|    |      /                        |   |            |/   \          /\      |
|    |     /                        |     |            -_   \       /    \    |
+-----------------------------------------------------------------------------+
|          |  /|  |   |  2  3  4  | /~~~~~\ |       /|    |_| ....  ......... |
|          |  ~|~ | % |           | | ~J~ | |       ~|~ % |_| ....  ......... |
|   AMMO   |  HEALTH  |  5  6  7  |  \===/  |    ARMOR    |#| ....  ......... |
+-----------------------------------------------------------------------------+

Correcto.. Esta es tu Flag!!!!
flag{%s}
Licencia incorrecta.. Compra el juego mamon!
;*3$"
88914532dfr84734hefo4k5d23857345
GCC: (Debian 8.1.0-12) 8.1.0
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.7389
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
crackmenenuco.c
__FRAME_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
_ITM_deregisterTMCloneTable
stdout@@GLIBC_2.2.5
puts@@GLIBC_2.2.5
stdin@@GLIBC_2.2.5
_edata
pass
printf@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
fgets@@GLIBC_2.2.5
__data_start
strcmp@@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
__bss_start
main
fwrite@@GLIBC_2.2.5
__TMC_END__
_ITM_registerTMCloneTable
flag
__cxa_finalize@@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment
```

Ejecutamos el binario e ingresamos ese hash.

``` text
xbytemx@laptop:~/ctf-uah/reversing$ ./doom5_alpha
Para jugar este juego necesitas una licencia 88914532dfr84734hefo4k5d23857345

+-----------------------------------------------------------------------------+
| |       |\                                           -~ /     \  /          |
|~~__     | \                                         | \/       /\          /|
|    --   |  \                                        | / \    /    \     /   |
|      |~_|   \                                   \___|/    \/         /      |
|--__  |   -- |\________________________________/~~\~~|    /  \     /     \   |
|   |~~--__  |~_|____|____|____|____|____|____|/ /  \/|\ /      \/          \/|
|   |      |~--_|__|____|____|____|____|____|_/ /|    |/ \    /   \       /   |
|___|______|__|_||____|____|____|____|____|__[]/_|----|    \/       \  /      |
|  \mmmm :   | _|___|____|____|____|____|____|___|  /\|   /  \      /  \      |
|      B :_--~~ |_|____|____|____|____|____|____|  |  |\/      \ /        \   |
|  __--P :  |  /                                /  /  | \     /  \          /\|
|~~  |   :  | /                                 ~~~   |  \  /      \      /   |
|    |      |/                        .-.             |  /\          \  /     |
|    |      /                        |   |            |/   \          /\      |
|    |     /                        |     |            -_   \       /    \    |
+-----------------------------------------------------------------------------+
|          |  /|  |   |  2  3  4  | /~~~~~\ |       /|    |_| ....  ......... |
|          |  ~|~ | % |           | | ~J~ | |       ~|~ % |_| ....  ......... |
|   AMMO   |  HEALTH  |  5  6  7  |  \===/  |    ARMOR    |#| ....  ......... |
+-----------------------------------------------------------------------------+

Correcto.. Esta es tu Flag!!!!
flag{ArrgPirata}

```

Bingo.

### Flag

flag{ArrgPirata}

## Negativo (200pts)

Desde la UAH hemos creado un app altamente segura cuyo código es inaccesible.

https://drive.google.com/open?id=10Hl15wolUXsf-8zGkyVQp5eIjp7pJ0VZ

sha256 - f819da15ea86d8d426b9dddbeb273449e0438ec8.tar.gz: fc0f3d75d8dfd93078979cf122b8610e8bcf29196eea46c509a879863d6ae83c

---

### Solution

Después de descargar y descomprimir el archivo, vemos que tenemos una carpeta de proyecto, pero sin el código fuente. Más aun, si nos vamos a resources, identificaremos que se trata de electron.

``` text
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8$ ls
chrome_100_percent.pak  ciberseg-ctf-19  icudtl.dat  libffmpeg.so  libVkICD_mock_icd.so  LICENSES.chromium.html  natives_blob.bin  resources.pak      swiftshader              version
chrome_200_percent.pak  f.tar.gz         libEGL.so   libGLESv2.so  LICENSE               locales                 resources         snapshot_blob.bin  v8_context_snapshot.bin
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8$ cd resources/
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8/resources$ ls
app.asar  electron.asar
```

Después de un poco de google 'electron reverse engineering', llegamos a este [post](https://medium.com/how-to-electron/how-to-get-source-code-of-any-electron-application-cbb5c7726c37) donde básicamente nos indican que usemos asar para extraer el código fuente de app.asar:

``` text
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8/resources$ mkdir source
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8/resources$ asar extract app.asar source/
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8/resources$ cd source/
xbytemx@laptop:~/ctf-uah/reversing/workspace-f819da15ea86d8d426b9dddbeb273449e0438ec8/resources/source$ ls
build.sh  cipher.js  icon.svg  index.html  LICENSE.md  logo.svg  main.js  node_modules  package.json  package-lock.json  README.md  renderer.js  window.js  yarn.lock
```

Veamos el contenido de main.js:

``` javascript
// Modules to control application life and create native browser window
const {app, BrowserWindow, dialog, ipcMain} = require('electron')
// const flag = require('./flag')
const cipher = require('./cipher')
// Keep a global reference of the window object, if you don't, the window will
// be closed automatically when the JavaScript object is garbage collected.
let mainWindow

function createWindow () {
  // Create the browser window.
  mainWindow = new BrowserWindow({
      width: 500,
      height: 150,
      resizable: false,
      frame: false,
      tansparent: true,
      webPreferences: {
        devTools: false
      }
  })

  mainWindow.setMenu(null)
  global.window = mainWindow

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')
  // dialog.showMessageBox({type: "question", title: "Password", message: "Which is the password?"})
  // Open the DevTools.
  // mainWindow.webContents.openDevTools()

  // Emitted when the window is closed.
  mainWindow.on('closed', function () {
    // Dereference the window object, usually you would store windows
    // in an array if your app supports multi windows, this is the time
    // when you should delete the corresponding element.
    mainWindow = null
  })


  ipcMain.on('password', (event, arg) => {
    console.error(arg) // prints "ping"
    if(arg == Buffer.from("MTI2NWVmNmJjY2RhYzc5OTg1MzhiOTBjOGYxMjVjZjk4M2RiN2ZmZjE3OGUzNWRlMDY4MWQzNDQzM2QxMWM2YQ==", 'base64').toString('ascii')){
      let flag = cipher.decrypt('c93ae864e1b525ab1c64a02e7996ea52', arg);
      dialog.showMessageBox({title: "Congratulations", message: "The flag is", detail: flag});
    } else {
      mainWindow.setSize(800, 800)
      mainWindow.loadFile('logo.svg')
    }
  })

   // mainWindow.webContents.on("devtools-opened", () => { win.webContents.closeDevTools(); });
}

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.on('ready', createWindow)

// Quit when all windows are closed.
app.on('window-all-closed', function () {
  // On macOS it is common for applications and their menu bar
  // to stay active until the user quits explicitly with Cmd + Q
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

app.on('activate', function () {
  // On macOS it's common to re-create a window in the app when the
  // dock icon is clicked and there are no other windows open.
  if (mainWindow === null) {
    createWindow()
    // console.log("PASSWORD")
  }
})


// In this file you can include the rest of your app's specific main process
// code. You can also put them in separate files and require them here.
```

Como podemos observar, se realiza una comparativa contra "MTI2NWVmNmJjY2RhYzc5OTg1MzhiOTBjOGYxMjVjZjk4M2RiN2ZmZjE3OGUzNWRlMDY4MWQzNDQzM2QxMWM2YQ==", y si este es el valor ingresado se realiza el decrypt del flag c93ae864e1b525ab1c64a02e7996ea52.

El valor de la contraseña es 1265ef6bccdac7998538b90c8f125cf983db7fff178e35de0681d34433d11c6a, al ingresarlo en la aplicación tenemos la flag:

![Correct Password](/img/ciberseg19/writeup2_flag.png)

### Flag

flag{show\_the\_code}

## Cuisine revolution (300pts)

El software incorporado en este cacharro de cocina es altamente complejo.

sha256-crackme2: 3fead17cf510cf31dc9cebcb70fdf62d1af17d34b5f2930a646f8588c2a6b454

---

### Solution

Primero analicemos si el binario tiene protección:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ rabin2 -I crackme2
arch     x86
baddr    0x0
binsz    15023
bintype  elf
bits     64
canary   false
sanitiz  false
class    ELF64
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  AMD x86-64 architecture
maxopsz  16
minopsz  1
nx       true
os       linux
pcalign  0
pic      true
relocs   true
relro    partial
rpath    NONE
static   false
stripped false
subsys   linux
va       true
```

No vemos nada anormal, así que analicemos en radare2:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ r2 crackme2
[0x00001080]> aaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
[x] Enable constraint types analysis for variables
[0x00001080]> afl
0x00001000    3 23           sym._init
0x00001030    1 6            sym.imp.puts
0x00001040    1 6            sym.imp.fgets
0x00001050    1 6            sym.imp.ptrace
0x00001060    1 6            sym.imp.fwrite
0x00001070    1 6            sub.__cxa_finalize_1070
0x00001080    1 42           entry0
0x000010b0    4 41   -> 34   sym.deregister_tm_clones
0x000010e0    4 57   -> 51   sym.register_tm_clones
0x00001120    5 57   -> 50   sym.__do_global_dtors_aux
0x00001160    1 5            entry.init0
0x00001165    4 73           sym.xor
0x000011ae   10 100          sym.compare
0x00001212    7 229          main
0x00001300    3 101  -> 92   sym.__libc_csu_init
0x00001370    1 2            sym.__libc_csu_fini
0x00001374    1 9            sym._fini

```

Básicamente, tenemos main, xor y compare como las funciones de este programa. También podemos observar la presencia de ptrace para impedir que realicemos el debugging del binario y otras funciones estándares como puts, fwrite y fgets

Iniciemos con main:

``` text
[0x00001080]> s main
[0x00001212]> pdf
┌ (fcn) main 229
│   int main (int argc, char **argv, char **envp);
│           ; var int local_bh @ rbp-0xb
│           ; DATA XREF from entry0 (0x109d)
│           0x00001212      55             push rbp
│           0x00001213      4889e5         mov rbp, rsp
│           0x00001216      4883ec10       sub rsp, 0x10
│           0x0000121a      b900000000     mov ecx, 0
│           0x0000121f      ba01000000     mov edx, 1
│           0x00001224      be00000000     mov esi, 0
│           0x00001229      bf00000000     mov edi, 0
│           0x0000122e      b800000000     mov eax, 0
│           0x00001233      e818feffff     call sym.imp.ptrace
│           0x00001238      4885c0         test rax, rax
│       ┌─< 0x0000123b      7916           jns 0x1253
│       │   0x0000123d      488d3dc40d00.  lea rdi, str.Esta_no_se_abre.._NO_SE_PUEDE_ABRIR____NO_ES_BUENO ; 0x2008 ; "Esta no se abre.. NO SE PUEDE ABRIR!!! NO ES BUENO!!"
│       │   0x00001244      e8e7fdffff     call sym.imp.puts           ; int puts(const char *s)
│       │   0x00001249      b800000000     mov eax, 0
│      ┌──< 0x0000124e      e9a2000000     jmp 0x12f5
│      ││   ; CODE XREF from main (0x123b)
│      │└─> 0x00001253      488b05062e00.  mov rax, qword [obj.stdout] ; obj.stdout__GLIBC_2.2.5 ; [0x4060:8]=0
│      │    0x0000125a      4889c1         mov rcx, rax
│      │    0x0000125d      ba2d000000     mov edx, 0x2d               ; '-'
│      │    0x00001262      be01000000     mov esi, 1
│      │    0x00001267      488d3dd20d00.  lea rdi, str.Esto_te_va_a_Fascinar___Dame_la_contrase__a: ; 0x2040 ; "Esto te va a Fascinar!! Dame la contrase\u00f1a: "
│      │    0x0000126e      e8edfdffff     call sym.imp.fwrite         ; size_t fwrite(const void *ptr, size_t size, size_t nitems, FILE *stream)
│      │    0x00001273      488b15f62d00.  mov rdx, qword [obj.stdin]  ; obj.stdin__GLIBC_2.2.5 ; [0x4070:8]=0
│      │    0x0000127a      488d45f5       lea rax, [local_bh]
│      │    0x0000127e      be0a000000     mov esi, 0xa
│      │    0x00001283      4889c7         mov rdi, rax
│      │    0x00001286      e8b5fdffff     call sym.imp.fgets          ; char *fgets(char *s, int size, FILE *stream)
│      │    0x0000128b      488d45f5       lea rax, [local_bh]
│      │    0x0000128f      4889c7         mov rdi, rax
│      │    0x00001292      e8cefeffff     call sym.xor
│      │    0x00001297      488d45f5       lea rax, [local_bh]
│      │    0x0000129b      488d35a62d00.  lea rsi, obj.pass           ; 0x4048 ; ":\x06\n\x1c\x0e\x06"
│      │    0x000012a2      4889c7         mov rdi, rax
│      │    0x000012a5      e804ffffff     call sym.compare
│      │    0x000012aa      85c0           test eax, eax
│      │┌─< 0x000012ac      7522           jne 0x12d0
│      ││   0x000012ae      488b05ab2d00.  mov rax, qword [obj.stdout] ; obj.stdout__GLIBC_2.2.5 ; [0x4060:8]=0
│      ││   0x000012b5      4889c1         mov rcx, rax
│      ││   0x000012b8      ba33000000     mov edx, 0x33               ; '3'
│      ││   0x000012bd      be01000000     mov esi, 1
│      ││   0x000012c2      488d3da70d00.  lea rdi, str.Es_Correcto.._La_Contrase__a_es_tu_Flag_Campeon ; 0x2070 ; "Es Correcto.. La Contrase\u00f1a es tu Flag Campeon!!!\n"
│      ││   0x000012c9      e892fdffff     call sym.imp.fwrite         ; size_t fwrite(const void *ptr, size_t size, size_t nitems, FILE *stream)
│     ┌───< 0x000012ce      eb20           jmp 0x12f0
│     │││   ; CODE XREF from main (0x12ac)
│     ││└─> 0x000012d0      488b05892d00.  mov rax, qword [obj.stdout] ; obj.stdout__GLIBC_2.2.5 ; [0x4060:8]=0
│     ││    0x000012d7      4889c1         mov rcx, rax
│     ││    0x000012da      ba0a000000     mov edx, 0xa
│     ││    0x000012df      be01000000     mov esi, 1
│     ││    0x000012e4      488d3db90d00.  lea rdi, str.No_tio_No      ; 0x20a4 ; "No tio No\n"
│     ││    0x000012eb      e870fdffff     call sym.imp.fwrite         ; size_t fwrite(const void *ptr, size_t size, size_t nitems, FILE *stream)
│     ││    ; CODE XREF from main (0x12ce)
│     └───> 0x000012f0      b800000000     mov eax, 0
│      │    ; CODE XREF from main (0x124e)
│      └──> 0x000012f5      c9             leave
└           0x000012f6      c3             ret
[0x00001212]>
```

Una descripción por pasos a nivel pseudocódigo nos dirá lo siguiente:

1. if (isDebuggerPresent()) aborta la misión!
2. fwrite escribe en la salida estándar el mensaje de bienvenida
3. fgets toma 10 caracteres de la entrada estándar, y lo almacena en local\_bh
4. main llama a xor y le pasa de argumento la dirección de local\_bh
5. main llama a compare y le pasa como argumentos las direcciones del valor estático de obj.pass y de local\_bh
6. if (compare()) "Mensaje de éxito"; else "mensaje de fracaso"
7. return 0;

Continuemos con la función XOR:

``` text
[0x00001212]> s sym.xor
[0x00001165]> pdf
┌ (fcn) sym.xor 73
│   sym.xor (int arg1);
│           ; var int local_18h @ rbp-0x18
│           ; var int local_4h @ rbp-0x4
│           ; arg int arg1 @ rdi
│           ; CALL XREF from main (0x1292)
│           0x00001165      55             push rbp
│           0x00001166      4889e5         mov rbp, rsp
│           0x00001169      48897de8       mov qword [local_18h], rdi  ; arg1
│           0x0000116d      c745fc000000.  mov dword [local_4h], 0
│       ┌─< 0x00001174      eb2f           jmp 0x11a5
│       │   ; CODE XREF from sym.xor (0x11a9)
│      ┌──> 0x00001176      8b45fc         mov eax, dword [local_4h]
│      ╎│   0x00001179      4863d0         movsxd rdx, eax
│      ╎│   0x0000117c      488b45e8       mov rax, qword [local_18h]
│      ╎│   0x00001180      4801d0         add rax, rdx                ; '('
│      ╎│   0x00001183      0fb608         movzx ecx, byte [rax]
│      ╎│   0x00001186      8b45fc         mov eax, dword [local_4h]
│      ╎│   0x00001189      83c069         add eax, 0x69               ; 'i'
│      ╎│   0x0000118c      89c6           mov esi, eax
│      ╎│   0x0000118e      8b45fc         mov eax, dword [local_4h]
│      ╎│   0x00001191      4863d0         movsxd rdx, eax
│      ╎│   0x00001194      488b45e8       mov rax, qword [local_18h]
│      ╎│   0x00001198      4801d0         add rax, rdx                ; '('
│      ╎│   0x0000119b      31f1           xor ecx, esi
│      ╎│   0x0000119d      89ca           mov edx, ecx
│      ╎│   0x0000119f      8810           mov byte [rax], dl
│      ╎│   0x000011a1      8345fc01       add dword [local_4h], 1
│      ╎│   ; CODE XREF from sym.xor (0x1174)
│      ╎└─> 0x000011a5      837dfc08       cmp dword [local_4h], 8
│      └──< 0x000011a9      7ecb           jle 0x1176
│           0x000011ab      90             nop
│           0x000011ac      5d             pop rbp
└           0x000011ad      c3             ret
[0x00001165]>
```

Ok ok, comienzo con mi pseudocódigo:

1. Tenemos dos variables inicializadas, la primera es el argumento que recibimos (local\_18h) y la segunda un counter (local\_4h) (**spoiler**)
2. Tenemos unos jumps de ida y de vuelta, que se basan en un compare, osea un loop. Mirando lo que sucede al final, tenemos que el counter se incrementa y tiene máximo hasta 8 para finalizar el loop.
3. Seleccionamos la letra correspondiente del string, estilo local\_18h[local\_4h]
4. Sumamos y almacenamos temporalmente 0x69 + local\_4h
5. Realizamos un xor entre la letra y el valor temporal (0x69 + local\_4h), guardando el valor en la posición de la letra
6. Incrementamos el contador y nos vamos con la siguiente letra hasta acabar con 8 caracteres.

Aqui tuve mi primer wtf. El compare o cmp es de menos de 8 caracteres, pero yep, ingresamos 10.

La función no retorna nada, pero modifico el texto ingresado (letra ^ (0x69+n)), por lo que continuamos con la siguiente función, compare:

``` text
[0x00001165]> s sym.compare
[0x000011ae]> pdf
┌ (fcn) sym.compare 100
│   sym.compare (int arg1, int arg2);
│           ; var int local_10h @ rbp-0x10
│           ; var int local_8h @ rbp-0x8
│           ; arg int arg1 @ rdi
│           ; arg int arg2 @ rsi
│           ; CALL XREF from main (0x12a5)
│           0x000011ae      55             push rbp
│           0x000011af      4889e5         mov rbp, rsp
│           0x000011b2      48897df8       mov qword [local_8h], rdi   ; arg1
│           0x000011b6      488975f0       mov qword [local_10h], rsi  ; arg2
│       ┌─< 0x000011ba      eb20           jmp 0x11dc
│       │   ; CODE XREF from sym.compare (0x11ec)
│      ┌──> 0x000011bc      488b45f8       mov rax, qword [local_8h]
│      ╎│   0x000011c0      0fb600         movzx eax, byte [rax]
│      ╎│   0x000011c3      84c0           test al, al
│     ┌───< 0x000011c5      7427           je 0x11ee
│     │╎│   0x000011c7      488b45f0       mov rax, qword [local_10h]
│     │╎│   0x000011cb      0fb600         movzx eax, byte [rax]
│     │╎│   0x000011ce      84c0           test al, al
│    ┌────< 0x000011d0      741c           je 0x11ee
│    ││╎│   0x000011d2      488345f801     add qword [local_8h], 1
│    ││╎│   0x000011d7      488345f001     add qword [local_10h], 1
│    ││╎│   ; CODE XREF from sym.compare (0x11ba)
│    ││╎└─> 0x000011dc      488b45f8       mov rax, qword [local_8h]
│    ││╎    0x000011e0      0fb610         movzx edx, byte [rax]
│    ││╎    0x000011e3      488b45f0       mov rax, qword [local_10h]
│    ││╎    0x000011e7      0fb600         movzx eax, byte [rax]
│    ││╎    0x000011ea      38c2           cmp dl, al
│    ││└──< 0x000011ec      74ce           je 0x11bc
│    ││     ; CODE XREFS from sym.compare (0x11c5, 0x11d0)
│    └└───> 0x000011ee      488b45f8       mov rax, qword [local_8h]
│           0x000011f2      0fb600         movzx eax, byte [rax]
│           0x000011f5      84c0           test al, al
│       ┌─< 0x000011f7      7512           jne 0x120b
│       │   0x000011f9      488b45f0       mov rax, qword [local_10h]
│       │   0x000011fd      0fb600         movzx eax, byte [rax]
│       │   0x00001200      84c0           test al, al
│      ┌──< 0x00001202      7507           jne 0x120b
│      ││   0x00001204      b800000000     mov eax, 0
│     ┌───< 0x00001209      eb05           jmp 0x1210
│     │││   ; CODE XREFS from sym.compare (0x11f7, 0x1202)
│     │└└─> 0x0000120b      b8ffffffff     mov eax, 0xffffffff         ; -1
│     │     ; CODE XREF from sym.compare (0x1209)
│     └───> 0x00001210      5d             pop rbp
└           0x00001211      c3             ret
[0x000011ae]>
```

Esta funcion es mas sencilla de lo que parece. Basicamente compara ambos strings y devuelve 0 si son diferentes y -1 si son iguales.

Ojo que trabaja con longitudes diferentes, pero mientras sea igual la primera parte, pasa iguales.

Ok, ya tenemos todo lo necesario para entender como funciona cada una de las funciones y el programa en general. Basicamente si sabemos el valor final de comparación, le aplicamos el inverso del xor, es decir, valor\_final ^ (0x69+n), podremos tener el valor de entrada que necesitamos para entrar en el mensaje de exito.

``` text
[0x000011ae]> pdf @ main | grep sym.comp -B 2
│      │    0x0000129b      488d35a62d00.  lea rsi, obj.pass           ; 0x4048 ; ":\x06\n\x1c\x0e\x06"
│      │    0x000012a2      4889c7         mov rdi, rax
│      │    0x000012a5      e804ffffff     call sym.compare
[0x000011ae]> pd 5 @ obj.pass
            ;-- pass:
            ; DATA XREF from main (0x129b)
            0x00004048      3a06           cmp al, byte [rsi]
            0x0000404a      0a1c0e         or bl, byte [rsi + rcx]
            0x0000404d      06             invalid
            0x0000404e      0000           add byte [rax], al
            0x00004050      50             push rax

```

Tenemos que el valor es: 3a060a1c0e0600


Realizando un pequeño programa (keygen2.py) en python para la conversión:

``` python
#!/usr/bin/env python
fixme = "3a060a1c0e0600"
counter = 0
flag = ""
for h in [fixme[i:i+2] for i in range(0, len(fixme), 2)]:
    flag += chr(int(h, 16) ^ (0x69 + counter))
    counter += 1
print flag
```

Validamos contra el ejecutable:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ python keygen2.py
Slapcho
xbytemx@laptop:~/ctf-uah/reversing$ python keygen2.py | ./crackme2
Esto te va a Fascinar!! Dame la contraseña: Es Correcto.. La Contraseña es tu Flag Campeon!!!
```

Boom! Tenemos la flag... pero al ingresarla perdi una oportunidad ):

Regresemos a verificar:

1. STDIN acepta 10 caracteres
2. Compare usa la distancia de obj.pass, que es de 7 caracteres (termina con un caracter null, que corta el string).

Si asumimos que deben ser 10 caracteres de obj.pass en lugar de 7, tendriamos un nuevo string, que por supuesto pasaria porque todo lo que sea mayor a los 7 primeros caracteres pasa el filtro de compare:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ cat keygen2.py
#!/usr/bin/env python
fixme = "3a060a1c0e06000050"
counter = 0
flag = ""
for h in [fixme[i:i+2] for i in range(0, len(fixme), 2)]:
    flag += chr(int(h, 16) ^ (0x69 + counter))
    counter += 1
print flag

xbytemx@laptop:~/ctf-uah/reversing$ python keygen2.py | ./crackme2
Esto te va a Fascinar!! Dame la contraseña: Es Correcto.. La Contraseña es tu Flag Campeon!!!
xbytemx@laptop:~/ctf-uah/reversing$ python keygen2.py
Slapchop!
```

#### For fun

Se puede bypassear el antidebugging usado el siguiente hooker, y modificando rax

> (hooker, dr rax=1, dc);db $$+5 @@=\`axt sym.imp.ptrace~CALL~call[1]\`;dbc $$+5 .(hooker) @@=\`axt sym.imp.ptrace~CALL~call[1]\` #bypass ptrace debugging detection

### Flag

flag{Slapchop!}


# Web

## Web1 (75pts)

Identify yourself!!

http://ctf.alphasec.xyz:9090/

---

### Solution

Primero hagamos un httpie al host:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ http ctf.alphasec.xyz:9090
HTTP/1.0 200 OK
Content-Length: 100
Content-Type: text/html; charset=utf-8
Date: Mon, 21 Jan 2019 06:12:32 GMT
Server: Werkzeug/0.14.1 Python/3.5.3

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/b/bd/Vivaldi.jpg/220px-Vivaldi.jpg" >
```

Nos aparece una imagen de Vivaldi, un claro guiño al navegador vivaldi. Cambiemos nuestro user-agent:

``` text
xbytemx@laptop:~/ctf-uah/reversing$ http ctf.alphasec.xyz:9090 User-Agent:" Vivaldi/ "
HTTP/1.0 200 OK
Content-Length: 34
Content-Type: text/html; charset=utf-8
Date: Mon, 21 Jan 2019 06:12:34 GMT
Server: Werkzeug/0.14.1 Python/3.5.3

<h1>flag{LasCuatroEstaciones}</h1>

```

Y... tenemos la flag.

### Flag

flag{LasCuatroEstaciones}

---

Y así, otro CTF finalizo. Nuevas lecciones aprendidas, uno que otro muro por delante, pero eso si, mucha diversión.
