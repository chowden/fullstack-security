# Full Stack 

This architecture utilises Beat modules for data sources.

## Pre-requisites

1. Current Version of Docker 
1. Docker-Compose 
1. Ensuring the following ports are free on the host, as they are mounted by the containers:

    - `80` (Nginx)
    - `8000` (Apache2)
    - `5601` (Kibana)
    - `9200` (Elasticsearch)
    - `3306` (Mysql)
    
1. Atleast 4Gb of available RAM

## Versions

All Elastic Stack components are version 6.2.2. THis can be changed in the environment file (see at the bottom)

## Architecture 

The following containers are deployed:

* `Elasticsearch`
* `Kibana`
* `Filebeat` - Collecting logs from the apache2, nginx and mysql containers. Also responsible for indexing the host's system and docker logs.
* `Packetbeat` - Monitoring communication between all containers with respect to http, flows, dns and mysql.
* `Heartbeat` - Pinging all other containers over icmp. Additionally monitoring Elasticsearch, Kibana, Nginx and Apache over http. Monitors mysql over TCP.
* `Metricbeat` - Monitors nginx, apache2 and mysql containers using status check interfaces. Additionally, used to monitor the host system with respect cpu, disk, memory and network. Monitors the hosts docker statistics with respect to disk, cpu, health checks, memory and network.
* `Nginx` - Supporting container for Filebeat (access+error logs) and Metricbeat (server-status)
* `Apache2` - Supporting container for Filebeat (access+error logs) and Metricbeat (server-status)
* `Mysql` - Supporting container for Filebeat (slow+error logs), Metricbeat (status) and Packetbeat data.
* `Snort` - TODO - Intent here is to capture malicious traffic (and ultimately simulate it).

In addition to the above containers, a `configure_stack` container is deployed at startup.  This is responsible for:

* Setting the Elasticsearch passwords
* Importing any dashboards
* Ìnserting any custom templates and ingest pipelines

This container uses the Metricbeat images as it contains the required dashboards.

## Modules & Data TODO

The following Beats modules are utilised in this stack example to provide data and dashboards:

1. Packetbeat, capturing traffic on all interfaces:
    - `dns` - port `53`
    - `http` - ports `9200`, `80`, `8080`, `8000`, `5000`, `8002`, `5601`
    - `icmp`
    - `flows`
    - `mysql` - port `3306`
    
    NOTE: THIS WILL HAMMER ELASTICSEARCH, SO YOU MAY BE BETTER ADVISED TO REDUCE THE TRAFFIC CAPTURE DOWN TO SOMETHING MORE REALISTIC AND SAVE YOUR CPU.
    
1. Metricbeat
    - `apache` module with `status` metricset
    - `docker` module with `container`, `cpu`, `diskio`, `healthcheck`, `info`, `memory` and `network` metricsets 
    - `mysql` module with `status` metricset
    - `nginx` module with `stubstatus` metricset
    - `system` module with `core`,`cpu`,`load`,`diskio`,`filesystem`,`fsstat`,`memory`,`network`,`process`,`socket`
    
1. Heartbeat
    - `http` - monitoring Elasticsearch (9200), Kibana (5601), Nginx (80), Apache(80)
    - `tcp` - monitoring Mysql (3306)
    - `icmp` - monitoring all containers
    
1. Filebeat
    - `system` module with `syslog` metricset
    - `mysql` module with `access` and `slowlog` `metricsets`
    - `nginx` module with `access` and `error` `metricsets` 
    - `apache` module with `access` and `error` `metricsets`

## Step by Step Instructions - Deploying the Stack

1. git clone https://github.com/chowden/fullstack-security.git

1. make sure you're in the folder with the docker-compose.yml file 

## Dashboards with data TODO 

The following dashboards will ultimately be accessible and populated.

* CPU/Memory per container
* DNS
* Filebeat Apache2 Dashboard
* Filebeat MySQL Dashboard
* Filebeat Nginx Dashboard
* Filebeat syslog dashboard - Not available on Windows
* Heartbeat HTTP monitoring
* Metricbeat - Apache HTTPD server status
* Metricbeat Docker
* Metricbeat MySQL
* Metricbeat filesystem per Host
* Metricbeat system overview
* Metricbeat-cpu
* Metricbeat-filesystem
* Metricbeat-memory
* Metricbeat-network
* Metricbeat-overview
* Metricbeat-processes
* Packetbeat Dashboard (limited)
* Packetbeat Flows
* Packetbeat HTTP
* Packetbeat MySQL performance

## Generating data

The majority of the dashboards will simply populate due to inherent “noise” caused by the images.  However, we do expose a few additional ports for interaction to allow unique generation.  These include:

* MySQL - port 3306 is exposed allowing the user to connect. Any subsequent Mysql traffic will in turn be visible in the dashboards “Filebeat MySQL Dashboard”, “Metricbeat MySQL” and “Packetbeat MySQL performance”.
* Nginx - port 80. Currently we don’t host any content in Nginx so requests will result in 404s.  However, content can easily be added as described here.
* Apache2 - port 8000.  
* Docker Logs - Any activity to the docker containers, including requests to Kibana, are logged.  These logs are captured in JSON form and indexed into a index “docker-logs-<yyyy.mm.dd>”.

## Customising the Stack

With respect to the current example, we have provided a few simple entry points for customisation:

1. The example includes an .env file listing environment variables which alter the behaviour of the stack.  These environment variables allow the user to change:
    * `ELASTIC_VERSION` - the Elastic Stack version (currently 6.2.2) 
    * `ES_PASSWORD` - the password used for authentication with the elastic user. This password is applied for all system users i.e. kibana and logstash_system. Defaults to “changeme”.
    * `MYSQL_ROOT_PASSWORD` - the password used for the root mysql user. Defaults to “changeme”.
    * `DEFAULT_INDEX_PATTERN` - The index pattern used as the default in Kibana. Defaults to “metricbeat-*”.
    * `ES_MEM_LIMIT` - The memory limit used for the Elasticsearch container. Defaults to 2g. Consider reducing for smaller machines.
    * `ES_JVM_HEAP` - The Elasticsearch JVM heap size. Defaults to 1024m and should be set to half of the ES_MEM_LIMIT.
1. Modules and Configuration - All configuration to the containers is provided through a mounted “./config” directory.  Where possible, this exploits the dynamic configuration loading capabilities of Beats. For example, an additional module could be added by simply adding a file to the directory “./config/beats/metricbeat/modules.d/” in the required format.
1. Pipelines and templates - we provide the ability to add custom ingest pipelines and templates to Elasticsearch when the stack is first deployed. More specifically: `
    * Ingest templates should be added under `./init/templates/`.  These will be added on startup of the stack, with an id equal to the filename. For example, `docker-logs.json` will be added as a template with id `docker-logs`.
    * Pipelines should be added under `./init/pipelines/`. These will be added on startup of the stack, with an id equal to the filename. For example, `docker-logs.json` will be added as a pipeline with id `docker-logs`.


## Shutting down the stack

```shell
docker-compose down 
```

## Disclaimer and Acknowledgements
It is an ongoing project, forked and updated from https://www.elastic.co/blog/a-full-stack-in-one-command. It's been updated to work with 6.2.2

Some of it may or may not work

Authenication to Elastic and Kibana certainly doesn't and I'm not sure I can bothered to fix it as it's certainly not for production.
