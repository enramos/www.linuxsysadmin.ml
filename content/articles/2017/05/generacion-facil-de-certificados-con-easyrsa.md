---
title: "Generación fácil de certificados con easyrsa"
slug: "generacion-facil-de-certificados-con-easyrsa"
date: 2017-05-29
categories: ['Sistemas']
tags: ['easyrsa', 'openssl', 'ssl', '2 way ssl', 'certificado']
---

Ya vimos en [otro artículo]({{< relref "/articles/2016/02/restringiendo-accesos-mediante-certificados-de-cliente.md" >}}) como restringir los accesos a una web usando certificados SSL. Sin embargo, la generación de los mismos era un poco confusa. Sin embargo, existe una herramienta llamada **easyrsa** que nos permite generar peticiones de forma fácil, firmarlas con nuestra CA y obtener el producto final.<!--more-->

Vamos a intentar seguir los pasos del citado artículo, solo en la parte de generación de los certificados. Si no se necesitara, se puede obviar la parte del certificado cliente, o porque no, generar certificados para decenas o cientos de usuarios de forma fácil.

Vamos a partir de una distribución [Alpine Linux](https://alpinelinux.org/). Para los que no lo sospechen ya es un contenedor **Docker** por la facilidad de crearlo y de destruirlo al acabar el artículo. Además, esta distribución nos ofrece la herramienta en la versión 3, que me ha parecido más intuitiva que la versión anterior. Asumo también que se dispone del paquete **easy-rsa**, que se puede instalar con `apk add easy-rsa`.

## Preparando la CA

lo primero para hacer una CA es copiar la estructura base a cualquier sitio. La idea es que tenemos una copia para cada CA que tengamos en el servidor, y no quiero trabajar en las carpetas de sistema para no destruir la plantilla.

```bash
rsa:~# cp -R /usr/share/easy-rsa/* .
rsa:~# 
```

Otro paso necesario es inicializar las estructura de PKI, que básicamente es crear un esqueleto de carpetas para contener nuestros ficheros.

```bash
rsa:~# easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /root/pki

rsa:~# 
```

Finalmente creamos los certificados y claves necesarios para la CA con un simple comando único.

```bash
rsa:~# easyrsa build-ca
Generating a 2048 bit RSA private key
........................+++
...............................................+++
writing new private key to '/root/pki/private/ca.key.XXXXPPMHOb'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/root/pki/ca.crt

rsa:~# 
```

El resultado que deberemos exportar a nuestro servidor es el certificado de la CA, cuya localización es */root/pki/ca.crt*.

## Generando el certificado de nuestro servidor

Este paso se debe repetir tantas veces como servidores queramos que utilicen un certificado SSL. De momento, nos basta con uno. Además, lo vamos a crear sin contraseña porque no queremos tener que introducirla cada vez que se reinicie el servidor.

```bash
rsa:~# easyrsa gen-req private nopass
Generating a 2048 bit RSA private key
......................................................+++
.+++
writing new private key to '/root/pki/private/private.key.XXXXhfGNmO'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [private]:

Keypair and certificate request completed. Your files are:
req: /root/pki/reqs/private.req
key: /root/pki/private/private.key

rsa:~# 
```

**IMPORTANTE**: el *Common Name* es el parámetro mas importante; debe coincidir con el dominio para que se dé por bueno.

Lo firmamos con nuestra CA y ya habremos acabado.

```bash
rsa:~# easyrsa sign-req server private


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 3650 days:

subject=
    commonName                = private


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
Using configuration from /root/openssl-1.0.cnf
Enter pass phrase for /root/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'private'
Certificate is to be certified until Nov  9 11:24:39 2026 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /root/pki/issued/private.crt

rsa:~# 
```

Vamos a necesitar los ficheros */root/pki/private/private.key* y */root/pki/private/private.crt* para ponerlos en nuestro servidor, juntamente con el certificado de la CA.

## Generando certificados cliente (opcional)

Se trata de la misma filosofía; generamos una *request*, la firmamos y finalmente la vamos a empaquetar en un fichero *.p12* para su fácil y segura distribución.

Repetimos la generación de la *request* de la misma manera:

```bash
rsa:~# easyrsa gen-req gerard
Generating a 2048 bit RSA private key
............................................+++
...........+++
writing new private key to '/root/pki/private/gerard.key.XXXXfHJDkE'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [gerard]:

Keypair and certificate request completed. Your files are:
req: /root/pki/reqs/gerard.req
key: /root/pki/private/gerard.key

rsa:~# 
```

Firmamos la petición. Es especialmente importante el parámetro *client*, ya que sino, no va a funcionar.

```bash
rsa:~# easyrsa sign-req client gerard


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 3650 days:

subject=
    commonName                = gerard


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
Using configuration from /root/openssl-1.0.cnf
Enter pass phrase for /root/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'gerard'
Certificate is to be certified until Nov  9 11:42:10 2026 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /root/pki/issued/gerard.crt

rsa:~# 
```

Finalmente tenemos los ficheros *gerard.key* y *gerard.crt*, que no son lo que solemos importar en nuestro navegador. Para ellos lo empaquetamos en un fichero *gerard.12* que está protegido por contraseña y es el que deberá importar el usuario de nuestra web.

```bash
rsa:~# openssl pkcs12 -export -in pki/issued/gerard.crt -inkey pki/private/gerard.key -out gerard.p12
Enter pass phrase for pki/private/gerard.key:
Enter Export Password:
Verifying - Enter Export Password:
rsa:~# 
```

## Un ejemplo de servidor nginx funcional

Tenemos 3 ficheros para nuestro servidor, que son *ca.crt*, *server.crt* y *server.key*, con los que podemos montar un dominio estándar, como en el artículo citado.

```bash
root@server:~# cat /etc/nginx/sites-enabled/private.linuxsysadmin.tk
server {
    listen                      443 ssl;
    server_name                 private;
    root                        /srv/www;

    ssl_certificate             /etc/ssl/certs/server.crt;
    ssl_certificate_key         /etc/ssl/private/server.key;
    ssl_client_certificate      /etc/ssl/certs/ca.crt;
    ssl_verify_client           on;
}
root@server:~#
```

Y suponiendo que el cliente ha importado su clave con éxito, ya tenemos el dominio montado.
