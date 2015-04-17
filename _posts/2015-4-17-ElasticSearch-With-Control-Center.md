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

# Building the image
To run a service in Control Center, you need a Docker image for the application you want to run. There’s nothing unique to Control Center in this; [an existing Docker image will probably work](https://registry.hub.docker.com/). For the purposes of this example, here’s a very simple Dockerfile that defines an image with elasticsearch 1.4.2 installed to /usr/local/elasticsearch:

```docker
# Use semiofficial image with Java installed (https://github.com/dockerfile/java)
FROM dockerfile/java:oracle-java7
# Download and unpack elasticsearch into /usr/local
RUN curl -s https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.2.tar.gz | tar -C /usr/local -xz
RUN mv /usr/local/elasticsearch-1.4.2 /usr/local/elasticsearch
# Make a directory to hold data (more about this later)
RUN mkdir -p /var/data/elasticsearch
```
