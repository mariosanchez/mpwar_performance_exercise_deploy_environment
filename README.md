#Ejercicio de web performance
##Diario de trabajo

En este ejercicio parto de un código base al que necesito aplicar una serie de 
mejoras que optimicen los tiempos de carga que tendría este proyecto de ejemplo en 
prodcución.

El código: https://github.com/fiunchinho/mpwar_performance_exercise

###Preparando el entorno de desarrollo y deployment

Lo primero que he hecho ha sido hacer un fork del proyecto en mi cuenta de github e
intentar entender que es lo que hace y como funciona.

Ahora necesito poder ponerlo en marcha. Como no tengo ningún entorno listo para 
correrlo he decidido crearme una máquina virtual en Vagrant provisionada mediante
un playbook de Ansible. Esto me irá bien para luego usar este mismo playbook en AWS.
En este caso la máquina es un Centos7. Es importante que en el caso de necesitar las 
Guest Additions de VirtualBox nos aseguremos de que la versión de kernel y kernel-devel
son las mismas, en caso de que no lo sean, lo que me ha funcionad es lanzar el comando
"yum updrade" en el guest y luego "vagrant reload" en el host para que se reinstalen
corretamente.

Como ya tengo playbooks hechos decido basarme en estos par tejer el nuevo haciendo
las modificaciones necesarias como añadir los paquetes que requiere el proyecto para 
funcionar de manera básica.

Cuando trabajamos con Ansible vía SSH contra un vagrant es importante recordar que en
archivo inventory/host de Ansible añadamos los parametros ansible_user (por defecto 
vagrant) y ansible_ssh_private_key_file (ruta relativa a las claves ssh partiendo del
directorio donde se encuentra nuestro playbook.yml).

Si queremos instalar versiones más modernas de PHP (por ejemplo 7 o 7.1) vamos a 
necesitar instalar el repositorio EPEL. Hemos creado un nuevo rol de Ansible que tiene
este cometido.

Además he hecho uso de dos roles externos, el de mysql y el de ansistrano. Hay que 
tener en cuenta que algunos de estos roles tienen una versión mínima de Ansible para
poder ser instalados, el de mysql de geerlinguy por ejemplo pide la versión 2.2, por 
lo que tenemos que comprobar esto en caso de error al ejecutar un playbook.

Para Ansistrano he creado un archivo de variables para su configuración y un script 
post deploy para que instale las dependencias de composer.

Como instalé la versión 7.1 de php he tenido que hacer un update de composer y subir 
el composer.json y el composer.lock al for para que ansistrano fuera capaz de hacer 
"composer install".

Puede parecer que ya está todo listo para ver, al menos, para visualizar la web que 
hemos deployadocon Ansistrano, pero hemos de asegurarnos de que nuestra configuración 
de base de datos está bien puesta en la configuración de Silex y también de que hemos 
creado las tablas de MySQL para que la aplicación funcione correctamente, de otra forma 
podemos perder mucho tiempo, literalmente, que es justo lo que he ha pasado a mí.

Una vez ya he podido ver la página de inicio de la aplicación, he necesitado crear un
document .htaccess para habilitar el 'RewriteEngine' de apache y que de ese modo me 
funcionen las rutas. Esto es un workarround teniendo en cuenta que lo ideal sería tener un
virutal host de apache con toda la configuración ahí, pero ya he invertido mucho tiempo
en la base de datos, así que, por el momento quedará así. Para que podamos leer los assets
como CSSs y JSs, he añadido posteriormente las directivas `RewriteCond %{REQUEST_FILENAME} !-d`
y `RewriteCond %{REQUEST_FILENAME} !-f`.

Para evitar tener que hacer virtual hosts he creado una copia en el del httpd.conf en ansible
con las modificaciones que he necesitad para que luego en el aprovisionamiento se reemplace por
el por defecto.


###Mejoras en la aplicación

####Bootstrap
Las primeras mejoras que he decidido abordar son las de bootstrap ya que creo que son las más
sencillas de hacer y que me llevarán menos tiempo.

Lo primero es hacer un poco de refactor en las vistas de twig para que se repita un poco menos 
de codigo haciendo uso de `include` y `extends`.

También he preparado la configuración de `prod.php` para que la dirección de los assets sea 
dinamica y se pueda cambiar fácilmente en el momento de implementar cloudfront.

Hecho esto he hecho referencia a los ficheros minificados de bootstrap desde la plantilla base.


