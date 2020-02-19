---
title: "Flare-on 2014 - Challenge 05: 5get\_it"
date: 2019-02-05T00:09:23-06:00
draft: false
tags: ["flareon","revisitado","flareon2014","radare2","reversing","writeup","dll"]
categories: ["reversing","ctf"]

---

5get\_it, el quinto reto del primer flare-on de Fireeye. No puedo más que decirles que este si que me ha costado. Comencemos con el writeup de este keylogger.
<!--more-->

Bien comenzamos por descomprimir el challenge "C5.zip", el cual esta protegido por la contraseña "malware".

> Nota: Todos los archivos de challenge se encuentran dentro de un zip que se descarga [aquí][c2c76e1c]
[c2c76e1c]: http://www.flare-on.com/files/2014_FLAREOn_Challenges.zip "Flare-On 2014"

![Unzip](/img/flareon2014-c5/unzip.png)

Tomamos algo de información base sobre el binario:

![Información Base](/img/flareon2014-c5/informacion-base.png)

Ok, tenemos el primer vistazo que nos dice que tratamos con una DLL que no se encuentra empaquetada, con formato PE para 32 bits y que salvo la imagen base, todo parece normal.

Continuemos identificándolo en radare2:

![Análisis base de Radare2](/img/flareon2014-c5/ia1.png)

El show info all o _ia_ de radare2 nos entrega mucha información, que coincide con lo primero que empezamos a observar, adicionalmente nos devuelve que tenemos bastante funciones importadas (al menos 82) las cuales no coloco en imagen para no perderlas de vista:

| Num | Vaddr      | Bind | Type | Name                                               |
|-----|------------|------|------|----------------------------------------------------|
| 1   | 0x10014000 | NONE | FUNC | ADVAPI32.dll\_RegQueryValueExA                      |
| 2   | 0x10014004 | NONE | FUNC | ADVAPI32.dll\_RegOpenKeyExA                         |
| 3   | 0x10014008 | NONE | FUNC | ADVAPI32.dll\_RegSetValueExA                        |
| 4   | 0x1001400c | NONE | FUNC | ADVAPI32.dll\_RegCreateKeyA                         |
| 5   | 0x10014010 | NONE | FUNC | ADVAPI32.dll\_RegCloseKey                           |
| 1   | 0x10014018 | NONE | FUNC | KERNEL32.dll\_Sleep                                 |
| 2   | 0x1001401c | NONE | FUNC | KERNEL32.dll\_GetModuleHandleA                      |
| 3   | 0x10014020 | NONE | FUNC | KERNEL32.dll\_CopyFileA                             |
| 4   | 0x10014024 | NONE | FUNC | KERNEL32.dll\_GetModuleFileNameA                    |
| 5   | 0x10014028 | NONE | FUNC | KERNEL32.dll\_GetLastError                          |
| 6   | 0x1001402c | NONE | FUNC | KERNEL32.dll\_GetModuleHandleExA                    |
| 7   | 0x10014030 | NONE | FUNC | KERNEL32.dll\_AllocConsole                          |
| 8   | 0x10014034 | NONE | FUNC | KERNEL32.dll\_CreateFileW                           |
| 9   | 0x10014038 | NONE | FUNC | KERNEL32.dll\_GetStringTypeW                        |
| 10  | 0x1001403c | NONE | FUNC | KERNEL32.dll\_LCMapStringW                          |
| 11  | 0x10014040 | NONE | FUNC | KERNEL32.dll\_HeapSize                              |
| 12  | 0x10014044 | NONE | FUNC | KERNEL32.dll\_HeapAlloc                             |
| 13  | 0x10014048 | NONE | FUNC | KERNEL32.dll\_EnterCriticalSection                  |
| 14  | 0x1001404c | NONE | FUNC | KERNEL32.dll\_LeaveCriticalSection                  |
| 15  | 0x10014050 | NONE | FUNC | KERNEL32.dll\_GetCurrentThreadId                    |
| 16  | 0x10014054 | NONE | FUNC | KERNEL32.dll\_DecodePointer                         |
| 17  | 0x10014058 | NONE | FUNC | KERNEL32.dll\_GetCommandLineA                       |
| 18  | 0x1001405c | NONE | FUNC | KERNEL32.dll\_HeapFree                              |
| 19  | 0x10014060 | NONE | FUNC | KERNEL32.dll\_CloseHandle                           |
| 20  | 0x10014064 | NONE | FUNC | KERNEL32.dll\_UnhandledExceptionFilter              |
| 21  | 0x10014068 | NONE | FUNC | KERNEL32.dll\_SetUnhandledExceptionFilter           |
| 22  | 0x1001406c | NONE | FUNC | KERNEL32.dll\_IsDebuggerPresent                     |
| 23  | 0x10014070 | NONE | FUNC | KERNEL32.dll\_EncodePointer                         |
| 24  | 0x10014074 | NONE | FUNC | KERNEL32.dll\_TerminateProcess                      |
| 25  | 0x10014078 | NONE | FUNC | KERNEL32.dll\_GetCurrentProcess                     |
| 26  | 0x1001407c | NONE | FUNC | KERNEL32.dll\_SetHandleCount                        |
| 27  | 0x10014080 | NONE | FUNC | KERNEL32.dll\_GetStdHandle                          |
| 28  | 0x10014084 | NONE | FUNC | KERNEL32.dll\_InitializeCriticalSectionAndSpinCount |
| 29  | 0x10014088 | NONE | FUNC | KERNEL32.dll\_GetFileType                           |
| 30  | 0x1001408c | NONE | FUNC | KERNEL32.dll\_GetStartupInfoW                       |
| 31  | 0x10014090 | NONE | FUNC | KERNEL32.dll\_DeleteCriticalSection                 |
| 32  | 0x10014094 | NONE | FUNC | KERNEL32.dll\_RtlUnwind                             |
| 33  | 0x10014098 | NONE | FUNC | KERNEL32.dll\_IsProcessorFeaturePresent             |
| 34  | 0x1001409c | NONE | FUNC | KERNEL32.dll\_GetProcAddress                        |
| 35  | 0x100140a0 | NONE | FUNC | KERNEL32.dll\_GetModuleHandleW                      |
| 36  | 0x100140a4 | NONE | FUNC | KERNEL32.dll\_ExitProcess                           |
| 37  | 0x100140a8 | NONE | FUNC | KERNEL32.dll\_WriteFile                             |
| 38  | 0x100140ac | NONE | FUNC | KERNEL32.dll\_GetModuleFileNameW                    |
| 39  | 0x100140b0 | NONE | FUNC | KERNEL32.dll\_HeapCreate                            |
| 40  | 0x100140b4 | NONE | FUNC | KERNEL32.dll\_HeapDestroy                           |
| 41  | 0x100140b8 | NONE | FUNC | KERNEL32.dll\_TlsAlloc                              |
| 42  | 0x100140bc | NONE | FUNC | KERNEL32.dll\_TlsGetValue                           |
| 43  | 0x100140c0 | NONE | FUNC | KERNEL32.dll\_TlsSetValue                           |
| 44  | 0x100140c4 | NONE | FUNC | KERNEL32.dll\_TlsFree                               |
| 45  | 0x100140c8 | NONE | FUNC | KERNEL32.dll\_InterlockedIncrement                  |
| 46  | 0x100140cc | NONE | FUNC | KERNEL32.dll\_SetLastError                          |
| 47  | 0x100140d0 | NONE | FUNC | KERNEL32.dll\_InterlockedDecrement                  |
| 48  | 0x100140d4 | NONE | FUNC | KERNEL32.dll\_FreeEnvironmentStringsW               |
| 49  | 0x100140d8 | NONE | FUNC | KERNEL32.dll\_WideCharToMultiByte                   |
| 50  | 0x100140dc | NONE | FUNC | KERNEL32.dll\_GetEnvironmentStringsW                |
| 51  | 0x100140e0 | NONE | FUNC | KERNEL32.dll\_QueryPerformanceCounter               |
| 52  | 0x100140e4 | NONE | FUNC | KERNEL32.dll\_GetTickCount                          |
| 53  | 0x100140e8 | NONE | FUNC | KERNEL32.dll\_GetCurrentProcessId                   |
| 54  | 0x100140ec | NONE | FUNC | KERNEL32.dll\_GetSystemTimeAsFileTime               |
| 55  | 0x100140f0 | NONE | FUNC | KERNEL32.dll\_SetStdHandle                          |
| 56  | 0x100140f4 | NONE | FUNC | KERNEL32.dll\_GetConsoleCP                          |
| 57  | 0x100140f8 | NONE | FUNC | KERNEL32.dll\_GetConsoleMode                        |
| 58  | 0x100140fc | NONE | FUNC | KERNEL32.dll\_FlushFileBuffers                      |
| 59  | 0x10014100 | NONE | FUNC | KERNEL32.dll\_CreateFileA                           |
| 60  | 0x10014104 | NONE | FUNC | KERNEL32.dll\_LoadLibraryW                          |
| 61  | 0x10014108 | NONE | FUNC | KERNEL32.dll\_GetCPInfo                             |
| 62  | 0x1001410c | NONE | FUNC | KERNEL32.dll\_GetACP                                |
| 63  | 0x10014110 | NONE | FUNC | KERNEL32.dll\_GetOEMCP                              |
| 64  | 0x10014114 | NONE | FUNC | KERNEL32.dll\_IsValidCodePage                       |
| 65  | 0x10014118 | NONE | FUNC | KERNEL32.dll\_HeapReAlloc                           |
| 66  | 0x1001411c | NONE | FUNC | KERNEL32.dll\_WriteConsoleW                         |
| 67  | 0x10014120 | NONE | FUNC | KERNEL32.dll\_MultiByteToWideChar                   |
| 68  | 0x10014124 | NONE | FUNC | KERNEL32.dll\_SetFilePointer                        |
| 69  | 0x10014128 | NONE | FUNC | KERNEL32.dll\_SetEndOfFile                          |
| 70  | 0x1001412c | NONE | FUNC | KERNEL32.dll\_GetProcessHeap                        |
| 71  | 0x10014130 | NONE | FUNC | KERNEL32.dll\_ReadFile                              |
| 1   | 0x10014138 | NONE | FUNC | USER32.dll\_ShowWindow                              |
| 2   | 0x1001413c | NONE | FUNC | USER32.dll\_GetAsyncKeyState                        |
| 3   | 0x10014140 | NONE | FUNC | USER32.dll\_GetWindowLongA                          |
| 4   | 0x10014144 | NONE | FUNC | USER32.dll\_DialogBoxIndirectParamW                 |
| 5   | 0x10014148 | NONE | FUNC | USER32.dll\_EndDialog                               |
| 6   | 0x1001414c | NONE | FUNC | USER32.dll\_FindWindowA                             |

Tenemos básicamente 3 bibliotecas dinámicas de las que importamos funciones: user32.dll, kernel32.dll y advapi32.dll. Decidí que para este writeup tengamos un enfoque mas estático de lo que a continuación sucede, ya que en un análisis mas tradicional, continuaríamos por la parte dinámica.

Comenzamos por llamar al shorcut de análisis:

![AAAA](/img/flareon2014-c5/aaaa.png)

Ok, con el análisis completado, veamos que funciones pudimos recuperar:

![afl](/img/flareon2014-c5/afl.png)

Como el resultado de afl era muy extenso, decidí convertirlo en tabla:

