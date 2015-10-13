# ideas a tener en cuenta
  Estandarizacion - para toda aplicación que corre en un contenedor debe ser siempre la misma, no una a un fichero, otra que monta un volumen, etc.
  
  Las opciones que ofrece docker - desde la version 1.6 - permiten incluir logging drivers --log-driver=xxxxx:
  1. none	Disables any logging for the container. docker logs won’t be available with this driver.
  2. json-file	Default logging driver for Docker. Writes JSON messages to file.
  3. syslog	Syslog logging driver for Docker. Writes log messages to syslog.
  4. journald	Journald logging driver for Docker. Writes log messages to journald.
  5. gelf	Graylog Extended Log Format (GELF) logging driver for Docker. Writes log messages to a GELF endpoint likeGraylog or Logstash.
  6. fluentd	Fluentd logging driver for Docker. Writes log messages to fluentd (forward input).


 de que logs estamos hablando? del contenedor? del servidor de aplicaciones? de la aplicación? 
 
 Aquí es cuando empezamos a ver como si piensas en docker tienes cosas que ves que se pueden resolver de otra manera.
 
 Opciones:
 --log-driver=syslog --log-opt syslog-address=tcp://192.168.0.42:123 
 vuelco todo esto a la salida  /var/log/messages.
  
# Perspectiva de Produban
  todo al syslog - quien lo quiera explotar alli lo tiene...
  esto implica que los contenedores dentro de openshift no se van a ejecutar posibilitando la opción de especificar un 

# log-centralizado

Servicio que debe hacer las veces de servicio centralizador de logs de contenedores/pods. Debe ser un servicio escalable 

Al desplegar en un entorno con un orquestador - k8s - habrá que pensar en crear el servicio centralizador de logs que por detras tenga 1 o N pods que ejecuten el siguiente comando:
`docker run -d -v /var/lib/docker/containers:/var/lib/docker/containers kiyoto/docker-fluentd`

En el volumen /var/lib/docker/containers es donde se almacenaran los logs de los contenedores. Los logs son bufferizados de modo que será posible ver
ficheros del estilo: 

`/var/log/docker/20141114.b507c71e6fe540eab.log donde "b507c71e6fe540eab" es un identificador hash`

Un ejemplo de la salida  seria muy similar a la salida de docker daemon pero en formato json que incluye el id del contenedor.

{"log":"Fri Nov 14 01:03:19 UTC 2014\r\n","stream":"stdout","container_id":"db18480728ed247a64bf6df49684cb246a38bbe11f14276d4c2bb84f56255ff4"}

# ¿como lo usan otros?
http://rancher.com/running-our-own-elk-stack-with-docker-and-rancher/

Se incluye una dependencia y es que los contenedores que se arranquen, deberan indicar la IP/servicio del servicio de logs

# ¿como funciona?
Docker daemon genera ficheros de logs atendiendo a la siguiente estructura /var/lib/docker/containers/<CONTAINER_ID>/<CONTAINER_ID>-json.log
¿esto varia entre entornos?

Pues bien, fluentd usa uno de los multiples plugins - 300+ plugins - TAIL INPUT PLUGIN https://docs.fluentd.org/articles/in_tail 
para leer eventos que ocurren en estos plugins.

# Lista completa de plugins 
https://www.fluentd.org/plugins

hay que pensar en como será el INPUT y como será el OUTPUT

requiere td-agent corriendo en el contenedor?
