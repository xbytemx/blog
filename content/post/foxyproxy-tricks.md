---
title: "Foxyproxy Tricks: Proxies by patterns and order"
date: 2019-11-09T08:46:24-06:00
description: "Cuando solo quieres mandar al proxy un scope definido"
tags: ["pentesting", "foxyproxy", "tips"]
categories: ["pentesting"]

---

Esta es una entrada rápida, una referencia a una practica que puede ayudarnos a no perder enfoque en el objetivo y distraernos con "Deje habilitado el proxy?" o "Necesito apagar el proxy para buscar algo en internet".
<!--more-->

Foxyproxy es un plugin de firefox muy popular que nos permite definir la configuración de varios proxies, de manera tal que con darle un click su icono podemos alternar entre diferentes perfiles. Una característica que me llamo la atención cuando lo estuve explorando más a detalle fue la de "Use Enabled Proxies By Patterns and Order":

![Menú de opciones](/img/foxyproxy-tricks/options1.png)

> En mi caso, tengo configurado ya BurpSuite en su configuración por defecto `127.0.0.1:8080`.

Si seleccionamos esta opción, como su nombre lo dice tomara el orden como primer valor de arriba hacia abajo para seleccionar entre los diferentes perfiles de proxy definidos y evaluará en cada perfil los patrones asociados a cada perfil.

Seleccionemos en este caso, patrones para el perfil de BurpSuite:

![Editando la configuración global](/img/foxyproxy-tricks/options2.png)

La ventana de patterns luce de la siguiente manera:

![Patterns](/img/foxyproxy-tricks/patterns1.png)

El scope que estaré manejando para BurpSuite sera el de hackthebox. ¿Cual es el scope tradicionalmente? Direcciones ip 10.10.10.X y dominios \*.htb

Todo lo que no este dentro del scope lo ignorare por lo que únicamente aplicare whitelist y lo demás pasará al proxy Default. El proxy Default tiene habilitado mandarle el trafico al navegador, por lo que la conexión seria de acuerdo a la configuración del navegador.

Seleccionamos "New White", le agregamos el nombre de "domains" y el pattern de "\*.htb":

![Patterns - Wildcard](/img/foxyproxy-tricks/patterns2.png)

Agregaremos ahora una regex para las direcciones IP cambiando únicamente de type "wildcard" a "regex":

![Patterns - RegEx](/img/foxyproxy-tricks/patterns3.png)

La expresión regular solo aceptara direcciones ip validas y lo enviara a Burp.

Una vez configurado, guardamos los cambios con el boton "Save" y probamos nuestra configuración en abriendo una nueva pestaña. Como se puede ver por la siguiente imagen, aparece "Defa" en la parte superior, indicando que el perfil Default esta habilitado.

![Nueva Pestaña](/img/foxyproxy-tricks/test1.png)

> Dato cultural; ¿porqué aparece Default si no he abierto ninguna pagina? Porque detect portal esta habilitado en mi navegador tratando de identificar un portal cautivo.

Ahora vemos en acción esta configuración:

![Foxyproxy en acción](/img/foxyproxy-tricks/foxyproxy1.gif)

Si tenemos dudas sobre las regex o el wildcard que estamos ingresando, siempre podemos usar el "pattern tester" que tenemos a lado derecho y validar varios casos.

---

Eso es todo por el momento, espero que te haya gustado.