| address    | nbbs | size         | name                                                       |
|------------|------|--------------|------------------------------------------------------------|
| 0x10001000 | 4    | 85           | sub.KERNEL32.dll\_Sleep\_0                                   |
| 0x10001060 | 1    | 425          | fcn.10001060                                               |
| 0x10001240 | 1    | 1024         | fcn.10001240                                               |
| 0x10009340 | 1    | 15           | fcn.10009340                                               |
| 0x100093b0 | 1    | 10           | fcn.100093b0                                               |
| 0x100093c0 | 1    | 15           | fcn.100093c0                                               |
| 0x100093d0 | 1    | 15           | fcn.100093d0                                               |
| 0x100093e0 | 1    | 15           | fcn.100093e0                                               |
| 0x100093f0 | 1    | 15           | fcn.100093f0                                               |
| 0x10009400 | 1    | 15           | fcn.10009400                                               |
| 0x10009410 | 1    | 15           | fcn.10009410                                               |
| 0x10009420 | 1    | 15           | fcn.10009420                                               |
| 0x10009430 | 1    | 15           | fcn.10009430                                               |
| 0x10009440 | 6    | 77           | fcn.10009440                                               |
| 0x10009490 | 1    | 15           | fcn.10009490                                               |
| 0x100094a0 | 1    | 15           | fcn.100094a0                                               |
| 0x100094b0 | 1    | 15           | fcn.100094b0                                               |
| 0x100094c0 | 1    | 15           | fcn.100094c0                                               |
| 0x100094d0 | 6    | 77           | fcn.100094d0                                               |
| 0x10009520 | 1    | 15           | fcn.10009520                                               |
| 0x10009530 | 1    | 15           | fcn.10009530                                               |
| 0x10009540 | 1    | 15           | fcn.10009540                                               |
| 0x10009550 | 1    | 15           | fcn.10009550                                               |
| 0x10009560 | 1    | 15           | fcn.10009560                                               |
| 0x10009570 | 1    | 15           | fcn.10009570                                               |
| 0x10009580 | 1    | 15           | fcn.10009580                                               |
| 0x10009590 | 1    | 15           | fcn.10009590                                               |
| 0x100095a0 | 1    | 15           | fcn.100095a0                                               |
| 0x100095b0 | 1    | 15           | fcn.100095b0                                               |
| 0x100097d0 | 8    | 108          | fcn.100097d0                                               |
| 0x10009840 | 1    | 15           | fcn.10009840                                               |
| 0x10009850 | 4    | 46           | fcn.10009850                                               |
| 0x10009880 | 10   | 139          | fcn.10009880                                               |
| 0x10009910 | 6    | 77           | fcn.10009910                                               |
| 0x10009960 | 4    | 46           | fcn.10009960                                               |
| 0x10009990 | 8    | 108          | fcn.10009990                                               |
| 0x10009a00 | 4    | 46           | fcn.10009a00                                               |
| 0x10009a30 | 4    | 46           | fcn.10009a30                                               |
| 0x10009a60 | 1    | 15           | fcn.10009a60                                               |
| 0x10009a70 | 4    | 46           | fcn.10009a70                                               |
| 0x10009aa0 | 6    | 77           | fcn.10009aa0                                               |
| 0x10009af0 | 3    | 29           | fcn.10009af0                                               |
| 0x10009b10 | 6    | 77           | fcn.10009b10                                               |
| 0x10009b60 | 12   | 173          | fcn.10009b60                                               |
| 0x10009c10 | 1    | 15           | fcn.10009c10                                               |
| 0x10009c20 | 1    | 15           | fcn.10009c20                                               |
| 0x10009c30 | 8    | 108          | fcn.10009c30                                               |
| 0x10009ca0 | 4    | 46           | fcn.10009ca0                                               |
| 0x10009cd0 | 12   | 173          | fcn.10009cd0                                               |
| 0x10009d80 | 4    | 46           | fcn.10009d80                                               |
| 0x10009db0 | 1    | 15           | fcn.10009db0                                               |
| 0x10009dc0 | 1    | 15           | fcn.10009dc0                                               |
| 0x10009dd0 | 1    | 15           | fcn.10009dd0                                               |
| 0x10009de0 | 1    | 15           | fcn.10009de0                                               |
| 0x10009e30 | 1    | 10           | fcn.10009e30                                               |
| 0x10009e40 | 1    | 10           | fcn.10009e40                                               |
| 0x10009e50 | 1    | 10           | sub.RETURN\_e50                                             |
| 0x10009e60 | 1    | 10           | sub.BACKSPACE\_e60                                          |
| 0x10009e70 | 1    | 10           | sub.TAB\_e70                                                |
| 0x10009e80 | 1    | 10           | sub.CTRL\_e80                                               |
| 0x10009e90 | 1    | 10           | sub.DELETE\_e90                                             |
| 0x10009ea0 | 1    | 10           | sub.CAPS\_LOCK\_ea0                                          |
| 0x10009eb0 | 114  | 1274-> 1125  | sub.USER32.dll\_GetAsyncKeyState\_eb0                        |
| 0x1000a4c0 | 9    | 162          | sub.KERNEL32.dll\_Sleep\_4c0                                 |
| 0x1000a570 | 9    | 157          | sub.ADVAPI32.dll\_RegOpenKeyExA\_570                         |
| 0x1000a610 | 6    | 103          | sub.SOFTWARE\_\_Microsoft\_\_Windows\_\_CurrentVersion\_\_Run\_610  |
| 0x1000a680 | 5    | 275          | sub.KERNEL32.dll\_AllocConsole\_680                          |
| 0x1000a793 | 9    | 109          | fcn.1000a793                                               |
| 0x1000a800 | 7    | 105          | fcn.1000a800                                               |
| 0x1000a86c | 1    | 8            | fcn.1000a86c                                               |
| 0x1000a874 | 16   | 251          | fcn.1000a874                                               |
| 0x1000a972 | 1    | 8            | fcn.1000a972                                               |
| 0x1000a97a | 11   | 178          | fcn.1000a97a                                               |
| 0x1000aa2c | 1    | 10           | fcn.1000aa2c                                               |
| 0x1000aa36 | 1    | 23           | fcn.1000aa36                                               |
| 0x1000aa4d | 3    | 33           | fcn.1000aa4d                                               |
| 0x1000aa6e | 3    | 15   -> 277  | fcn.1000aa6e                                               |
| 0x1000aa80 | 4    | 43           | fcn.1000aa80                                               |
| 0x1000aab0 | 33   | 6722 -> 305  | fcn.1000aab0                                               |
| 0x1000ab2a | 16   | 148          | sub.KERNEL32.dll\_HeapAlloc\_b2a                             |
| 0x1000abbe | 1    | 33           | fcn.1000abbe                                               |
| 0x1000abe0 | 14   | 139          | fcn.1000abe0                                               |
| 0x1000ac6b | 19   | 258          | fcn.1000ac6b                                               |
| 0x1000ad6d | 1    | 10           | fcn.1000ad6d                                               |
| 0x1000ad77 | 1    | 6            | fcn.1000ad77                                               |
| 0x1000ae4e | 5    | 65           | sub.KERNEL32.dll\_EnterCriticalSection\_e4e                  |
| 0x1000ae8f | 3    | 50           | sub.KERNEL32.dll\_EnterCriticalSection\_e8f                  |
| 0x1000aec1 | 4    | 60           | sub.KERNEL32.dll\_LeaveCriticalSection\_ec1                  |
| 0x1000aefd | 3    | 47           | sub.KERNEL32.dll\_LeaveCriticalSection\_efd                  |
| 0x1000af2c | 28   | 356  -> 332  | sub.KERNEL32.dll\_GetCommandLineA\_f2c                       |
| 0x1000b005 | 4    | 20           | fcn.1000b005                                               |
| 0x1000b090 | 23   | 246  -> 226  | fcn.1000b090                                               |
| 0x1000b186 | 3    | 35           | entry0                                                     |
| 0x1000b1a9 | 4    | 58           | sub.KERNEL32.dll\_HeapFree\_1a9                              |
| 0x1000b1e3 | 13   | 156          | sub.KERNEL32.dll\_CloseHandle\_1e3                           |
| 0x1000b27f | 12   | 185          | fcn.1000b27f                                               |
| 0x1000b33b | 1    | 8            | fcn.1000b33b                                               |
| 0x1000b343 | 3    | 38           | fcn.1000b343                                               |
| 0x1000b369 | 4    | 49           | fcn.1000b369                                               |
| 0x1000b39a | 9    | 104          | fcn.1000b39a                                               |
| 0x1000b402 | 8    | 72           | fcn.1000b402                                               |
| 0x1000b44a | 17   | 209  -> 187  | fcn.1000b44a                                               |
| 0x1000b4ec | 1    | 17           | fcn.1000b4ec                                               |
| 0x1000b51b | 1    | 9            | fcn.1000b51b                                               |
| 0x1000b524 | 1    | 9            | fcn.1000b524                                               |
| 0x1000b52d | 1    | 15           | fcn.1000b52d                                               |
| 0x1000b53c | 7    | 297          | sub.KERNEL32.dll\_IsDebuggerPresent\_53c                     |
| 0x1000b665 | 1    | 37           | sub.KERNEL32.dll\_GetCurrentProcess\_665                     |
| 0x1000b68a | 3    | 45           | sub.KERNEL32.dll\_DecodePointer\_68a                         |
| 0x1000b6b7 | 1    | 16           | fcn.1000b6b7                                               |
| 0x1000b6c7 | 7    | 66           | fcn.1000b6c7                                               |
| 0x1000b709 | 3    | 19           | fcn.1000b709                                               |
| 0x1000b71c | 3    | 19           | fcn.1000b71c                                               |
| 0x1000b72f | 1    | 35           | fcn.1000b72f                                               |
| 0x1000b760 | 1    | 69           | fcn.1000b760                                               |
| 0x1000b7a5 | 1    | 20           | fcn.1000b7a5                                               |
| 0x1000b94f | 13   | 156          | fcn.1000b94f                                               |
| 0x1000b9eb | 5    | 52           | fcn.1000b9eb                                               |
| 0x1000ba1f | 37   | 343          | fcn.1000ba1f                                               |
| 0x1000bb76 | 49   | 581          | sub.KERNEL32.dll\_GetStartupInfoW\_b76                       |
| 0x1000bdbb | 10   | 83           | sub.KERNEL32.dll\_DeleteCriticalSection\_dbb                 |
| 0x1000be0e | 74   | 663          | sub.UTF\_8\_e0e                                              |
| 0x1000c0a5 | 18   | 295          | sub.KERNEL32.dll\_InitializeCriticalSectionAndSpinCount\_a5  |
| 0x1000c1cf | 1    | 9            | fcn.1000c1cf                                               |
| 0x1000c1e0 | 7    | 144          | fcn.1000c1e0                                               |
| 0x1000c2d2 | 1    | 23           | fcn.1000c2d2                                               |
| 0x1000c2e9 | 1    | 25           | fcn.1000c2e9                                               |
| 0x1000c302 | 1    | 25           | fcn.1000c302                                               |
| 0x1000c31b | 1    | 23           | fcn.1000c31b                                               |
| 0x1000c332 | 3    | 262          | loc.1000c332                                               |
| 0x1000c502 | 4    | 43           | sub.mscoree.dll\_502                                        |
| 0x1000c52d | 1    | 23           | sub.KERNEL32.dll\_ExitProcess\_52d                           |
| 0x1000c545 | 1    | 9            | fcn.1000c545                                               |
| 0x1000c54e | 1    | 9            | fcn.1000c54e                                               |
| 0x1000c557 | 1    | 51           | fcn.1000c557                                               |
| 0x1000c58a | 7    | 36           | fcn.1000c58a                                               |
| 0x1000c5ae | 13   | 151          | fcn.1000c5ae                                               |
| 0x1000c645 | 26   | 320          | sub.KERNEL32.dll\_DecodePointer\_645                         |
| 0x1000c785 | 1    | 22           | fcn.1000c785                                               |
| 0x1000c79b | 1    | 15           | fcn.1000c79b                                               |
| 0x1000c7aa | 1    | 30           | fcn.1000c7aa                                               |
| 0x1000c7c8 | 5    | 38           | fcn.1000c7c8                                               |
| 0x1000c7ee | 23   | 431          | sub.Runtime\_Error\_\_\_\_\_Program:\_7ee                         |
| 0x1000c99d | 5    | 57           | fcn.1000c99d                                               |
| 0x1000c9d6 | 1    | 30           | sub.KERNEL32.dll\_HeapCreate\_9d6                            |
| 0x1000c9f4 | 1    | 20           | sub.KERNEL32.dll\_HeapDestroy\_9f4                           |
| 0x1000ca08 | 1    | 15           | fcn.1000ca08                                               |
| 0x1000ca17 | 4    | 40           | sub.KERNEL32.dll\_DecodePointer\_a17                         |
| 0x1000ca3f | 1    | 9            | sub.KERNEL32.dll\_EncodePointer\_a3f                         |
| 0x1000ca51 | 3    | 52           | sub.KERNEL32.dll\_TlsGetValue\_a51                           |
| 0x1000ca85 | 16   | 5010 -> 148  | sub.KERNEL32.dll\_DecodePointer\_a85                         |
| 0x1000cac2 | 3    | 156          | sub.KERNEL32.DLL\_ac2                                       |
| 0x1000cb64 | 1    | 9            | fcn.1000cb64                                               |
| 0x1000cb6d | 1    | 9            | fcn.1000cb6d                                               |
| 0x1000cb76 | 6    | 121          | sub.KERNEL32.dll\_GetLastError\_b76                          |
| 0x1000cbef | 3    | 26           | fcn.1000cbef                                               |
| 0x1000cc09 | 28   | 279          | sub.KERNEL32.dll\_InterlockedDecrement\_c09                  |
| 0x1000cd23 | 1    | 9            | fcn.1000cd23                                               |
| 0x1000cd2f | 1    | 9            | fcn.1000cd2f                                               |
| 0x1000cd38 | 9    | 110          | sub.KERNEL32.dll\_TlsGetValue\_d38                           |
| 0x1000cda6 | 17   | 379          | sub.KERNEL32.DLL\_da6                                       |
| 0x1000cf21 | 11   | 135          | fcn.1000cf21                                               |
| 0x1000cfa8 | 8    | 51           | fcn.1000cfa8                                               |
| 0x1000cfdb | 11   | 116          | fcn.1000cfdb                                               |
| 0x1000d04f | 234  | 2953         | sub.KERNEL32.dll\_DecodePointer\_4f                          |
| 0x1000dbfb | 7    | 69           | sub.KERNEL32.dll\_Sleep\_bfb                                 |
| 0x1000dc40 | 7    | 76           | sub.KERNEL32.dll\_Sleep\_c40                                 |
| 0x1000dc8c | 8    | 78           | sub.KERNEL32.dll\_Sleep\_c8c                                 |
| 0x1000dcda | 10   | 147          | sub.KERNEL32.dll\_DeleteCriticalSection\_cda                 |
| 0x1000dd6d | 1    | 9            | fcn.1000dd6d                                               |
| 0x1000dd76 | 7    | 74           | sub.KERNEL32.dll\_InitializeCriticalSectionAndSpinCount\_d76 |
| 0x1000de17 | 1    | 23           | sub.KERNEL32.dll\_LeaveCriticalSection\_e17                  |
| 0x1000de2e | 13   | 185          | sub.KERNEL32.dll\_InitializeCriticalSectionAndSpinCount\_e2e |
| 0x1000dee7 | 1    | 9            | fcn.1000dee7                                               |
| 0x1000def0 | 4    | 51           | sub.KERNEL32.dll\_EnterCriticalSection\_ef0                  |
| 0x1000df23 | 21   | 220          | fcn.1000df23                                               |
| 0x1000dfff | 61   | 410          | fcn.1000dfff                                               |
| 0x1000e199 | 12   | 187          | sub.KERNEL32.dll\_GetModuleFileNameA\_199                    |
| 0x1000e254 | 13   | 151          | sub.KERNEL32.dll\_GetEnvironmentStringsW\_254                |
| 0x1000e2eb | 5    | 38           | fcn.1000e2eb                                               |
| 0x1000e337 | 40   | 330          | fcn.1000e337                                               |
| 0x1000e481 | 3    | 32           | fcn.1000e481                                               |
| 0x1000e4a1 | 9    | 155          | sub.KERNEL32.dll\_GetSystemTimeAsFileTime\_4a1               |
| 0x1000e53c | 14   | 129          | sub.KERNEL32.dll\_SetStdHandle\_53c                          |
| 0x1000e5bd | 15   | 134          | sub.KERNEL32.dll\_SetStdHandle\_5bd                          |
| 0x1000e643 | 8    | 105          | fcn.1000e643                                               |
| 0x1000e6ac | 9    | 145          | main                                                       |
| 0x1000e742 | 1    | 9            | fcn.1000e742                                               |
| 0x1000e74b | 1    | 39           | sub.KERNEL32.dll\_LeaveCriticalSection\_74b                  |
| 0x1000e772 | 29   | 400  -> 385  | sub.KERNEL32.dll\_InitializeCriticalSectionAndSpinCount\_772 |
| 0x1000e844 | 1    | 9            | fcn.1000e844                                               |
| 0x1000e902 | 1    | 9            | fcn.1000e902                                               |
| 0x1000e90b | 99   | 1789         | sub.KERNEL32.dll\_GetConsoleMode\_90b                        |
| 0x1000f008 | 12   | 201          | fcn.1000f008                                               |
| 0x1000f0d4 | 1    | 8            | fcn.1000f0d4                                               |
| 0x1000f0dc | 16   | 206          | sub.KERNEL32.dll\_FlushFileBuffers\_dc                       |
| 0x1000f1ad | 1    | 8            | fcn.1000f1ad                                               |
| 0x1000f1b5 | 1    | 8            | fcn.1000f1b5                                               |
| 0x1000f1c0 | 4    | 53           | fcn.1000f1c0                                               |
| 0x1000f200 | 7    | 68           | fcn.1000f200                                               |
| 0x1000f250 | 3    | 139          | fcn.1000f250                                               |
| 0x1000f30c | 7    | 86           | fcn.1000f30c                                               |
| 0x1000f362 | 31   | 356          | fcn.1000f362                                               |
| 0x1000f4d0 | 56   | 10779 -> 668 | fcn.1000f4d0                                               |
| 0x1000f831 | 144  | 1844         | sub.KERNEL32.dll\_CreateFileA\_831                           |
| 0x1000ff65 | 8    | 145          | fcn.1000ff65                                               |
| 0x1000fffb | 5    | 46           | fcn.1000fffb                                               |
| 0x10010029 | 1    | 32           | fcn.10010029                                               |
| 0x10010049 | 59   | 516          | fcn.10010049                                               |
| 0x1001024d | 1    | 26           | fcn.1001024d                                               |
| 0x10010267 | 36   | 332          | fcn.10010267                                               |
| 0x100103b3 | 1    | 26           | fcn.100103b3                                               |
| 0x10010435 | 8    | 132          | fcn.10010435                                               |
| 0x100104e5 | 1    | 31           | fcn.100104e5                                               |
| 0x10010504 | 1    | 3            | fcn.10010504                                               |
| 0x10010540 | 1    | 17           | sub.KERNEL32.dll\_EncodePointer\_540                         |
| 0x10010551 | 1    | 30           | fcn.10010551                                               |
| 0x1001056f | 7    | 55           | fcn.1001056f                                               |
| 0x100105a6 | 1    | 13           | sub.KERNEL32.dll\_DecodePointer\_5a6                         |
| 0x100105b3 | 43   | 419  -> 398  | sub.KERNEL32.dll\_DecodePointer\_5b3                         |
| 0x1001071a | 3    | 15           | fcn.1001071a                                               |
| 0x10010756 | 1    | 15           | fcn.10010756                                               |
| 0x10010765 | 1    | 15           | fcn.10010765                                               |
| 0x10010774 | 13   | 182          | sub.KERNEL32.dll\_DecodePointer\_774                         |
| 0x1001085b | 1    | 54           | fcn.1001085b                                               |
| 0x10010891 | 1    | 6            | fcn.10010891                                               |
| 0x10010897 | 1    | 23           | fcn.10010897                                               |
| 0x100108ae | 3    | 35           | sub.KERNEL32.dll\_EncodePointer\_8ae                         |
| 0x100108d1 | 23   | 364          | sub.USER32.DLL\_8d1                                         |
| 0x10010a3d | 16   | 117          | fcn.10010a3d                                               |
| 0x10010ab2 | 28   | 205          | fcn.10010ab2                                               |
| 0x10010b7f | 3    | 27           | fcn.10010b7f                                               |
| 0x10010b9a | 12   | 99           | fcn.10010b9a                                               |
| 0x10010bfd | 6    | 63           | fcn.10010bfd                                               |
| 0x10010c3c | 17   | 143          | sub.KERNEL32.dll\_InterlockedIncrement\_c3c                  |
| 0x10010ccb | 19   | 153          | sub.KERNEL32.dll\_InterlockedDecrement\_ccb                  |
| 0x10010d64 | 28   | 331          | fcn.10010d64                                               |
| 0x10010eaf | 10   | 77           | fcn.10010eaf                                               |
| 0x10010efc | 7    | 109          | fcn.10010efc                                               |
| 0x10010f69 | 1    | 12           | fcn.10010f69                                               |
| 0x10010f75 | 9    | 47           | fcn.10010f75                                               |
| 0x10010fa4 | 5    | 100          | fcn.10010fa4                                               |
| 0x10011008 | 26   | 400          | sub.KERNEL32.dll\_GetCPInfo\_8                               |
| 0x10011198 | 13   | 152          | sub.KERNEL32.dll\_InterlockedDecrement\_198                  |
| 0x10011233 | 1    | 9            | fcn.10011233                                               |
| 0x1001123c | 12   | 124          | sub.KERNEL32.dll\_GetOEMCP\_23c                              |
| 0x100112b8 | 37   | 489          | sub.KERNEL32.dll\_IsValidCodePage\_2b8                       |
| 0x100114a1 | 27   | 410  -> 399  | sub.KERNEL32.dll\_InterlockedDecrement\_4a1                  |
| 0x10011602 | 1    | 9            | fcn.10011602                                               |
| 0x1001163b | 3    | 30           | fcn.1001163b                                               |
| 0x10011659 | 1    | 22           | fcn.10011659                                               |
| 0x1001166f | 35   | 341          | sub.KERNEL32.dll\_WideCharToMultiByte\_66f                   |
| 0x100117c4 | 1    | 29           | fcn.100117c4                                               |
| 0x100117e1 | 3    | 56           | fcn.100117e1                                               |
| 0x10011819 | 1    | 19           | fcn.10011819                                               |
| 0x10011830 | 11   | 149          | fcn.10011830                                               |
| 0x100118c5 | 15   | 130          | sub.KERNEL32.dll\_HeapAlloc\_8c5                             |
| 0x10011947 | 18   | 173          | sub.KERNEL32.dll\_HeapReAlloc\_947                           |
| 0x100119f4 | 13   | 95           | fcn.100119f4                                               |
| 0x10011a53 | 9    | 83           | fcn.10011a53                                               |
| 0x10011aa6 | 1    | 24           | fcn.10011aa6                                               |
| 0x10011abe | 6    | 66           | sub.KERNEL32.dll\_WriteConsoleW\_abe                         |
| 0x10011b00 | 26   | 278          | sub.KERNEL32.dll\_MultiByteToWideChar\_b00                   |
| 0x10011c16 | 1    | 26           | fcn.10011c16                                               |
| 0x10011c30 | 8    | 133          | sub.KERNEL32.dll\_SetFilePointer\_c30                        |
| 0x10011cb5 | 12   | 224          | fcn.10011cb5                                               |
| 0x10011d95 | 1    | 10           | fcn.10011d95                                               |
| 0x10011d9f | 4    | 73           | fcn.10011d9f                                               |
| 0x10011eeb | 33   | 438          | sub.KERNEL32.dll\_GetProcessHeap\_eeb                        |
| 0x100120a1 | 137  | 1463         | sub.KERNEL32.dll\_ReadFile\_a1                               |
| 0x10012658 | 10   | 117          | sub.KERNEL32.dll\_SetFilePointer\_658                        |
| 0x100126cd | 13   | 187          | fcn.100126cd                                               |
| 0x10012788 | 3    | 45           | fcn.10012788                                               |
| 0x100127b5 | 21   | 226          | fcn.100127b5                                               |
| 0x10012897 | 7    | 2857 -> 83   | fcn.10012897                                               |
| 0x100128ea | 30   | 192          | fcn.100128ea                                               |
| 0x100129aa | 5    | 51           | fcn.100129aa                                               |
| 0x100129dd | 4    | 32           | fcn.100129dd                                               |
| 0x100129fd | 3    | 51           | sub.KERNEL32.dll\_HeapSize\_9fd                              |
| 0x10012a39 | 3    | 887          | fcn.10012a39                                               |
| 0x10012db0 | 12   | 105          | fcn.10012db0                                               |
| 0x10012e19 | 28   | 254          | fcn.10012e19                                               |
| 0x10012f17 | 47   | 487          | sub.KERNEL32.dll\_MultiByteToWideChar\_f17                   |
| 0x100130fe | 3    | 70           | fcn.100130fe                                               |
| 0x10013144 | 18   | 231          | sub.KERNEL32.dll\_MultiByteToWideChar\_144                   |
| 0x1001322b | 3    | 64           | fcn.1001322b                                               |
| 0x1001326b | 1    | 31           | sub.CONOUT\_26b                                             |
| 0x100132a1 | 19   | 277          | fcn.100132a1                                               |
| 0x100133c0 | 16   | 97           | fcn.100133c0                                               |
| 0x10013430 | 1    | 22           | fcn.10013430                                               |
| 0x1001345c | 13   | 184          | fcn.1001345c                                               |
| 0x10013674 | 1    | 6            | sub.KERNEL32.dll\_RtlUnwind\_674                             |

