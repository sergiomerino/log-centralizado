

# problematica
Logs are a critical part of any system, they give you insight into what a system is doing as well what happened. Virtually every process running on a system generates logs in some form or another. Usually, these logs are written to files on local disks. When your system grows to multiple hosts, managing the logs and accessing them can get complicated. Searching for a particular error across hundreds of log files on hundreds of servers is difficult without good tools. A common approach to this problem is to setup a centralized logging solution so that multiple logs can be aggregated in a central location.

# alternativas
  replicacion de ficheros mediante un planificador cron. Usually rsync and cron are used since they are simple and straightforward to setup. This solution can work for a while but it doesn’t provide timely access to log data. It also doesn’t aggregate the logs and only co-locates them.
  
  syslog. Most people use rsyslog or syslog-ng which are two syslog implementations.These daemons allow processes to send log messages to them and the syslog configuration determines how the are stored. In a centralized logging setup, a central syslog daemon is setup on your network and the client logging dameons are setup to forward messages to the central daemon. A good write-up of this kind of setup can be found at:jasonwilder.com/blog/2012/01/03/centralized-logging/
  
  With a central syslog server, you will likely need to figure out how to scale the server and make it highly-available.
  
  Distributed Log Collectors. A new class of solutions that have come about have been designed for high-volume and high-throughput log and event collection. Most of these solutions are more general purpose event streaming and processing systems and logging is just one use case that can be solved using them. All of these have their specific features and differences but their architectures are fairly similar. They generally consist of logging clients and/or agents on each specific host. The agents forward logs to a cluster of collectors which in turn forward the messages to a scalable storage tier. The idea is that the collection tier is horizontally scalable to grow with the increase number of logging hosts and messages. Similarly, the storage tier is also intended to scale horizontally to grow with increased volume. This is gross simplification of all of these tools but they are a step beyond traditional syslog options.
  
  Flume - Flume is an Apache project for collecting, aggregating, and moving large amounts of log data. It stores all this data on HDFS.

logstash - logstash lets you ship, parse and index logs from any source. It works by defining inputs (files, syslog, etc.), filters (grep, split, multiline, etc..) and outputs (elasticsearch, mongodb, etc..). It also provides a UI for accessing and searching your logs. See Getting Started jasonwilder.com/blog/2012/01/03/centralized-logging/

Chukwa - Chukwa is another Apache project that collects logs onto HDFS.

fluentd - Fluentd is similar to logstash in that there are inputs and outputs for a large variety of sources and destination. Some of it’s design tenets are easy installation and small footprint. It doesn’t provide any storage tier itself but allows you to easily configure where your logs should be collected.

kafka - Kafka was developed at LinkedIn for their activity stream processing. Although Kafka could be used for log collection this is not it’s primary use case. Setup requires Zookeeper to manage the cluster state.

Graylog2 - Graylog2 provides a UI for searching and analyzing logs. Logs are stored in MongoDB and/or elasticsearch. Graylog2 also provides the GELF logging format to overcome some issues with syslog message: 1024 byte limit and unstructured log messages. If you are logging long stacktraces, you may want to look into GELF.

splunk - Splunk is commercial product that has been around for several years. It provides a whole host of features for not only collecting logs but also analyzing and viewing them.
  
# arquitectura centralizada de logs

Many of these tools address only a portion of the problem which means you need to use several of them together to build a robust solution.

The main aspects you will need to address are: collection, transport, storage, and analysis. In some special cases, you may also want to have an alerting capability as well.


## Collection
Applications create logs in different ways, some log through syslog, others log directly to files. If you consider a typical web application running on a linux hosts, there will be a dozen or more log files in /var/log as well as a few application specific logs in home directories or other locations.

If you are supporting a web based application and your developers or operations staff need access to log data quickly in order to troubleshoot live issues, you need a solution that is able to monitor changes to log files in near real-time.

If you are using a file replication based approach where files are replicated to a central server on a fixed schedule, then you can only inspect logs as frequently as the replication runs. A one minute rsync cron job might not be fast enough when your site is down and you are waiting for the relevant log data to be replicated.

On the other hand, if you need to analyze log data offline for calculating metrics or other batch related work, a file replication strategy might be a good fit.

Transport
Log data can accumulate quickly on multiple hosts. Transporting it reliably and quickly to your centralized location may need additional tooling in order to effectively transmit it and ensure data is not lost.

Frameworks such as Scribe, Flume, Heka, Logstash, Chukwa, fluentd, nsq and Kafka are designed for transporting large volumes of data from one host to another reliably. Although each of these frameworks addresses the transport problem, they do so quite differently.

For example, Scribe, nsq and Kafka, require clients to log data via their API. Typically, application code is written to log directly to these sources which allows them to reduce latency and improve reliability. If you want to centralize typical log file data, you would need something to tail and stream the logs via their respective APIs. If you control the app that is logging the data you want to collect, these can be much more efficient.

Logstash, Heka, fluentd and Flume provide a number of input sources but also support natively tailing files and transporting them reliably. These are a better fit for more general log collection.

