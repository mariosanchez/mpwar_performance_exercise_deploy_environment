#Ejercicio de web performance
##Diario de trabajo

En este ejercicio parto de un código base al que necesito aplicar una serie de 
mejoras que optimicen los tiempos de carga que tendría este proyecto de ejemplo en 
prodcución.

El código: https://github.com/fiunchinho/mpwar_performance_exercise

###Preparando el entorno de desarrollo

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
en la base de datos, así que, por el momento quedará así.

Tras esto parece que todo funciona.