Si ordenamos esta tabla por tamaño, tomando únicamente los 10 primeros lugares tendríamos algo como lo siguiente:

| address    | nbbs | size | name                                 |
|------------|------|------|--------------------------------------|
| 0x1000d04f | 234  | 2953 | sub.KERNEL32.dll\_DecodePointer\_4f    |
| 0x1000f831 | 144  | 1844 | sub.KERNEL32.dll\_CreateFileA\_831     |
| 0x1000e90b | 99   | 1789 | sub.KERNEL32.dll\_GetConsoleMode\_90b  |
| 0x100120a1 | 137  | 1463 | sub.KERNEL32.dll\_ReadFile\_a1         |
| 0x10001240 | 1    | 1024 | fcn.10001240                         |
| 0x10012a39 | 3    | 887  | fcn.10012a39                         |
| 0x1000be0e | 74   | 663  | sub.UTF\_8\_e0e                        |
| 0x1000bb76 | 49   | 581  | sub.KERNEL32.dll\_GetStartupInfoW\_b76 |
| 0x10010049 | 59   | 516  | fcn.10010049                         |
| 0x100112b8 | 37   | 489  | sub.KERNEL32.dll\_IsValidCodePage\_2b8 |


Analicemos esas tres funciones muy grandes que no están identificadas:

