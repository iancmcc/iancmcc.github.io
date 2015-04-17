---
layout: post
title: Easy ElasticSearch with Control Center
---
Intro.

As an example, here’s how you might deploy an elasticsearch cluster with
Control Center. The source for this example is [available on
GitHub](https://github.com/iancmcc/elasticsearch-ctrlctr). You may also find [Control Center's
service definition documentation](http://controlcenter.io/docs/index.html)
useful. 

This assumes you've [installed Control Center
1.0.x](http://controlcenter.io/gettingstarted.html), which will also install
[Docker](https://www.docker.com).

## Building the image
To run a service in Control Center, you need a Docker image for the application you want to run. There’s nothing unique to Control Center in this; [an existing Docker image will probably work](https://registry.hub.docker.com/). For the purposes of this example, here’s a very simple Dockerfile that defines an image with elasticsearch 1.4.2 installed to /usr/local/elasticsearch:

```docker
# Use semiofficial image with Java installed 
# (https://github.com/dockerfile/java)
FROM dockerfile/java:oracle-java7
# Download and unpack elasticsearch into /usr/local
RUN curl -s https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.2.tar.gz | tar -C /usr/local -xz
RUN ln -s /usr/local/elasticsearch-1.4.2 /usr/local/elasticsearch
# Make a directory to hold data (more about this later)
RUN mkdir -p /var/data/elasticsearch
```

Stick that Dockerfile in an empty directory, cd to it, and run:

```bash
$ docker build -t elasticsearch-example .
```

Make sure it's there:

```bash
$ docker images | grep elasticsearch-example
```

You can try running it quick to make sure it works:

```bash
$ docker run --rm -it elasticsearch-example \
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
$ mkdir elasticsearch
# -CONFIGS- is a convention the template compiler looks for
$ mkdir elasticsearch/-CONFIGS- 
$ touch elasticsearch/service.json
```

### Service Metadata
First, we describe a few simple things about our service in ``service.json``:

```json
{
     "Title": "elasticsearch-cluster",
     "Name": "ElasticSearch",
     "Version": "1.4.2",
     "Launch": "auto",
     "Command": "/usr/local/elasticsearch/bin/elasticsearch",
     "ImageID": "elasticsearch-example"
}
```

Technically, that’s enough to run the thing, but it won’t be super useful yet, since you have a) no way to access the elasticsearch server, and b) no way to configure it.

### Config Files
Control Center manages the config files for your service. You edit them in the
Control Center UI and restart your service to see the changes. Control Center
will inject them into the container on startup wherever in the filesystem you
tell it to. This helps two things: you don’t have to rebuild or retag your
service image every time you want to tweak a config file, and services can
start up on any host in the cluster without worrying about that host’s state.
Also, config files injected via Control Center can take advantage of templating
to set values based on the service definition or the state of the system.

For our elasticsearch service, we should provide the two elasticsearch config
files, ``elasticsearch.yml`` and ``logging.yml``. Let’s pull the defaults out
of the elasticsearch distribution in the image we just built and include them
in our service template:

```bash
# Create the directory
$ export ESDIR=/usr/local/elasticsearch/config
$ export CFGDIR=elasticsearch/-CONFIGS-$ESDIR
$ mkdir -p $CFGDIR
# Create a container based on your image, to get the files out
$ docker run -n es-configs elasticsearch-example echo
# Copy the files from that container
$ docker cp es-configs:$ESDIR/elasticsearch.yml $CFGDIR
$ docker cp es-configs:$ESDIR/logging.yml $CFGDIR
# Clean up the container
$ docker rm es-configs
```

Now let’s tell our service template where they are and where to put them. Add
to ``service.json``:

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
When the template is compiled, those files will be pulled in and set as the defaults.

### Networking
Next, we need to expose some ports from inside the container, so we can access
the elasticsearch REST API. This is similarly straightforward. Since we’re not
worried about clustering at this stage, let’s just expose the default HTTP port
(9200). Let’s also add a default virtual host we can use to access the service
through the HTTPS interface Control Center provides. Add to ``service.json``:

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

