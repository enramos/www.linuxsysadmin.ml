---
title: "Escribiendo un servidor DNS en python con twisted"
slug: "escribiendo-un-servidor-dns-en-python-con-twisted"
date: 2017-02-13
categories: ['Desarrollo']
tags: ['python', 'dns', 'twisted']
---

El otro día tuvimos una caída del centro de datos de desarrollo. Inmediatamente después vimos que teníamos afectación en el entorno de producción, ya que lanzaba peticiones al **DNS** de desarrollo. Sin saber claramente porque pasaba, hice un servidor **DNS** en **python**, para ver que tipos de peticiones se lanzaban.<!--more-->

Para hacerlo, utilicé una librería magnífica llamada **twisted**, que hace la mayoría del trabajo. Aunque su documentación es bastante escasa, tirando de ejemplos pude sacar algo interesante en poco tiempo.

## Un servidor chivato

Cambiamos el fichero */etc/resolv* de nuestro servidor de producción, para añadirle su propia dirección IP, en la que ejecutamos el siguiente *script*:

```python
#!/usr/bin/env python

from twisted.internet import reactor, defer
from twisted.names import client, dns, error, server


class DynamicResolver(object):
    def query(self, query, timeout=None):
        print query
        return defer.fail(error.DomainError())


def main():
    factory = server.DNSServerFactory(
        clients=[
            DynamicResolver(),
        ],
    )
    protocol = dns.DNSDatagramProtocol(controller=factory)
    reactor.listenUDP(10053, protocol)
    reactor.listenTCP(10053, factory)
    reactor.run()

if __name__ == '__main__':
    raise SystemExit(main())
```

Si os interesa trabajar con *virtualenv*, aquí os dejo el *requirements.txt*:

```bash
gerard@sirius:~/projects/dns$ cat requirements.txt 
Twisted==16.3.2
zope.interface==4.2.0
gerard@sirius:~/projects/dns$ 
```

Es el resultado de ejecutar *pip install twisted*.

Comprobamos lo que pasa cuando el servidor recibe peticiones:

```bash
gerard@sirius:~$ dig @localhost -p 10053 nowhere.com

; <<>> DiG 9.9.5-9+deb8u6-Debian <<>> @localhost -p 10053 nowhere.com
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 42644
;; flags: qr ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;nowhere.com.			IN	A

;; Query time: 1 msec
;; SERVER: 127.0.0.1#10053(127.0.0.1)
;; WHEN: Wed Aug 24 15:29:38 CEST 2016
;; MSG SIZE  rcvd: 29

gerard@sirius:~$ 
```

Y en el otro terminal:

```bash
gerard@sirius:~/projects/dns$ ./env/bin/python dns.py 
<Query nowhere.com A IN>

```

Este *script* implementa un servidor que no es capaz de encontrar el dominio, pero registra la petición recibida. Con el tipo de peticiones nos dimos cuenta de que había algo mal configurado en el entorno de producción.

## Añadiendo resolución de peticiones

La librería **twisted** ofrece clases para entender las peticiones y para formular las respuestas. Eso nos da una *toolbox* muy interesante para hacer nuestro propio servidor.

Supongamos que queremos asociar *example.com* a las direcciones *1.2.3.4* y *1.2.3.5*; adicionalmente asociaremos también el nombre *example.my* a la dirección *1.2.3.5*. Se podría utilizar una base de datos, pero vamos a no hacerlo para mantener el ejemplo pequeño.

```python
#!/usr/bin/env python

from twisted.internet import reactor, defer
from twisted.names import client, dns, error, server
from itertools import groupby

records = [
    ('example.com', '1.2.3.4'),
    ('example.com', '1.2.3.5'),
    ('example.my', '1.2.3.5'),
]

name2ip = dict((key, [e[1] for e in group]) for key, group in groupby(records, lambda x: x[0]))
ip2name = dict((key, [e[0] for e in group]) for key, group in groupby(records, lambda x: x[1]))


class DynamicResolver(object):
    def calculate_responses(self, query):
        if query.type == dns.A:
            records = name2ip.get(query.name.name, [])
            for record in records:
                yield dns.Record_A(address=record)
        if query.type == dns.PTR:
            aux = '.'.join(reversed(query.name.name.split('.')))[13:]
            records = ip2name.get(aux, [])
            for record in records:
                yield dns.Record_PTR(name=record)

    def query(self, query, timeout=None):
        responses = self.calculate_responses(query)
        answers = [
            dns.RRHeader(
                type=answer.TYPE,
                name=query.name.name,
                payload=answer,
            ) for answer in responses]
        authority = []
        additional = []
        return defer.succeed((answers, authority, additional))


def main():
    factory = server.DNSServerFactory(
        clients=[
            DynamicResolver(),
        ],
    )
    protocol = dns.DNSDatagramProtocol(controller=factory)
    reactor.listenUDP(10053, protocol)
    reactor.listenTCP(10053, factory)
    reactor.run()

if __name__ == '__main__':
    raise SystemExit(main())
```

Y si ejecutamos este nuevo *script* y le lanzamos peticiones, vemos que funciona.

```bash
gerard@sirius:~$ dig @localhost -p 10053 +short example.com
1.2.3.4
1.2.3.5
gerard@sirius:~$ dig @localhost -p 10053 +short example.my
1.2.3.5
gerard@sirius:~$ dig @localhost -p 10053 +short nowhere.com
gerard@sirius:~$ dig @localhost -p 10053 +short -x 1.2.3.4
example.com.
gerard@sirius:~$ dig @localhost -p 10053 +short -x 1.2.3.5
example.com.
example.my.
gerard@sirius:~$ dig @localhost -p 10053 +short -x 1.2.3.6
gerard@sirius:~$ 
```

Aunque ya hay servidores **DNS** magníficos, podemos levantar uno con **python** de forma fácil, rápida y sencilla.