> Se omiten las variables de inicio y se reduce el font para poder tener mayor visibilidad:

![pdf de 10001240](/img/flareon2014-c5/pdf-10001240.png)

Vaya vaya, parece que aquí se esta cargando algo en el stack... mas aun se nota el texto de Flare-On. Desgraciadamente pdf no logro identificar el tamaño completo de la función, pero eso se puede resolver ampliando el bloque:

![pd5552 de 10001240](/img/flareon2014-c5/pd5552-10001240.png)

Rango: 0x10001240 - 0x10009335

Esta función no recibe nada ni devuelve nada, pero lo que si es que algo hace porque podemos ver claramente en la imagen la palabra flare-on mientras se sube al stack. 

Ahora bien, este podría ser nuestro destino final en caso de éxito, es decir, el win msg. Tomare esto como valido y continuare viendo como llegar hasta aquí...

Para ello lo primero seria ver que funciones llaman a esta función, para ello ubicaremos cuales son las referencias cruzadas e iremos de función en función.

![xref 0x10001240](/img/flareon2014-c5/axg-10001240.png)

Como podemos observar, fnc 0x10009af0 es la primera y única función que llama a 0x10001240, veamos su contenido:

![pdf 0x10009af0](/img/flareon2014-c5/pdf-10009af0.png)

