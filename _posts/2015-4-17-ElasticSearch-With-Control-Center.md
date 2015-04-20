---
layout: post
title: Delivering Applications in Control Center, Part 1
---
[Control Center](http://controlcenter.io) is an open source tool for deploying
and managing complex applications across any number of hosts. We use it at
[Zenoss](http://www.zenoss.org) to allow easy scale out of Zenoss services
(like HBase, RabbitMQ, and Zenoss itself) with almost no configuration.  If
you've got an application with a bunch of moving parts, you can describe that
service in a Control Center service template, then provide that template to the
world.  Anybody can then deploy that same template into their own Control
Center to run your application.  This tutorial will go over the basics of how
to define your own application in Control Center.

Part 1 of this tutorial will go over a simplistic case, running a single
instance of [Elasticsearch](https://www.elastic.co). Part 2 will turn that same
service into an easily expandable cluster.

The source for this example is [available on
GitHub](https://github.com/iancmcc/elasticsearch-ctrlctr). You may also find
[Control Center's service definition
documentation](http://controlcenter.io/docs/index.html) useful. 

This assumes you've [installed Control Center
1.0.x](http://controlcenter.io/gettingstarted.html), which will also install
[Docker](https://www.docker.com).

## Overview
Control Center provides an application with a distributed filesystem,
centralized configuration and logging, and internal networking. You build
a Docker image containing the software, wire the components of your application
up in a JSON service definition, and turn it loose.

We'll go through all the steps:

 * Building the Docker image
 * Describing config files
 * Describing networking
 * Defining persistent storage
 * Defining health checks

## Building the image

_Note: Instead of building your own image, you can use [one I've already
built](https://registry.hub.docker.com/u/iancmcc/elasticsearch://registry.hub.docker.com/u/iancmcc/elasticsearch/).
Just replace all references to ``elasticsearch-example`` with
``iancmcc/elasticsearch``._

To run a service in Control Center, you need a Docker image for the application
you want to run. There’s nothing unique to Control Center in this image; [an
existing Docker image will probably work](https://registry.hub.docker.com/).
For the purposes of this example, here’s a very simple Dockerfile that defines
an image with Elasticsearch 1.4.2 installed:

```docker
# Use semiofficial image with Java installed 
# (https://github.com/dockerfile/java)
FROM dockerfile/java:oracle-java7
# Download and unpack Elasticsearch into /usr/local
RUN curl -s https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.2.tar.gz | tar -C /usr/local -xz
RUN ln -s /usr/local/elasticsearch-1.4.2 /usr/local/elasticsearch
# Make a directory to hold data (more about this later)
RUN mkdir -p /var/data/elasticsearch
# Install jq to make health checks easier to write
RUN apt-get update && \
    apt-get install jq && \
    rm -rf /var/lib/apt/lists/*
```

Stick that Dockerfile in an empty directory, cd to it, and run:

```bash
docker build -t elasticsearch-example .
```

Make sure it's there:

```bash
docker images | grep elasticsearch-example
```

You can try running it quick to make sure it works:

```bash
docker run --rm -it elasticsearch-example \
    /usr/local/elasticsearch/bin/elasticsearch
```

## Creating the Service Template

Control Center deploys services based on two things: the Docker image, and a
JSON template defining the various attributes of the service — storage,
networking, configuration files, health checks and so on. We’ve got the image,
so it’s time to create the template. 

The basic structure of a service template is a JSON file, but serviced (the
Control Center binary) has a built-in template compiler that can operate on a
directory to build out the JSON you need. This is essential, because otherwise
you have to put all your config files in the JSON, and it gets unreadable and
uneditable. So let’s make the template directory, the config file directory,
and the base service template:

```bash
mkdir elasticsearch
# -CONFIGS- is a convention the template compiler looks for
mkdir elasticsearch/-CONFIGS- 
touch elasticsearch/service.json
```

### Service Metadata
First, we describe a few simple things about our service in ``service.json``:

```json
{
     "Title": "elasticsearch-cluster",
     "Name": "Elasticsearch",
     "Version": "1.4.2",
     "Launch": "auto",
     "Command": "/usr/local/elasticsearch/bin/elasticsearch",
     "ImageID": "elasticsearch-example",
     "Instances": {
        "Default": 1
     }
}
```

Notice I called it "elasticsearch-cluster"; admittedly, this is jumping the
gun for this part of the tutorial. Bear with me until Part 2 for the payoff.

So, technically, what you've defined is enough to run the thing, but it won’t
be super useful yet, since you have a) no way to access the Elasticsearch
server, and b) no way to configure it.

### Config Files
Control Center manages the config files for your service. You edit them in the
Control Center UI and restart your service to see the changes. Control Center
will inject them into the container on startup wherever in the filesystem you
tell it to. 

This helps two things: you don’t have to rebuild or retag your
service image every time you want to tweak a config file, and services can
start up on any host in the cluster without worrying about that host’s state.

Also, config files injected via Control Center can take advantage of templating
to set values based on the service definition or the state of the system.

For our Elasticsearch service, we should at least expose the two Elasticsearch
config files, ``elasticsearch.yml`` and ``logging.yml``. Let’s pull the
defaults out of the Elasticsearch distribution in the image we just built and
include them in our service template. We'll make a temporary container, copy
the files out, and remove the container:

```bash
# Create the directory
export ESDIR=/usr/local/elasticsearch/config
export CFGDIR=elasticsearch/-CONFIGS-$ESDIR
mkdir -p $CFGDIR
# Create a container based on your image, to get the files out
docker run --name es-configs elasticsearch-example echo
# Copy the files from that container
docker cp es-configs:$ESDIR/elasticsearch.yml $CFGDIR
docker cp es-configs:$ESDIR/logging.yml $CFGDIR
# Clean up the container
docker rm es-configs
```

Now let’s tell our service template where they are and where to put them. Add
to ``service.json`` (don't forget appropriate commas):

```json
{
     ...
     "ConfigFiles": {
          "/usr/local/elasticsearch/config/elasticsearch.yml": {
               "FileName": "/usr/local/elasticsearch/config/elasticsearch.yml",
               "Owner": "root:root",
               "Permissions": "0664"
          },
          "/usr/local/elasticsearch/config/logging.yml": {
               "FileName": "/usr/local/elasticsearch/config/logging.yml",
               "Owner": "root:root",
               "Permissions": "0664"
          }
     }
}
```
When the template is compiled, those files will be pulled in and set as the
defaults, and when the service is deployed, they'll be editable in the UI.

### Networking
In Part 1, we don't have any internal networking to configure, since we're not
defining a cluster, but we still want some ports exposed, so we can access
the Elasticsearch REST API. This is pretty simple.  We'll just expose the
default HTTP port (9200). Let’s also add a default virtual host we can use to
access the service through the HTTPS interface Control Center provides. Add to
``service.json``:

```json
{
     ...
     "Endpoints": [
          {
               "Name": "elasticsearch",
               "Application": "elasticsearch",
               "PortNumber": 9200,
               "Protocol": "tcp",
               "Purpose": "export",
               "Vhosts": ["elasticsearch"]
          }
     ]
}
```

At this point, we could run this service and access the API, but when it
restarted, the data would be lost, since it would be written to the container’s
filesystem. We need to configure a persistent volume and tell elasticsearch to
put its data there.

### Storage
Control Center manages a distributed filesystem, which it makes available to
services at runtime. Let’s define one to hold elasticsearch data. Add to
``service.json``:

```json
{
     ...
     "Volumes": [
          {
               "Owner": "root:root",
               "Permission": "0755",
               "ResourcePath": "elasticsearch-data",
               "ContainerPath": "/var/data/elasticsearch"
          }
     ]
}
```

Next, we tell elasticsearch to put data there. Edit
``elasticsearch/-CONFIGS-/usr/local/elasticsearch/config/elasticsearch.yml``
and add:

```YAML
path:
     data: /var/data/elasticsearch
```

### Health Checks

It's good practice to add health checks for your service. Control Center will run these in each container and report the results back. We'll add a quick one to ``service.json`` that simply checks the Elasticsearch cluster status:

```json
{
     ...
     "HealthChecks": {
        "cluster": {
            "Script": "curl -s 'http://localhost:9200/_cluster/health?pretty=true' | jq '.status' | grep -q green",
            "Interval": 10.0
        }
    }
}
```
This uses [jq](http://stedolan.github.io/jq/) to parse the JSON returned by Elasticsearch.

## Compiling the template

Now all we need is to deploy this thing. Compiling the template simply combines
all the parts into a single JSON file, which, in this example, really just
means bringing the config files in. You use the same binary (``serviced``) to
compile the template as you do to run Control Center. 

```bash
serviced template compile elasticsearch > /tmp/elasticsearch.json
```

## Deploying the template

You can do this from the Control Center UI just as easily,
but for ease we'll do it from the command line. First, add the template you
just created to Control Center, and save off the ID it will output:

```bash
ELASTICTPL=$(serviced template add /tmp/elasticsearch.json)
```

Then deploy an instance of it to your default pool, named "Elasticsearch":

```bash
serviced template deploy $ELASTICTPL default Elasticsearch
```

Now you can start it up:

```bash
serviced service start Elasticsearch 
```

## Accessing the virtual host
In order to use the virtual host, you'll need to access it by hostname. Add an entry to /etc/hosts to add ``elasticsearch.HOSTNAME`` to the entry for ``HOSTNAME``. For example (specific values obviously won't work 
for you):

```bash
sudo bash -c 'echo 10.1.2.3 elasticsearch.cchost cchost >> /etc/hosts'
```

Then you should be able to access the API:

```bash
curl -k https://elasticsearch.cchost
```
