---
title: "Recomendaciones de seguridad para Terraform (primera parte)"
date: 2023-12-13T17:51:15-06:00
draft: true
description: "Practicas de seguridad en infrastructura como codigo en Hashicorp Terraform. Primera parte, requerimientos para un aprovisionamiento seguro."
tags: ["iac","terraform","devops"]
categories: ["devsecops","cloudsec"]

---
# Mi definicion

Todo parte siempre desde una definicion, asi que, que es terraform (el producto de Hashicorp) en mis propias palabras?

> Una herramienta de aprovisionamiento, creada para escribir codigo enfocandonos en el estado objetivo, con una gran capacidad modular para poder construir, reutilizar y compartir codigo, que empuja a todo una comunidad de profesionales y entusiastas

Desde esta definicion, comienzo con esta primera parte que habla sobre "herramienta de aprovisionamiento" y como podemos hacerlo de manera segura. Las primeras preguntas que nos debemos hacer son:

- Quien o que ejecuta terraform?
- Como se ejecuta? Localmente, un pipeline, una funcion, via CDK?
- Tenemos registro y claridad de las acciones de terraform?

Estas preguntas estan orientadas al inicio, a la seguridad que debemos agregar como REQUERIMIENTO en la arquitectura de infraestructura como codigo o como parte de la estructura de nuestra revision de la preparacion operativa (ORR). 

Pensemos por un momento en la segregacion de responsabilidades respecto a quien o que ejecuta terraform; tenemos un usuario que crea el codigo, otras identidades que autorizan y validan los cambios (CI/CD y/o otro usuario "autorizador"), y otra identidad que aplica los cambios descritas por el codigo.

## Crear codigo

Nuestros usuarios que crean el codigo (HCL) dentro de esta segregacion deben tener permisos de read/write y acceso al repositorio via organizaciones, grupos o otra estructura que permita agregarlos segun su nivel de responsabilidad o participacion (RBAC). Esta primera diferenciacion sobre el rol del usuario y su temporalidad es la primera barrera para nuestros repositorios. La autenticacion por SAML nos permite que una vez que un usuario es revocado de la fuente de autenticacion, este no pueda volver a acceder a los recursos a los que tuvo acceso, pero esto significa que hasta que los usuarios son eliminados debemos hacer una revision de quienes tienen acceso a nuestro repositorio? Por supuesto que no. Como una medida de higiene, debemos verificar y darle mantenimiento tanto a repositorios como accesos a estos. Los permisos no son estaticos ni para siempre. 

Un patron comun para establecer permisos es: Ningun usuario por defecto debe tener acceso al repositorio, despues solo las personas que pertenecen aun grupo o org, que puedan ver el codigo, logs de ci/cd e informacion del pipeline. Finalmente solo los developers de IaC que crean cambios, deberian tener acceso de escritura en el repositorio. Esto puede sonar muy simple, pero una buena estructura y mantener el principio del menor privilegio (PoLP) nos ayuda a evitar indexaciones inecesarias, una arquitectura expuesa o riesgo si hacemos un push de informacion sensible accidentalmente.

Ya hablamos de los permisos, pero a que deberian tener acceso los usuarios que crean o modifican el codigo? Unicamente al repositorio que mantienen. No deberian tener acceso al almacenamiento que contiene el tfstate file o a deployear ellos la infraestructura. Historicamente, la persona que mantiene el codigo, tiene acceso a realizar los cambios en los ambientes desplegados, pero es esto un must? La realidad es que no. No necesariamente el usuario que hace cambios debe tener acceso a modificar la infraestructura donde se realizan los cambios, es solo una comodidad que los desarrolladores de IaC ha heredado de roles operativos.

## Provedores, modulos y versiones