Tenemos una comparación a 0 en 0x100194fc, que en caso de ser mayor a 0, llama a la función winmsg, pero antes va a llamar a otra función, fcn 0x10001060. Veamos que hay en esta función previa:

![pd25 @ 0x10009af0](/img/flareon2014-c5/pd25-10001060.png)

Esta función como podemos observar se encarga de colocar las equivalencias de esas ubicaciones a 0, como alguna especie de controlador/limpiador. Observamos que tiene 91 referencias! He inclusive por su tamaño no pude extraer toda la función, pero debido a que solo se encarga de inicializar/limpiar (no recibe ni devuelve nada), decidí colocarlo en la siguiente tabla:

| #   | Address    | Value |
| --- | ---        | ---   |
| 01  | 0x10017000 | 1     |
| 02  | 0x10019460 | 0     |
| 03  | 0x10019464 | 0     |
| 04  | 0x10019468 | 0     |
| 05  | 0x1001946c | 0     |
| 06  | 0x10019470 | 0     |
| 07  | 0x10019474 | 0     |
| 08  | 0x10019478 | 0     |
| 09  | 0x1001947c | 0     |
| 10  | 0x10019480 | 0     |
| 11  | 0x10019484 | 0     |
| 12  | 0x10019488 | 0     |
| 13  | 0x1001948c | 0     |
| 14  | 0x10019490 | 0     |
| 15  | 0x10019494 | 0     |
| 16  | 0x10019498 | 0     |
| 17  | 0x1001949c | 0     |
| 18  | 0x100194a0 | 0     |
| 19  | 0x100194a4 | 0     |
| 20  | 0x100194a8 | 0     |
| 21  | 0x100194ac | 0     |
| 22  | 0x100194b0 | 0     |
| 23  | 0x100194b4 | 0     |
| 24  | 0x100194b8 | 0     |
| 25  | 0x100194bc | 0     |
| 26  | 0x100194c0 | 0     |
| 27  | 0x100194c4 | 0     |
| 28  | 0x100194c8 | 0     |
| 29  | 0x100194cc | 0     |
| 30  | 0x100194d0 | 0     |
| 31  | 0x100194d4 | 0     |
| 32  | 0x100194d8 | 0     |
| 33  | 0x100194dc | 0     | 
| 34  | 0x100194e0 | 0     |
| 35  | 0x100194e4 | 0     |
| 36  | 0x100194e8 | 0     |
| 37  | 0x100194ec | 0     |
| 38  | 0x100194f0 | 0     |
| 39  | 0x100194f4 | 0     |
| 40  | 0x100194f8 | 0     |
| 41  | 0x100194fc | 0     |
| 42  | 0x10019500 | 0     |

