---
title: "Recomendaciones de seguridad para Terraform (primera parte)"
date: 2023-12-13T17:51:15-06:00
draft: true
description: "Esta es una entrada que discute algunas de las mejores practicas de seguridad en infrastructura como codigo (iac por sus siglas en ingles), y forma parte de una serie de entradas sobre seguridad como codigo. En esta entrada, me estare enfocando en particular con Hashicorp Terraform y en como podemos hacer nuestro codigo y modulos mas seguros."
tags: ["iac","terraform","devops"]
categories: ["devsecops","cloudsec"]

---
Como defino que es terraform en mis propias palabras:

> Una herramienta de aprovisionamiento, creada para escribir codigo enfocandonos en el estado objetivo, con una gran capacidad modular para poder construir, reutilizar y compartir codigo, que empuja a todo una comunidad de profesionales y entusiastas

Desde esta definicion, comienzo con la primera pregunta sobre "herramienta de aprovisionamiento". Quien o que ejecuta terraform? Como se ejecuta? Tenemos registro y claridad de las acciones de terraform?