While rsyslog and Syslog-ng are typically thought of as the defacto log collector, not all applications use syslog.


Storage
Now that your log data is being transfered, it needs a destination. Your centralized storage system needs to be able to handle the growth in data over time. Each day will add a certain amount of storage that is relative to the number of hosts and processes that are generating log data.

How you store things depends on a few things:

How long should it be stored - If the logs are for long-term, archival purposes and do not require immediate analysis, S3, AWS Glacier, or tape backup might be a suitable option since they provide relatively low cost for large volumes of data. If you only need a few days or months worth of logs, storing them on some form distributed storage systems such as HDFS, Cassandra, MongoDB or ElasticSearch also works well. If you only need a few hours worth of retention for real-time analysis, Redis might work as well.

Your environments data volume. - A days worth of logs for Google is much different than a days worth of logs for ACME Fishing Supplies. The storage system you chose should allow you to scale-out horizontally if your data volume will be large.

How will you need to access the logs - Some storage is not suitable for real-time or even batch analysis. AWS Glacier or tape backup can take hours to load a file. These don’t work if you need log access for production troubleshooting. If you plan to do more interactive data analysis, storing log data in ElasticSearch or HDFS may allow you work with the raw data more effectively. Some log data is so large that it can only be analyzed in more batch oriented frameworks. The defacto standard is this case is Apache Hadoop along with HDFS.

Analysis
Once your logs are stored on a centralized storage platform, you need a way to analyze them. The most common approach is a batch oriented process that runs periodically. If you are storing log data in HDFS, Hive or Pig might help analyzing the data easier than writing native MapReduce jobs.

If you need a UI for analysis, you can store parsed log data in ElasticSearch and use a front-end such as Kibana or Graylog2 to query and inspect the data. The log parsing can be handled by Logstash, Heka or applications logging with JSON directly. This approach allows more real-time, interactive access to the data but is not really suited for a mass batch processing.

Alerting
The last component that is sometimes nice to have is the ability to alert on log patterns or calculated metrics based on log data. Two common uses for this are error reporting and monitoring.

Most log data is not interesting but errors almost always indicate a problem. It’s much more effective to have the logging system email or notify respective parties when errors ocurr instead of having someone repeatedly watch for the events. There are several services that solely provide application error logging such as Sentry or HoneyBadger. These can also aggregate repetitve exceptions which can give you and idea of how frequently an error is occuring.


Another use case is monitoring. For example, you may have hundreds of web servers and want to know if they start returning 500 status codes. If you can parse your web log files and record a metric on the status code, you can then trigger alerts when that metric crosses a certain threshold. Riemann is designed for detecting scenarios just like this.

Hopefully this helps provide a basic model for designing a centralized logging solution for your environment.

Fluentd and Logstash are two open-source projects that focus on the problem of centralized logging. Both projects address the collection and transport aspect of centralized logging using different approaches.

Logstash requiere una JVM which is usually means Linux, Mac OSX, and Windows. The package is shipped as single executable jar file which makes it very easy to install.

One of the downsides of depending on the JVM is that it’s memory footprint can be higher than you would want for transporting logs. Fortunately, Lumberjack can be run on individual hosts to collect and ship logs and Logstash can be run on the centralized log hosts.

Lumberjack is a Go based project with a much smaller memory and CPU footprint. Deployment is still straightforward as Logstash and is basically installing a single binary. The project provides deb and rpm packages to make it easier to deploy. An SSL certificates is required to setup authentication between Lumberjack and Logstash which is a little more complicated, but a nice benefit that encrypted transport is the default.

Fluentd is a CRuby application which requires Ruby 1.9.2 or later. There is an open-source version, fluentd, as well as a commercial version, td-agent. Fluentd runs on Linux and Mac OSX, but does not run on Windows currently.

For larger installs, they recommend using jemalloc to avoid memory fragmentation. This is included in the deb and rpm packages but needs to be installed manually if using the open-source version.

Logstash supports a number of inputs, codecs, filters and outputs. Inputs are sources of data. Codecs essentially convert an incoming format into an internal Logstash representation as well as convert back out to an output format. These are usually used if the incoming message is not just a single line of text. Filters are processing actions on events and allow you to modify events or drop events as they are processed. Finally, outputs are destinations where events can be routed.

Fluentd is similar in that it has inputs and outputs and a matching mechanism to route log data between destinations. Internally, log messages are converted to JSON which provides structure to an unstructered log message. Messages can be tagged and then routed to different outputs.

Both projects have very similar capabilities and highlighting the difference between them from a feature standpoint is difficult. They both have plugin models that allow you to extend their functionality if needed. They also have rich repository of plugins already available.

Probably the most significant difference between Fluentd and Logstash is their design focus.

Logstash emphasizes flexibility and interoperability whereas Fluentd prioritizes simplicity and robustness. This does not mean that Logstash is not robust or Fluentd is not flexible, rather each has prioritized features differently.

Fluentd has fewer inputs than Logstash, out of the box, but many of the inputs and outputs have built-in support for buffering, load-balancing, timeouts and retries. These types of features are necessary for ensuring data is reliably delivered.