Prestando atención al penúltimo resultado, podemos observar que es el valor que hemos evaluado durante la comparación en fcn 0x10009af0, veamos de que manera si podemos escribirlo buscando la referencia hacia esta dirección:

![axt 0x100194fc](/img/flareon2014-c5/axt-100194fc.png)

Como podemos ver, estan las dos funciones que acabamos de explorar, mas una tercera, fcn 0x10009b60, que se encarga de almacenar un 1, justo como buscamos romper el if.

![pdf @ 0x10009b60](/img/flareon2014-c5/pdf-10009b60.png)

Tuve nuevamente que reducir el tamaño de fuente para poder tomar la captura. 

Bueno, como podemos apreciar, esta función tiene varias cosas diferentes y a la vez comunes con la función anterior. La primera cosa en común es que también es llamada desde sub.USER32.dll\_GetAsyncKeyState\_10009eb0, presenta un compare, luego set a 0 y finalmente un set a 1 para activar el siguiente flujo o brinco a función.

Su característica principal, es que pareciera que tenemos varios if concatenados, siendo precisos unos 5 cmp, que si ninguno tiene la flag activada, pone en 0 la flag actual y levanta la siguiente. Pero todos los caminos conducen a roma, es decir, todas las comparaciones al final devuelve "o".

La función anterior (0x10009af0) devolvía "m", por lo que sea lo que sea GetAsyncKeyState, esta recibiendo estos valores char cuando llama a las funciones.