Si vienes del "mundo" dev, definir las versiones de las bibliotecas y el lenguaje, analizar las ventajas y desventajas entre las versiones y su soporte/desarrollo a largo plazo es una de las primeras definiciones que se hacen en los proyectos. En terraform no es diferente, pero si tenemos una limitacion y es que actualmente el control de versiones se hace por tags y no por hashes, lo cual no nos garantiza integridad. Si una version es sobreescrita, no nos daremos cuenta de ello.

Aqui tengo otra mala noticia. Al momento de escribir esto, no existe una manera de tener alertas si estamos usando una version vulnerable de terraform o si algun proveedor fue marcado como infectado/comprometido, que no sea usando un analizador estatico, es decir, si yo ejecuto `terraform init`, terraform no me alertara sobre esto como lo hace NPM.

Porque es importante el control de las versiones? Por los ataques de cadena de suministro (supply chain). La version de requerida de terraform, la version de los proveedores y de los modulos puede ser fija o dinamica. Dinamica cuando cualquier version superior a x.y.z (`>= x.y.z`) es aceptada o cuando buscamos que la version x.y tenga el ultimo parche de z (`~> x.y.z`).

Si usamos provedores no oficiales y/o modulos de la comunidad, nos exponemos aun mas a este tipo de ataques. Bueno, pero cual seria el impacto aqui con un modulo que sea comprometido? Owning total de la infraestructura donde desplegamos terraform. Como limitamos este impacto? Con los permisos limitados a la identidad que despliega terraform.

Finalmente, que es lo mejor que podemos hacer para prevenir lo mas posible con el menor esfuerzo? Dejar fijas las versiones (`version = "x.y.z"`). Ventajas adicionales: inmutabilidad.  

## tfstate file

En este punto ya tenemos un repositorio con los permisos correctos, nuestros usuarios con unicamente acceso al repositorio y a las herramientas del pipeline de CI/CD para validar y debuggear el codigo. El siguiente elemento a proteger y uno de los mas criticos, es el archivo de estado de terraform o tfstate file. Este archivo contiene la informacion de como se encuentra nuestra infraestructura actualmente, por ello resulta igual de sensible que exponer nuestro codigo, pero con un mayor impacto, porque si alguien altera este archivo, puede modificar nuestra infraestructura. Hablare mas a detalle sobre el contenido este archivo en otro post, pero por ahora centremonos en protegerlo.

Para proteger el tfstate file, se suele usar backends que ofrezcan lo siguiente:
- Cifrado at-rest y on-transit. Si alguien intercepta o logra descargar el archivo, este debe estar protegido.
- Bloqueo. Solo un usuario a la vez puede acceder al archivo para evitar corrupcion o tampering.
- Unico. Centralizar el archivo de estado en un solo lugar.
- Acceso restringido. Limitar por politicas el acceso a los archivos dentro del backend.

## Shift Left

Finalmente en la parte de crear codigo me gustaria mencionar el uso de algunas herramientas en particular que ayudan a hacer codigo mas seguro:

- git hooks. 
- LSP 

# LGTM

En la autorizacion y validacion del codigo es donde participan muchas herramientas y/o scanners estaticos que se encargan de comprender la "delta" de cambios; el estado destino (u objetivo) de como queremos tener nuestra infraestructura despues de la ejecucion terraform. Que pasa con los permisos de la identidad usada para ejecutar terraform? Tiene unicamente los permisos para crear, modificar y destruir aquellos elementos definidos por codigo o tiene mas permisos tipo "*"? Esta parte en terminos operativos es complicado y tedioso, porque si queremos agregarle nuevas funciones a nuestro codigo, tambien tenemos que actualizar el usuario que se usa para aplicar el codigo. 

Abramos ahora escenario para la identidad que se encarga de la integracion continua de nuestro codigo. Usualmente en un pipeline tenemos una puerta o stage de validacion del codigo, esta se encarga de tomar el codigo y utilizar herramientas que se encargan de dar formato (terraform fmt -recursive) y de validar la sintaxis del codigo (terraform validate). 
