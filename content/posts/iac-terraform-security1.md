---
title: "Recomendaciones de seguridad para Terraform (primera parte)"
date: 2023-12-13T17:51:15-06:00
draft: true
description: "Practicas de seguridad en infrastructura como codigo en Hashicorp Terraform. Primera parte, requerimientos para un aprovisionamiento seguro."
tags: ["iac","terraform","devops"]
categories: ["devsecops","cloudsec"]

---
Que es terraform en mis propias palabras?

> Una herramienta de aprovisionamiento, creada para escribir codigo enfocandonos en el estado objetivo, con una gran capacidad modular para poder construir, reutilizar y compartir codigo, que empuja a todo una comunidad de profesionales y entusiastas

Desde esta definicion, comienzo con esta primera parte sobre "herramienta de aprovisionamiento". Las primeras preguntas que nos debemos hacer son:

- Quien o que ejecuta terraform?
- Como se ejecuta? Localmente, un pipeline, una funcion? 
- Tenemos registro y claridad de las acciones de terraform?

Estas preguntas estan orientadas al inicio, a la seguridad que debemos agregar como REQUERIMIENTO en la arquitectura o definicion operativa. 

Pensemos por un momento en la segregacion de responsabilidades respecto a quien o que ejecuta terraform; tenemos un usuario que crea el codigo, otras identidades que autorizan y validan los cambios (CI/CD y/o otro usuario), y otra identidad que aplica los cambios descritas por el codigo.

Nuestros usuarios que crean el codigo dentro de esta segregacion deben tener  permisos de read/write y acceso al repositorio via organizaciones, grupos o otra estructura que permita agregarlos segun su nivel de responsabilidad o participacion. Esta primera diferenciacion sobre el rol del usuario y su temporalidad es la primera barrera para nuestros repositorios. La autenticacion por SAML nos permite que una vez que un usuario es revocado de la fuente de autenticacion, este no pueda volver a acceder a los recursos a los que tuvo acceso, pero esto significa que hasta que los usuarios son eliminados debemos hacer una revision de quienes tienen acceso a nuestro repositorio? Por supuesto que no. Como una medida de higiene, debemos verificar y darle mantenimiento tanto a repositorios como accesos a estos. Los permisos no son estaticos ni para siempre. 

A que deberian tener acceso los usuarios que crean o modifican el codigo? Unicamente al repositorio que mantienen. No deberian tener acceso al almacenamiento que contiene el tfstate file o a deployear ellos la infraestructura. Historicamente, la persona que mantiene el codigo, tiene acceso a realizar los cambios en los ambientes desplegados, pero es esto un must? La realidad es que no. No necesariamente el usuario que hace cambios debe tener acceso a modificar la infraestructura donde se realizan los cambios, es solo una comodidad que los desarrolladores de IaC ha heredado de roles operativos. El tema del almacenamiento del tfstate file lo veremos mas adelante.

En la autorizacion y validacion del codigo es donde entran muchas herramientas y/o scanners estaticos que se encargan de comprender la "delta" de cambios; el estado destino de como queremos tener nuestra infraestructura sea aplicada despues de la ejecucion terraform. Que pasa con los permisos de la identidad usada para ejecutar terraform? Tiene unicamente los permisos para crear, modificar y destruir aquellos elementos definidos por codigo o tiene mas permisos tipo "*"? Esta parte en terminos operativos es complicado y tedioso, porque si queremos agregarle nuevas funciones a nuestro codigo, tambien tenemos que actualizar el usuario que se usa para aplicar el codigo.

Abramos ahora escenario para la identidad que se encarga de la integracion continua de nuestro codigo. Usualmente en un pipeline tenemos una puerta o stage de validacion del codigo, esta se encarga de tomar el codigo y utilizar herramientas que se encargan de dar formato (terraform fmt -recursive) y de validar la sintaxis del codigo (terraform validate). 
