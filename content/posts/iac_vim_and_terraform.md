---
title: "Terraform, neovim y mi experiencia tras una semana"
date: 2021-01-23T13:50:09-06:00
toc: false
images:
tags: 
  - infraestructure-as-code
  - iac
  - terraform
  - vim
  - neovim
---

¿Haz tenido curiosidad de usar Terraform para administrar tus recursos en AWS o GCP? Bueno yo también. Tengo una semana explorando sus conceptos, capacidades y restricciones, en donde hasta el momento, ha sido una experiencia muy divertida y que realmente he estado disfrutando poco a poco. Los primeros dos días estuve construyendo un ambiente de trabajo que me ayudara a integrar y gestionar correctamente la herramienta de aprovisionamiento, así que decidí escribir esta pequeña guía de algunos consejos y herramientas para iniciar propiamente con terraform.

<!--more-->

Quiero comenzar por mencionar que propiamente no soy desarrollador de software, así que mis necesidades están orientadas a mantener un ambiente estable que pueda transportar y ejecutar remotamente. Dicho lo anterior, hace muchos años elegí VI como editor de texto y más actualmente uso nvim (NeoVIm), mí propósito con este articulo es extender la capacidad de nvim para poder detectar errores de sintaxis, autocompletar elementos predefinidos en el lenguaje y cuidar de manera más activa el formato del estilo HCL (espacios, lineas, tabs) de mis archivos de configuración de Terraform. Fuera de neovim, hice algunos ajustes en mi bash para poder autocompletar algunos comandos de terraform y para manejar de manera mas segura las claves de acceso de AWS.