[GetAsyncKeyState](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-getasynckeystate)
```C
SHORT GetAsyncKeyState(
  int vKey
);
```

Esta función básicamente recibe la tecla a evaluar y nos devuelve si fue o esta siendo presionada.

Ok, veamos que hace la función sub.USER32.dll\_GetAsyncKeyState\_10009eb0:

![pdf @ sub.USER32.dll\_GetAsyncKeyState\_10009eb0](/img/flareon2014-c5/pdf1-10009eb0.png)

El primer vistazo, nos revela muchos jumps, inclusive estamos hablando de un switch case, lo cual seria el tipo de loop adecuado para esto.

Las primeras 3 instrucciones corresponden al stack frame de 8 direcciones, las siguientes 11 instrucciones, desde _mov eax, 8_ hasta _cmp edx, 0xde_ son las condicionales que nos haran mantenernos dentro de un loop de switch case, básicamente desde 8 hasta 222.

![pdf @ sub.USER32.dll\_GetAsyncKeyState\_10009eb0](/img/flareon2014-c5/pdf2-10009eb0.png)

Como podemos ver, cuando se rompe el loop, se retorna 0 y termina la función.

Veamos que sucede dentro de este switch:

![pdf @ sub.USER32.dll\_GetAsyncKeyState\_10009eb0](/img/flareon2014-c5/pdf3-10009eb0.png)

Después de la evaluación inicial, el counter es pasado como argumento a GetAsyncKeyState, es decir si local\_4h vale 0x2b, se evaluara si la tecla '\*' fue presionada. Ese valor es compartido (tecla presionada ahora o antes) y se compara en el switch, si es no es igual, se va al caso ultimo, si es igual llama a la función correspondiente y brinca hacia el final (0x1000a3a6). En pocas palabras si la tecla "A" o la que sea es presionada, su función correspondiente es llamada.

Esto implica que nosotros controlamos el valor de las flags, presionándolas en el orden correcto debemos llegar a llamar a la función bandera (0x10001240).

Entonces ya que hemos entendido hagamos el reverso de cada letra presionada mediante el siguiente burdo script en python:

```python
#!/usr/bin/env python
  
import r2pipe, os

keys = ["0x10017000","0x10019460","0x10019464","0x10019468","0x1001946c","0x10019470","0x10019474","0x10019478","0x1001947c","0x10019480","0x10019484","0x10019488","0x1001948c","0x10019490","0x10019494","0x10019498","0x1001949c","0x100194a0","0x100194a4","0x100194a8","0x100194ac","0x100194b0","0x100194b4","0x100194b8","0x100194bc","0x100194c0","0x100194c4","0x100194c8","0x100194cc","0x100194d0","0x100194d4","0x100194d8","0x100194dc","0x100194e0","0x100194e4","0x100194e8","0x100194ec","0x100194f0","0x100194f4","0x100194f8","0x100194fc","0x10019500"]

flag = ""

if os.path.isfile("5get_it"):

    r2 = r2pipe.open("5get_it")
    r2.cmd("aaaa")

    for key in keys:
        xrefs = r2.cmdj("axtj " + key)
        for xref in xrefs:
            if xref["opcode"] == "mov dword [" + key + "], 1":
                fcn_called = r2.cmdj("pdfj@" + xref["fcn_name"])
                for eax_ret in fcn_called["ops"]:
                    if "mov eax, 0x100" in eax_ret["disasm"]:
                        flag += r2.cmd("pr 1 @ " + eax_ret["disasm"].split(",")[1])

r2.quit()
print flag
```

La salida del comando nos devuelve el string:

> l0ggingdoturdot5tr0ke5atflaredashondotco

La letra faltante, "m", se devuelve a la salida del la función de reseteo.

Validemos el DLL contra yara rules:

```
xbytemx@laptop:~/flare-on2014/challenge05$ yara ~/git/rules/Antidebug_AntiVM_index.yar 5get_it
anti_dbg 5get_it
keylogger 5get_it
win_registry 5get_it
win_files_operation 5get_it
```

Vemos que da positivo contra tecnicas de antidbg, keylogger, win\_registry y win\_files\_operation. Por ello crearemos un prefix de wine en donde ejecutemos con calma la dll.

Si ejecutamos el dll e ingresamos la salida del comando nos aparecerá el siguiente mensaje:

> wineconsole rundll32 5get\_it 

![wineconsole rundll32 5get\_it](/img/flareon2014-c5/wine-win.png)

> Nota: La dll se ejecuta en un prefix recien creado para ser eliminado despues.

Traduciendo al formato de la flag:

**flag: l0gging.ur.5tr0ke5@flare-on.com**

---

Espero que les haya gustado.

Saludos,