####Redis

He añadido un nuevo role al playbook de ansible para instalar el servicio de Redis y así poder 
hacer uso de este para almacenar sessiones y caché, así como otros usos de funcionalidad (rankings).

Para facilitar el uso de Redis en Silex he decido instalar el paquete `predis/service-provider`.
Para el caso de las sessiones voy a usar un SessionHandler de Symfony, `snc/redis-bundle`, y para
almacenar datos de cache en Redis usaré el ServiceProvider `moust/silex-cache`.

He hecho pruebas tanto de caché como de sesiones y todo parece funcionar a la perfección. Para esto
he añadido el provider de caché al archivo `DomainServiceProvider` y en el archivo `app.php` he dado
de alta los clientes de Redis y he sobreescrito el session handler de la configuración de Silex para
que me funcionaran las sesiones.

Para el ranking de post más visitados he registrado otro cliente de Predis en el container de silex y 
he usado un stored set para conseguir el ranking incrementando las visitas en cada lecura de articulo
y recuperandolos en la home.

He tenido problemas de permisos con redis por lo que he necesitado desactivar una configuración de 
 selinux que daba problemas al intentar acceder a redis desde PHP con Centos7.


####Subida de imagenes a S3

Para la subida de las imagenes de perfil a S3 he decidido usar el vendor `league/flysystem-aws-s3-v3` 
que nos provee de un cliente de S3 listo para usar simplemente pasando los parametros de conexión 
(key y secret).

He creado un nuevo bucket en S3 para el ejercicio y he creado una nueva clave de accesso y secret desde
el panel de IAM de AWS.

Para que las imagenes que se suben al bucket fueran publicas por defecto he creado una nueva politica y la 
he añadido al bucket de tal forma que los usuarios pudieran ver las imagenes.

Finalmente he hecho la implementación el en el caso de uso y las modificaciones en base de datos para poder
persistir el enlace a la imagen.


####HTTP Cache 

He añadido las cabeceras de cache a la home y a la vista de articulo para que el browser cachee assets y 
contenido de pagina a menos que el usuario haga f5.

Finalmente he tenido que revertir algunos de los cambios ya que me daban probelmas con los listados de redis
y no he sabido resolverlo.


###Subida a EC3 y gestión del Load Balancer

Cuando ya he tenido todas las mejoras hechas en la aplicación decido hacer pruebas en AWS para ver si todo 
se aprovisiona correctamente.

Primero he creado una maquina de prueba con todos los servicios para ver si funcionaba y tras algunas 
modificaciones parece que todo va correctamente.

El siguiente paso es crear un balanceador de carga que distribuya el tráfico entre 2 frontales.

###Cloudfront
Para cloudfront simplemente he dado de alta una nueva distribución con los valores por defecto y he cambiado la 
url de CDN (que ya había preparado previamente) en el proyecto.


###Blackfire
Siguiendo la instrucciones de los compañeros, he desactivado SELINUX en los frontales y e instalado manualmente
el agente de blackfire. Tras esto el proceso de profiling ha ido a la perfección.

Para desactivar SELINUX:
```
vim /etc/sysconfig/selinux
```

Y seteamos `SELINUX=disabled`.


###Testing & Profiling

####Apache A/B


Commit 7a9f368c4f3468d6a79ee28aa701f70a4b04571b

Commit 0ce58969ff260afb1afcbd19510f4fca35333092
```
Completed 5000 requests
Completed 10000 requests
Completed 15000 requests
Completed 20000 requests
Completed 25000 requests
Completed 30000 requests
Completed 35000 requests
Completed 40000 requests
Completed 45000 requests
Finished 49982 requests


Server Software:        Apache/2.4.18
Server Hostname:        localhost
Server Port:            80

Document Path:          /web
Document Length:        304 bytes

Concurrency Level:      500
Time taken for tests:   33.451 seconds
Complete requests:      49982
Failed requests:        0
Non-2xx responses:      49985
Total transferred:      26242125 bytes
HTML transferred:       15195440 bytes
Requests per second:    1494.20 [#/sec] (mean)
Time per request:       334.628 [ms] (mean)
Time per request:       0.669 [ms] (mean, across all concurrent requests)
Transfer rate:          766.11 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        4   11  12.4     10    1010
Processing:     6   72 970.0     19   33416
Waiting:        3   68 970.2     15   33415
Total:         13   83 970.5     29   33435

Percentage of the requests served within a certain time (ms)
  50%     29
  66%     33
  75%     35
  80%     37
  90%     42
  95%     47
  98%     53
  99%     59
 100%  33435 (longest request)
```

