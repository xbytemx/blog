---
title: "Recomendaciones de seguridad para Terraform (primera parte)"
date: 2023-12-13T17:51:15-06:00
draft: true
description: "Practicas de seguridad en infrastructura como codigo en Hashicorp Terraform. Primera parte, requerimientos para un aprovisionamiento seguro."
tags: ["iac","terraform","devops"]
categories: ["devsecops","cloudsec"]

---
# Terraform

Como todo parte siempre desde algun punto, trate de describir que es terraform en mis propias palabras para de ahi construir esta serie de posts, asi que sin mas que agregar: Que es terraform (el producto de Hashicorp)?

> Una herramienta de aprovisionamiento, creada para escribir codigo enfocandonos en el estado objetivo, con una gran capacidad modular para poder construir, reutilizar y compartir codigo, que empuja a todo una comunidad de profesionales y entusiastas

Desde esta definicion, comienzo con esta primera parte (de varias) donde me estare enfocando en la seguridad de la "herramienta de aprovisionamiento". Supongamos que tenemos una aplicacion segura, que sigue las mejores practicas y que ha pasado por sus respectivas pruebas. Tenemos nuestra aplicacion funcionando con algun proveedor de terraform o nos encontramos en medio de una implementacion nueva de la infraestructura.

Las primeras preguntas que nos debemos hacer son:

- Quien va a hacer que terraform construya la infraestructura?
- En donde se ejecuta Terraform? Localmente, un pipeline, una funcion, via CDK?
- Tenemos registro y claridad de las acciones que realizara Terraform?

Estas son algunas preguntas importantes de arranque cuando agregamos la seguridad como REQUERIMIENTO. La infraestructura como codigo tiene sus inputs y outputs, pero a nivel de ejecucion o de "aquello que permite aplicar cambios en terraform", existen elementos que dificilmente son evaluados automaticamente como las identidades y sus roles desde el punto de vista de interaccion con terraform, permisos adecuados, etc. La estructura de la revision de preparacion operativa (ORR) ejecuta este tipo de validaciones para que posteriormente un equipo operativo ya se encargue de las tareas del dia a dia sin estas preocupaciones, o en su defecto, sea conciente de los riesgos actuales que involucra la operacion.

Pensemos por un momento en la segregacion de responsabilidades respecto a quien o que ejecuta terraform; tenemos un usuario que crea el codigo, otras identidades que autorizan y validan los cambios (CI/CD y/o otro usuario "autorizador"), y otra identidad que aplica los cambios descritas por el codigo.

# Crear codigo

Nuestros usuarios que crean el codigo (HCL), de acuerdo a esta diferenciacion, deben tener permisos de read/write y acceso al repositorio. Se recomienda usar organizaciones, grupos o otra estructura que permita agregarlos segun su nivel de responsabilidad o participacion (RBAC idealmente). Esta primera diferenciacion sobre el rol del usuario y su temporalidad es la primera barrera para nuestros repositorios. La autenticacion por SAML nos permite que cuando un usuario es revocado de la fuente de autenticacion, este no pueda volver a acceder a los recursos a los que una vez tuvo acceso, pero esto significa que hasta que los usuarios son eliminados debemos hacer una revision de quienes tienen acceso a nuestro repositorio? Por supuesto que no. Como una medida de higiene, debemos verificar y darle mantenimiento tanto a repositorios como los accesos a estos ultimos. Los permisos no son estaticos ni para siempre. 

Un patron jerargico muy comun hoy en dia para establecer permisos es el siguiente:

- Other: Ningun usuario por defecto debe tener acceso al repositorio
- Group: Las personas que pertenecen aun grupo o org, que puedan ver el codigo, logs de CI/CD e informacion del pipeline. 
- Active Developer: Los developers de IaC que crean cambios, deberian tener acceso de escritura en el repositorio. 

Esto puede sonar muy simple, pero una buena estructura y mantener el principio del menor privilegio (PoLP) nos ayuda a evitar indexaciones inecesarias, una arquitectura expuesa o el riesgo de hacer un push de informacion sensible accidentalmente.

Ya hablamos de los permisos, pero a que deberian tener acceso los usuarios que crean o modifican el codigo? Unicamente al repositorio que mantienen. No deberian tener acceso al almacenamiento que contiene el tfstate file o a deployear ellos la infraestructura. Historicamente, la persona que mantiene el codigo, tiene acceso a realizar los cambios en los ambientes desplegados, pero es esto un *must?* La realidad es que no. No necesariamente el usuario que hace cambios debe tener acceso a modificar la infraestructura donde se realizan los cambios, es solo una comodidad que los desarrolladores de IaC que se ha heredado de roles operativos.

## Provedores, modulos y versiones

Si vienes del "mundo" dev, definir las versiones de las bibliotecas y el lenguaje, analizar las ventajas y desventajas entre las versiones y su soporte/desarrollo a largo plazo es una de las primeras definiciones que se hacen en los proyectos. En terraform no es diferente, pero si tenemos una observacion de seguridad importante y es que actualmente el control de versiones se hace por tags y no por hashes, lo cual no nos garantiza integridad. Si una version es sobreescrita, no nos daremos cuenta de ello, porque el tag es el mismo.

Aqui tengo otra mala noticia. Al momento de escribir esto, no existe una manera de tener alertas si estamos usando una version vulnerable de terraform o si algun proveedor fue marcado como infectado/comprometido, que no sea usando un analizador estatico, es decir, si yo ejecuto `terraform init`, terraform no me alertara sobre esto como lo hace NPM.

