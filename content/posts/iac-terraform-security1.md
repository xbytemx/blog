---
title: "Recomendaciones de seguridad para Terraform (primera parte)"
date: 2023-12-13T17:51:15-06:00
draft: true
description: "Practicas de seguridad en infrastructura como codigo con enfoque en Hashicorp Terraform. Primera parte, requerimientos para un aprovisionamiento seguro."
tags: ["iac","terraform","devops"]
categories: ["devsecops","cloudsec"]

---
Empezamos esta entrada con una definicion en mis propias palabras de lo que es terraform:

> Una herramienta de aprovisionamiento, creada para escribir codigo enfocandonos en el estado objetivo, con una gran capacidad modular para poder construir, reutilizar y compartir codigo, que empuja a todo una comunidad de profesionales y entusiastas

Desde esta definicion, comienzo con la primera pregunta sobre "herramienta de aprovisionamiento". Quien o que ejecuta terraform? Como se ejecuta? Tenemos registro y claridad de las acciones de terraform? Estas preguntas estan orientadas al inicio, a la seguridad que debemos agregar como REQUERIMIENTO en la arquitectura o definicion operativa. 

Pensemos por un momento en uno de los principios de segregacion de responsabilidades; tenemos un usuario que crea el codigo, otras identidades que autorizan los cambios (CI/CD y/o otro usuario), y otra identidad que despliega el codigo. Nuestros usuarios que crean el codigo por permisos read/write tienen acceso al repositorio via organizaciones, grupos o otra estructura que permita agregarlos segun los permisos del usuario.

A que deberian tener acceso los usuarios que crean o modifican el codigo? Unicamente al repositorio que mantienen. No deberian tener acceso al almacenamiento que contiene el tfstate file o a deployear ellos la infraestructura. Historicamente, la persona que mantiene el codigo, tiene acceso a realizar los cambios, pero es esto un must? La realidad es que no. No necesariamente el usuario que hace cambios debe tener acceso a la infraestructura donde se realizan los cambios, es solo una comodidad que se ha heredado de roles de devops. El tema del almacenamiento del tfstate file lo veremos mas adelante.

La segunda parte sobre la autorizacion, es donde entran muchas herramientas o scanners estaticos que se encargan de validar lo que indica el codigo; el estado destino de como queremos tener nuestra infraestructura este como resultado de la herramienta, pero que pasa con los permisos de la identidad usada para ejecutar terraform, tiene unicamente los permisos para crear, modificar y destruir aquellos elementos definidos por codigo o tiene mas permisos tipo "*"? Esta parte en terminos operativos es complicado y tedioso, porque si queremos agregarle nuevas funciones a nuestro codigo, tambien tenemos que actualizar el usuario que se usa para deployear.

Esta parte se refiere a la identidad que se encarga de hacer el continous integration. Usualmente en un pipeline tenemos una puerta o stage de validacion del codigo, esta se encarga de tomar el codigo y utilizar herramientas que se encargan de dar formato (terraform fmt -recursive) y de validar la sintaxis del codigo (terraform validate). 
