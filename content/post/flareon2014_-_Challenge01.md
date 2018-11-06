---
title: "Flare-On 2014 - Challenge 01: Bob Doge"
date: 2018-10-21T20:25:36-05:00
draft: false
tags: ["flareon","revisitado","flareon2014","dotnet","reversing","writeup"]
categories: ["reversing","ctf"]
---

Iniciamos otra serie de writeups acerca de reversing, en este caso siguiendo todas las soluciones hasta el momento de todos los eventos de flare-on (ambicioso proyecto jeje), bueno iniciemos por el primer challenge del 2014, llamado simplemente Bob Doge.
<!--more-->

Comenzamos por descargar el [zip](http://www.flare-on.com/files/2014_FLAREOn_Challenges.zip) que contiene todos los challenges, para posteriormente ir a cada challenge y resolverlos.

Bien, después de descargar el archivo general y descomprimirlo, debemos tener en nuestra carpeta un archivo llamando C1.exe, el cual posteriormente lo pasamos por file y nos indica vía headers que se trata de un 'PE32+ ejecutable con GUI x86_64', por lo que probamos a ejecutarlo en un sandbox, ahí veremos que se trata de Microsoft Cabinet, que nos hace aceptar un EULA de Fireeye, por lo que extraemos su contenido usando cabextract:

```bash
xbytemx@laptop:~/flare-on2014$ cabextract C1.exe
Extracting cabinet: C1.exe
  extracting Challenge1.exe

All done, no errors.
```

Nos despliega en pantalla que ha extraído correctamente "Challenge1.exe", el cual nuevamente lo pasamos por file:

```
xbytemx@laptop:~/flare-on2014$ file Challenge1.exe
Challenge1.exe: PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows
```

El comando file nos indica que se trata de un PE32 escrito en Mono/.Net, lo cual nos hace un guiño a que podemos decompilarlo usando alguna herramienta que entienda .Net, en mi caso use dnSpy:

```
xbytemx@laptop:~/flare-on2014$ mono ~/tools/dnSpy/dnSpy.Console.exe -o challenge01 Challenge1.exe
ERROR: File creation failed (/home/xbytemx/flare-on2014/challenge01/XXXXXXXXXXXXXXX/rev_challenge_1/Form1.resx), description = Create ResX file, Exception message: Object reference not set to an instance of an object
```

Tras descompilar el binario, depositamos su contenido en la carpeta challenge01, la cual tiene la siguiente estructura:

```
xbytemx@laptop:~/flare-on2014/challenge01$ ls -lahR
.:
total 16K
drwxr-xr-x 3 xbytemx xbytemx 4.0K oct 21 21:15 .
drwxr-xr-x 3 xbytemx xbytemx 4.0K oct 21 21:15 ..
-rw-r--r-- 1 xbytemx xbytemx  935 oct 21 21:15 solution.sln
drwxr-xr-x 4 xbytemx xbytemx 4.0K oct 21 21:15 XXXXXXXXXXXXXXX

./XXXXXXXXXXXXXXX:
total 40K
drwxr-xr-x 4 xbytemx xbytemx 4.0K oct 21 21:15 .
drwxr-xr-x 3 xbytemx xbytemx 4.0K oct 21 21:15 ..
-rw-r--r-- 1 xbytemx xbytemx  490 oct 21 21:15 app.manifest
-rw-r--r-- 1 xbytemx xbytemx 1018 oct 21 21:15 Form1.cs
-rw-r--r-- 1 xbytemx xbytemx 2.9K oct 21 21:15 Form1.Designer.cs
-rw-r--r-- 1 xbytemx xbytemx  390 oct 21 21:15 Program.cs
drwxr-xr-x 2 xbytemx xbytemx 4.0K oct 21 21:15 Properties
drwxr-xr-x 2 xbytemx xbytemx 4.0K oct 21 21:15 rev_challenge_1
-rw-r--r-- 1 xbytemx xbytemx   31 oct 21 21:15 rev_challenge_1.dat_secret.encode
-rw-r--r-- 1 xbytemx xbytemx 3.3K oct 21 21:15 XXXXXXXXXXXXXXX.csproj

./XXXXXXXXXXXXXXX/Properties:
total 100K
drwxr-xr-x 2 xbytemx xbytemx 4.0K oct 21 21:15 .
drwxr-xr-x 4 xbytemx xbytemx 4.0K oct 21 21:15 ..
-rw-r--r-- 1 xbytemx xbytemx  810 oct 21 21:15 AssemblyInfo.cs
-rw-r--r-- 1 xbytemx xbytemx 2.7K oct 21 21:15 rev_challenge_1.Properties.Resources.Designer.cs
-rw-r--r-- 1 xbytemx xbytemx  76K oct 21 21:15 rev_challenge_1.Properties.Resources.resx
-rw-r--r-- 1 xbytemx xbytemx  717 oct 21 21:15 Settings.Designer.cs
-rw-r--r-- 1 xbytemx xbytemx  279 oct 21 21:15 Settings.settings

./XXXXXXXXXXXXXXX/rev_challenge_1:
total 8.0K
drwxr-xr-x 2 xbytemx xbytemx 4.0K oct 21 21:15 .
drwxr-xr-x 4 xbytemx xbytemx 4.0K oct 21 21:15 ..
```

En la carpeta raíz, encontraremos el archivo _solution_ del proyecto "XXXXXXXXXXXXXXX", dentro de la siguiente carpeta "XXXXXXXXXXXXXXX" encontramos varios archivos:

| Archivo                           | Tipo       |
| --------------------------------- | ---------- |
| app.manifest                      | XML        |
| Form1.cs                          | C++ source |
| Form1.Designer.cs                 | C++ source |
| Program.cs                        | C++ source |
| Properties                        | Directory  |
| rev_challenge_1.dat_secret.encode | data       |
| rev_challenge_1                   | Directory  |
| XXXXXXXXXXXXXXX.csproj            | XML        |

Comenzamos por revisar el archivo Program.cs, el cual tiene el siguiente contenido:

```
using System;
using System.Windows.Forms;

namespace XXXXXXXXXXXXXXX
{
	// Token: 0x02000003 RID: 3
	internal static class Program
	{
		// Token: 0x06000005 RID: 5 RVA: 0x000023A9 File Offset: 0x000005A9
		[STAThread]
		private static void Main()
		{
			Application.EnableVisualStyles();
			Application.SetCompatibleTextRenderingDefault(false);
			Application.Run(new Form1());
		}
	}
}
```

En el código vemos que se crea una nueva aplicación, una con Form1. Veamos que tenemos en Form1.Designer:

```
namespace XXXXXXXXXXXXXXX
{
	// Token: 0x02000002 RID: 2
	public partial class Form1 : global::System.Windows.Forms.Form
	{
		// Token: 0x06000003 RID: 3 RVA: 0x0000215E File Offset: 0x0000035E
		protected override void Dispose(bool disposing)
		{
			if (disposing && this.components != null)
			{
				this.components.Dispose();
			}
			base.Dispose(disposing);
		}

		// Token: 0x06000004 RID: 4 RVA: 0x00002180 File Offset: 0x00000380
		private void InitializeComponent()
		{
			this.lbl_title = new global::System.Windows.Forms.Label();
			this.pbRoge = new global::System.Windows.Forms.PictureBox();
			global::System.Windows.Forms.Button button = new global::System.Windows.Forms.Button();
			((global::System.ComponentModel.ISupportInitialize)this.pbRoge).BeginInit();
			base.SuspendLayout();
			button.Font = new global::System.Drawing.Font("Microsoft Sans Serif", 16f, global::System.Drawing.FontStyle.Regular, global::System.Drawing.GraphicsUnit.Point, 0);
			button.Location = new global::System.Drawing.Point(210, 387);
			button.Name = "btnDecode";
			button.Size = new global::System.Drawing.Size(139, 52);
			button.TabIndex = 0;
			button.Text = "DECODE!";
			button.UseVisualStyleBackColor = true;
			button.Click += new global::System.EventHandler(this.btnDecode_Click);
			this.lbl_title.AutoSize = true;
			this.lbl_title.Font = new global::System.Drawing.Font("Microsoft Sans Serif", 20f, global::System.Drawing.FontStyle.Regular, global::System.Drawing.GraphicsUnit.Point, 0);
			this.lbl_title.Location = new global::System.Drawing.Point(95, 18);
			this.lbl_title.Name = "lbl_title";
			this.lbl_title.Size = new global::System.Drawing.Size(393, 31);
			this.lbl_title.TabIndex = 1;
			this.lbl_title.Text = "Let's start with something easy!";
			this.pbRoge.Image = global::XXXXXXXXXXXXXXX.Properties.Resources.bob_ross;
			this.pbRoge.Location = new global::System.Drawing.Point(87, 65);
			this.pbRoge.Name = "pbRoge";
			this.pbRoge.Size = new global::System.Drawing.Size(401, 316);
			this.pbRoge.TabIndex = 2;
			this.pbRoge.TabStop = false;
			base.AutoScaleDimensions = new global::System.Drawing.SizeF(6f, 13f);
			base.AutoScaleMode = global::System.Windows.Forms.AutoScaleMode.Font;
			base.ClientSize = new global::System.Drawing.Size(566, 444);
			base.Controls.Add(this.pbRoge);
			base.Controls.Add(this.lbl_title);
			base.Controls.Add(button);
			base.FormBorderStyle = global::System.Windows.Forms.FormBorderStyle.Fixed3D;
			base.Name = "Form1";
			this.Text = "Form1";
			((global::System.ComponentModel.ISupportInitialize)this.pbRoge).EndInit();
			base.ResumeLayout(false);
			base.PerformLayout();
		}

		// Token: 0x04000001 RID: 1
		private global::System.ComponentModel.IContainer components;

		// Token: 0x04000002 RID: 2
		private global::System.Windows.Forms.Label lbl_title;

		// Token: 0x04000003 RID: 3
		private global::System.Windows.Forms.PictureBox pbRoge;
	}
}
```

Aquí ya tenemos algo mas de detalle sobre las características de la aplicación, vemos que tenemos un label llamado lbl_title y un cuadro de imagen llamada pbRoge. Al inicializarse, sus variables son asignadas, vemos que se crea un botón, con el contenido "DECODE!", que tras recibir un evento de click, llama a this.btnDecode_Click.

El titulo tiene el mensaje "Let's start with something easy!" y en el contenido de la imagen tenemos una referencia a "global::XXXXXXXXXXXXXXX.Properties.Resources.bob_ross", que por la parte final debe ser alguna imagen de bob_ross.

Finalmente, revisamos el contenido del ultimo archivo de código fuente, Form1:

```
using System;
using System.ComponentModel;
using System.Drawing;
using System.Windows.Forms;
using XXXXXXXXXXXXXXX.Properties;

namespace XXXXXXXXXXXXXXX
{
	// Token: 0x02000002 RID: 2
	public partial class Form1 : Form
	{
		// Token: 0x06000001 RID: 1 RVA: 0x00002050 File Offset: 0x00000250
		public Form1()
		{
			this.InitializeComponent();
		}

		// Token: 0x06000002 RID: 2 RVA: 0x00002060 File Offset: 0x00000260
		private void btnDecode_Click(object sender, EventArgs e)
		{
			this.pbRoge.Image = Resources.bob_roge;
			byte[] dat_secret = Resources.dat_secret;
			string text = "";
			foreach (byte b in dat_secret)
			{
				text += (char)((b >> 4 | ((int)b << 4 & 240)) ^ 41);
			}
			text += "\0";
			string text2 = "";
			for (int j = 0; j < text.Length; j += 2)
			{
				text2 += text[j + 1];
				text2 += text[j];
			}
			string text3 = "";
			for (int k = 0; k < text2.Length; k++)
			{
				char c = text2[k];
				text3 += (char)((byte)text2[k] ^ 102);
			}
			this.lbl_title.Text = text3;
		}
	}
}
```

Como podemos observar por el código, este Form recibe el evento del botón y realiza un decoding de la información que se encuentra en dat_secret, si recordamos en la carpeta tenemos un archivo tipo data que incluye en su nombre "dat_secret". También vemos que el valor de la imagen bob_ross, cambia a bob_roge.

Ejecutemos el binario para ver que sucede:

![bob_ross](/img/flareon2014-c1/bob_ross.png)

Hagamos click sobre el botón de "DECODE!", vemos que tanto el texto como la imagen cambian:

![bob_roge](/img/flareon2014-c1/bob_roge.png)

El texto se ve ilegible, recordando que es el resultado del 3er decoding "text3", hagamos el mismo proceso tomando la información del archivo "rev_challenge_1.dat_secret.encode" con el siguiente programa:

```
#!/usr/bin/env python3

# En bash use: hexdump -e '"\\\x" /1 "%02x"' rev_challenge_1.dat_secret.encode
encode_data = bytearray(b'\xa1\xb5\x44\x84\x14\xe4\xa1\xb5\xd4\x70\xb4\x91\xb4\x70\xd4\x91\xe4\xc4\x96\xf4\x54\x84\xb5\xc4\x40\x64\x74\x70\xa4\x64\x44')

t1=""
t2=""
t3=""

for i in range(len(encode_data)):
    t1 += chr(((encode_data[i] >> 4) | ((encode_data[i] << 4) & 240)) ^ 41)
print(t1)

for j in range(0, len(t1)-1, 2):
    t2 += t1[j+1]
    t2 += t1[j]
print(t2)

for k in range(len(t2)):
    t3 += chr(ord(t2[k]) ^ 102)
print(t3)
```

Este pequeño código lo que realiza es tomar procesar el binario encodeado y devolvernos las mismas operaciones de encoding. Tras ejecutarlo tenemos lo siguiente:

```
xbytemx@laptop:~/flare-on2014/challenge01$ python3 decode.py
3rmahg3rd.b0b.d0ge@flare-on.com
r3amghr3.d0b.b0degf@alero-.noc
U
 UHVHV&
	KH
```

Como podemos observar, el primer resultado corresponde a una dirección de correo, la cual es la flag de este reto.

**flag: 3rmahg3rd.b0b.d0ge@flare-on.com**

---

Espero que les haya gustado.

Saludos,