For example, the out_forward plugin used to transfer logs from one fluentd instance to another has many robustness options that can be configured to ensure messages are delivered reliably.

Architecture comparison
From a deployment architecture standpoint, both frameworks are very similar. With Logstash, each web server would be configured to run Lumberjack and tail the web server logs. Lumberjack would forward the logs to a server running Logstash with a Lumberjack input. The Logstash server would also have an output configured using the S3 output. Since Lumberjack requires SSL certs, the log transfers would be encrypted from the web server to the log server.

With fluentd, each web server would run fluentd and tail the web server logs and forward them to another server running fluentd as well. This server would be configured to receive logs and write them to S3 using the S3 plugin

Improving Availability
One central log server is a single point of failure. What happens if we wanted to have more than one central log server?

Lumberjack can be configured to use multiple servers but will only send logs to one until that one fails. If that happens, previously collected log data won’t be accessible until that host is resurrected. Essentially, it supports a master with hot-standby servers.

Fluentd on the other hand can forward two copies of the logs to each server if needed, load-balance between multiple hosts or have a master with a hot-standy in case of failure. There are lot of options for not only improving availabilty but also scalability if your log volume increases substantially. Keep in mind, that if you forward multiple copies, this could create duplicate logs in S3 which might need to be handled when you analyze them.

http://jasonwilder.com/blog/2013/11/19/fluentd-vs-logstash/

http://logstash.net/docs/1.2.2/life-of-an-event


Temas a mirar:

dependencies, features, deployment architecture and potential issues. The point is not to figure out which one is the best, but rather to see which one would be a better fit for your environment.



# Ideas a tener en cuenta
  0. Estandarizacion. Para toda aplicación que corre en un contenedor debe ser siempre la misma, no una a un fichero, otra que monta un volumen, otra que utiliza un driver distinto, etc.
  1. Solucion end_2_end (captura, transformación y explotacion) multitenant (entendiendo por tenant la aplicación) de manera que se garantice el aislamiento de proceso y consulta de datos (sobre todo en la parte de explotación)
  2. Fuerte dependencia de la imagen docker, de la version de docker daemon, del SO donde corren los dockers y de los permisos que requiera la solucion (privileged) 
  
  Las opciones que ofrece docker - desde la version 1.6 - permiten incluir logging drivers --log-driver=xxxxx:
  1. none	Disables any logging for the container. docker logs won’t be available with this driver.
  2. json-file	Default logging driver for Docker. Writes JSON messages to file.
  3. syslog	Syslog logging driver for Docker. Writes log messages to syslog.
  4. journald	Journald logging driver for Docker. Writes log messages to journald.
  5. gelf	Graylog Extended Log Format (GELF) logging driver for Docker. Writes log messages to a GELF endpoint likeGraylog or Logstash.
  6. fluentd	Fluentd logging driver for Docker. Writes log messages to fluentd (forward input).

Graylog

https://github.com/deviantony/docker-elk

Elasticsearch is an advanced search engine which is super fast. With Elasticsearch, you can search and filter through all sorts of data via a simple API. The API is RESTful, so you can not only use it for data-analysis but also use it in production for web-based applications.

Logstash is a tool intended for organizing and searching logfiles. But it can also be used for cleaning and streaming big data from all sorts of sources into a database. Logstash also has an adapter for Elasticsearch, so these two play very well together.

Kibana is a visual interface for Elasticsearch that works in the browser. It is pretty good at visualizing data stored in Elasticsearch and does not require programming skills, as the visualizations are configured completely through the interface.

SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux into Permissive mode in order for fig-elk to start properly. For example on Redhat and CentOS, the following will apply the proper context:

.-root@centos ~
-$ chcon -R system_u:object_r:admin_home_t:s0 fig-elk/


fluentd versus logstash
recoleecion y transporte de logs. 

 De que logs estamos hablando? del contenedor? del servidor de aplicaciones? de la aplicación? 
 
 NOTA: Aquí es cuando empezamos a ver como si piensas en docker tienes cosas que ves que se pueden resolver de otra manera.
       ¿la solucion será distinta para local y para un entorno gestionado? en local necesito algo de esto - creo q no
       
 docker run --log-driver=fluentd --log-opt fluentd-address=localhost:24224 --log-opt fluentd-tag=docker.{{.Name}}
If container cannot connect to the Fluentd daemon on the specified address, the container stops immediately. 

 Opciones:
 --log-driver=syslog --log-opt syslog-address=tcp://192.168.0.42:123 
 vuelco todo esto a la salida  /var/log/messages.
  
  otro ejemplo. en este parece interesante que aparece la opcion del token.
  https://www.loggly.com/docs/docker-syslog/
  
  Deberia existir un servicio que fuera de logs en la plataforma.
  
  
# Perspectiva de Pro
  todo al syslog - quien lo quiera explotar alli lo tiene... en este caso la fuente serán los ficheros de log del daemon.
  esto implica que los contenedores dentro de openshift no se van a ejecutar posibilitando la opción de especificar un driver u otro.

# log-centralizado
Alternativas ELK
             EFK

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