Porque es importante el control de las versiones? Por los ataques de cadena de suministros (supply chain). La version de requerida de terraform, la version de los proveedores y de los modulos puede ser fija o dinamica. Dinamica cuando cualquier version superior a x.y.z (`>= x.y.z`) es aceptada o cuando buscamos que la version x.y tenga el ultimo parche de z (`~> x.y.z`).

Si usamos provedores no oficiales y/o modulos de la comunidad, nos exponemos aun mas a este tipo de ataques. Bueno, pero cual seria el impacto aqui con un modulo que sea comprometido? Owning total de la infraestructura donde desplegamos terraform. Como limitamos este impacto? Con los permisos limitados a la identidad que despliega terraform.

Finalmente, que es lo mejor que podemos hacer para prevenir lo mas posible con el menor esfuerzo? Dejar fijas las versiones (`version = "x.y.z"`). Ventajas adicionales: inmutabilidad.  

## tfstate file

En este punto ya tenemos un repositorio con los permisos correctos, nuestros usuarios tienen unicamente acceso al repositorio y a las herramientas del pipeline de CI/CD para validar y debuggear el codigo. El siguiente elemento a proteger y uno de los mas criticos, es el archivo de estado de terraform o tfstate file. Este archivo contiene la informacion de como se encuentra nuestra infraestructura actualmente, por ello resulta igual de sensible que exponer nuestro codigo, pero con un mayor impacto, porque si alguien altera este archivo, puede modificar nuestra infraestructura. Hablare mas a detalle sobre el contenido este archivo en otro post, pero por ahora centremonos en protegerlo. 

> Si usas Terraform Cloud, probablemente ya sabes que este no es un riesgo tuyo. Al menos por ahora.

Para proteger el tfstate file, se suele usar backends que ofrezcan lo siguiente:
- Cifrado at-rest y on-transit. Si alguien intercepta o logra descargar el archivo, este debe estar protegido.
- Bloqueo. Solo un usuario a la vez puede acceder al archivo para evitar corrupcion o tampering.
- Centralizado. Centralizar el archivo de estado en un solo lugar donde sera alcanzado por terraform.
- Acceso restringido. Limitar por politicas el acceso a los archivos dentro del backend.

### AWS - S3 + DynamoDB

Este es un ejemplo de como podriamos tener nuestro codigo con el backed de AWS S3+DynamoDB:

```hcl
```

### Azure

Este es un ejemplo de como podriamos tener nuestro codigo con el backed de Azure:

```hcl
```

### GCP

Este es un ejemplo de como podriamos tener nuestro codigo con el backed de GCP:

```hcl
```

## Shift Left Security at CODE

Finalmente en la parte de crear codigo me gustaria mencionar el uso de algunas herramientas que ayudan a hacer el codigo mas seguro antes de que sea mas complicado. La idea de aplicar shift left security para la etapa de desarrollo, es realizar evaluaciones de seguridad localmente para evitar hacer un commit de algo que ya tiene una vulnerabilidad. Evitar un problema de seguridad simple y poco complejo antes de compartirlo en el repositorio o que sea detectado por las herramientas SAST en el pipeline. Algunas de estas recomendaciones son:

### 1. Evitar publicar secretos y datos sensibles
Si necesitamos transportar informacion que pueda ponernos en riesgo via terraform, podemos usar el valor de sensitve en las variables para evitar dejar registro en los logs del CI/CD

### 2. Usar herramientas SAST y CSPM

### 3. Compliance

## Unit Testing

Al momento de escribir esto, la version [1.7](https://www.hashicorp.com/blog/terraform-1-7-adds-test-mocking-and-config-driven-remove) de terraform fue liberada y esta incluye capacidad de hacer una emulacion de los proveedores (al fin). Aunque aun no la he probado, la estrategia de unit and integration testing se ha comenzado a publicar desde la version 1.6, solo que ahora si tiene la integracion para emular los proveedores y no solo input/output.  

# LGTM

En la autorizacion y validacion del codigo es donde participan muchas herramientas y/o scanners estaticos que se encargan de comprender la "delta" de cambios; el estado destino (u objetivo) de como queremos tener nuestra infraestructura despues de la ejecucion terraform. Que pasa con los permisos de la identidad usada para ejecutar terraform? Tiene unicamente los permisos para crear, modificar y destruir aquellos elementos definidos por codigo o tiene mas permisos tipo "*"? Esta parte en terminos operativos es complicado y tedioso, porque si queremos agregarle nuevas funciones a nuestro codigo, tambien tenemos que actualizar el usuario que se usa para aplicar el codigo. 

Abramos ahora escenario para la identidad que se encarga de la integracion continua de nuestro codigo. Usualmente en un pipeline tenemos una puerta o stage de validacion del codigo, esta se encarga de tomar el codigo y utilizar herramientas que se encargan de dar formato (terraform fmt -recursive) y de validar la sintaxis del codigo (terraform validate). 

Las herramientas SAST y DAST vuelven a hacer su presencia aqui, tal como vimos en la seccion anterior, es importante tener analizadores que activamente esten buscando por problemas, malas configuraciones y leaks. Mi recomendacion todo en uno para el pipeline es [megalinter](https://megalinter.io/latest/). Uno de los retos mas importantes para la parte DAST en terraform, es que tentativamente terraform nos dice que cambios hara al momento de desplegar el plan a ejecutar. 

# Deploy