Commit 7d096fe2de6fcbb46644d261499775bb8c84330f
```
Completed 5000 requests
Completed 10000 requests
Completed 15000 requests
Completed 20000 requests
Completed 25000 requests
Completed 30000 requests
Completed 35000 requests
Completed 40000 requests
Completed 45000 requests
Completed 50000 requests
Finished 50000 requests


Server Software:        Apache/2.4.18
Server Hostname:        localhost
Server Port:            80

Document Path:          /web
Document Length:        304 bytes

Concurrency Level:      500
Time taken for tests:   19.029 seconds
Complete requests:      50000
Failed requests:        0
Non-2xx responses:      50000
Total transferred:      26250000 bytes
HTML transferred:       15200000 bytes
Requests per second:    2627.54 [#/sec] (mean)
Time per request:       190.292 [ms] (mean)
Time per request:       0.381 [ms] (mean, across all concurrent requests)
Transfer rate:          1347.13 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   11 125.7      1    3006
Processing:     2   73 955.8     11   19013
Waiting:        1   73 955.8     10   19013
Total:          6   84 980.5     11   19026

Percentage of the requests served within a certain time (ms)
  50%     11
  66%     13
  75%     15
  80%     16
  90%     20
  95%     25
  98%    201
  99%   1007
 100%  19026 (longest request)
```

Commit bc999b74bf7b2d56ca799a6abcb81b140fccd891
```
Completed 5000 requests
Completed 10000 requests
Completed 15000 requests
Completed 20000 requests
Completed 25000 requests
Completed 30000 requests
Completed 35000 requests
Completed 40000 requests
Completed 45000 requests
Finished 49955 requests


Server Software:        Apache/2.4.18
Server Hostname:        localhost
Server Port:            80

Document Path:          /web
Document Length:        304 bytes

Concurrency Level:      500
Time taken for tests:   28.217 seconds
Complete requests:      49955
Failed requests:        0
Non-2xx responses:      49955
Total transferred:      26226375 bytes
HTML transferred:       15186320 bytes
Requests per second:    1770.36 [#/sec] (mean)
Time per request:       282.429 [ms] (mean)
Time per request:       0.565 [ms] (mean, across all concurrent requests)
Transfer rate:          907.65 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   11 154.3      0    3008
Processing:     1   39 585.7     10   28204
Waiting:        1   39 585.7     10   28204
Total:          5   50 632.4     10   28216

Percentage of the requests served within a certain time (ms)
  50%     10
  66%     12
  75%     14
  80%     15
  90%     17
  95%     21
  98%     27
  99%    216
 100%  28216 (longest request)

```

Contra el load balancer `ab -n 2000 -c 25 http://exercice-lb-365865535.eu-west-2.elb.amazonaws.com/`

```
Completed 200 requests
Completed 400 requests
Completed 600 requests
Completed 800 requests
Completed 1000 requests
Completed 1200 requests
Completed 1400 requests
Completed 1600 requests
Completed 1800 requests
Completed 2000 requests
Finished 2000 requests


Server Software:        Apache/2.4.6
Server Hostname:        exercice-lb-365865535.eu-west-2.elb.amazonaws.com
Server Port:            80

Document Path:          /
Document Length:        1914 bytes

Concurrency Level:      25
Time taken for tests:   94.748 seconds
Complete requests:      2000
Failed requests:        0
Total transferred:      4430000 bytes
HTML transferred:       3828000 bytes
Requests per second:    21.11 [#/sec] (mean)
Time per request:       1184.348 [ms] (mean)
Time per request:       47.374 [ms] (mean, across all concurrent requests)
Transfer rate:          45.66 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       43  138  91.9    118     596
Processing:   144 1046 301.6   1064    3853
Waiting:      142 1043 301.7   1062    3851
Total:        192 1184 333.2   1196    3955

Percentage of the requests served within a certain time (ms)
  50%   1196
  66%   1277
  75%   1329
  80%   1359
  90%   1480
  95%   1610
  98%   1856
  99%   2303
 100%   3955 (longest request)

```

####Blackfire
https://blackfire.io/profiles/12ac3405-0f1d-4693-86f5-9b184988364f/graph