Mi recurso principal de aprendizaje actualmente es el libro [Terraform: Up and Running, 2nd Edition](https://learning.oreilly.com/library/view/terraform-up/9781492046899/) de donde también saque algunas ideas y consejos sobre la primera parte relacionada a como _configurar nuestro proveedor de nube_. Mis demás recursos han sido artículos en blogs y algunas respuestas de stackoverflow.

## Amazon Web Services

AWS es el proveedor de nube donde hice mis pruebas y pude comprobar que terraform tiene bastante madurez y una comunidad muy activa. Agregue esta sección porque considero importante compartir estas practicas antes de comenzar con terraform. Muchos tutoriales que encontré omiten practicas de seguridad en la nube. Ya que este no es un tutorial de AWS, omitiré algunos conceptos y supondré que el lector ya esta mas familiarizado con la consola de AWS y sus productos.

### Definir una región

En mi caso, estaré trabajando en la región *us-east-2* para mantener un poco separado lo que ya tengo en otras regiones. Elegir una región es importante en terraform, porque le dice al `provider` donde desplegara la infraestructura.

### Crear un usuario en IAM únicamente usar terraform

Crear un usuario o varios, delimita quienes pueden tener la capacidad de tener acceso a nuestra infraestructura y nos permite mantener una bitácora mas ordenada. Terraform requiere de acceso a la nube usando credenciales de "programación" con los permisos suficientes para crear, destruir o actualizar los componentes que nosotros definamos por código.

Para añadir un nuevo usuario en AWS accedemos a [IAM->Usuarios->Añadir Usuarios](https://console.aws.amazon.com/iam/home?region=us-east-2#/users$new?step=details) donde seguiremos el wizard de creación de usuarios. En mi caso asigne el nombre de *tfcli* para mi usuario. El tipo de acceso debe ser mediante "programación", es decir, usando un ID y una clave secreta, como se menciono anteriormente.

![Add user](/img/terraform1/aws1_usuario.png)

Lo siguiente que requerimos definirle a nuestro usuario son los permisos. En este _breve_ tutorial/guia, utilizare únicamente el dominio completo sobre EC2, pero en un caso mas practico, el usuario tendría acceso a todo lo que definiríamos por código, tales como el propio IAM, S3, Cloudfront, etc. 

Nota, una buena practica que aprendí posteriormente, es guardar el estado de nuestra infraestructura (tfstate) en un [S3 bucket](https://www.terraform.io/docs/language/settings/backends/s3.html) para poderlo compartir y agregarle una tabla con Dynamo DB para habilitar un control o seguro, cuando mas de un usuario quiera realizar cambios. Para el alcance de esta guia/tutorial, dejaremos eso de lado.

![Perms for tfcli](/img/terraform1/aws2_permisos.png)

Continuamos con las etiquetas, que de momento podemos ignorar. Después sigue la revisión donde debemos verificar que nuestro usuario es consistente con credenciales de "programación" y que únicamente tiene permisos para EC2. Seleccionamos crear para continuar y cuando pasemos al paso 5, se nos compartirá las credenciales: 'ID de clave de acceso' y 'Clave de acceso secreta'. Es muy importante *donde y como* guardamos estas credenciales, en mi caso decidí *NO* descargar el CVS, y en su lugar almacenar las credenciales en un archivo [keepassx](https://www.keepassx.org/) usando `kpcli`.

![Created User](/img/terraform1/aws3_creado.png)

![keepassx](/img/terraform1/aws4_keepassx.png)

### Usar una bóveda de credenciales

Finalmente la ultima nota respecto a AWS y terraform es sobre usar una bóveda de credenciales. Almacenar credenciales en texto plano es una horrible practica que seguimos por comodidad o por alguna otra razón. No solo es almacenar la información de manera segura, sino acceder a estas credenciales cuando estamos trabajando.

`aws-vault` es una herramienta que nos permite gestionar pares de credenciales de "programación" dentro de AWS. Tiene la ventaja interesante de usar un keyring para almacenar nuestras credenciales y de esa manera acceder con la identidad respectiva en el entorno necesario. Para comenzar con `aws-vault`, descargamos el binario precompilado, lo guardamos en algún directorio que este en nuestro `$PATH` y inicializamos la bóveda agregando un par de credenciales. En mi caso use `~/.local/bin` para almacenar el binario, estas credenciales me permitieron crear el keyring, y utilizando `aws-vault exec` pude cargar las variables de entorno que están guardadas de manera segura.

![aws-vault](/img/terraform1/aws5_awsvault.png)

## Terraform

Para instalar terraform hay varias formas, tenemos repositorios de distribuciones, podemos usar las fuentes o podemos bajar el precompilado (mi opción). Para mas información, [aquí](https://www.terraform.io/downloads.html) y [aquí](https://learn.hashicorp.com/collections/terraform/aws-get-started?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS).

![terraform install](/img/terraform1/tf1_instalacion.png)

Después de instalar el binario, una opción que no aparece en el menú de help, pero si en la documentación, es `-install-autocomplete`. Esta nos ayudara autocompletar alguna de sus funciones y con ello familiarizarnos más.

![terraform autocomplete](/img/terraform1/tf2_autocomplete.png)

Una vez aplicado el cambio y que nuestro bashrc ya carga el autocomplete, podemos presionar `<TAB>` y ver todas esas opciones que terraform nos ofrece.

![terraform opciones](/img/terraform1/tf3_opciones.png)

## Neovim

Por fin llegamos a la parte de neovim. Me encuentro usando `NVIM v0.4.4`, por lo que es importante que puedas verificar tu versión antes de continuar y no tener problemas de compatibilidad. Para mis plugins estoy usando Plug, el cual puedes descargar y configurar directamente del siguiente [repositorio](https://github.com/junegunn/vim-plug).

Mi configuración base (sin Plug) es la siguiente:

```
syntax on
set encoding=utf-8
set cursorline
set shiftwidth=4
set tabstop=4
set number relativenumber

set background=dark
set t_Co=256
colorscheme dracula

set nocompatible
filetype plugin indent on

set splitbelow
set splitright
```

Ya con la versión correcta o compatible de nvim y tu manejador de plugins favorito instalado, podemos pasar configurar el Language Server (LS) que estaremos usando y el cliente de [Language Server Protocol (LSP)](https://microsoft.github.io/language-server-protocol/) que estaremos integrando con nvim.

Aquí se crea una discusión entre [terraform-ls](https://github.com/hashicorp/terraform-ls) y [terraform-lsp](https://github.com/juliosueiras/terraform-lsp). Probé ambos y por compatibilidad e integración, como principiante, les puedo recomendar `terraform-ls`.

Para instalar `terraform-ls` solo debemos bajar el precompilado, descomprimir y guardar el binario. Nuevamente elegí `~/.local/bin` como el destino:

![terraform-ls](/img/terraform1/nvim1_terraformls.png)

Para hacer uso de este servidor de lenguajes, necesitamos integrar nvim con un cliente de LSP. Conquer of Completation o [CoC](https://github.com/neoclide/coc.nvim) es un plugin de nvim que podemos usar para esta tarea, ya que por sus capacidades nos sera fácil gestionar no solo este LS sino muchos más.

Otro plugin que nos puede resultar muy útil es [vim-terraform](https://github.com/hashivim/vim-terraform), que nos ayuda a hacer highlight de los archivos que terminan en `*.tf`, `*.tfvars`, `.terraformrc` y `terraform.rc` como código HCL (HashiCorp Configuration Language) y los `*.tfstate` como JSON.

En nuestro `~/.config/nvim/init.vim` integramos las siguientes lineas al inicio:

```
call plug#begin('~/.vim/plugins')
  Plug 'neoclide/coc.nvim', {'branch': 'release'}

  Plug 'hashivim/vim-terraform'
  let g:terraform_fmt_on_save=1
  let g:terraform_align=1
call plug#end()
```

Con esta configuración integraremos `coc.nvim` y `vim-terraform` con nvim. Las opciones `let` me permite que el formato se aplique al salvar los archivos y que la alineación de espacios se ejecute hasta salvar el archivo.

Guardamos nuestro `~/.config/nvim/init.vim` con el comando `:w` y recargamos nuestra configuración con `:source %`. Finalmente instalamos los plugins con `:PlugInstall`.

El ultimo paso seria configurar `CoC` con `terraform-ls`, para ello creamos (o editamos) el archivo `~/.config/nvim/coc-settings.json` con el siguiente contenido:

```
{
        "languageserver": {
                "terraform": {
                        "command": "terraform-ls",
                        "args": ["serve"],
                        "filetypes": [
                                "terraform",
                                "tf"
                        ],
                        "initializationOptions": {},
                        "settings": {}
                }
        }
}
```

Esta configuración llamara a `terraform-ls serve` cada que necesitemos autocompletar providers, resources, output y demas elementos de terraform, permitiéndonos escribir de manera mas eficiente nuestro código.

Podemos ver un ejemplo de esta integración completa en el siguiente GIF:

![final autocomplete](/img/terraform1/final1.gif)

![final vault](/img/terraform1/final2.gif)

---

Muchas gracias por llegar hasta aquí, espero que hayas disfrutado de esta lectura. Hasta la próxima!
