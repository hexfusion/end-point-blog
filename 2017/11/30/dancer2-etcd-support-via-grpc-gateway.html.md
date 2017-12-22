---
author: Sam Batschelet 
title: "Harnessing etcd's v3 API with Net::Etcd and Dancer2"
tags: perl, dancer2, etcd, ansible
---

### etcd

[etcd](https://github.com/coreos/etcd) is a open source distributed fault-tolerant key/value store written in Go. etcd utilities the [raft](https://raft.github.io/) protocaol to maintain consensus among it's nodes. The most common use-case is for persistent storage of container metadata and API objects for container scheduling systems such as [Kubernetes](https://kubernetes.io/). [Getting started with etcd](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/local_cluster.md).

### etcd v3 API
In the summer of 2016 etcd launched the [v3 API](https://coreos.com/blog/etcd3-a-new-etcd.html). Many lessons were learned with etcd v2. Quickly it's primary usecase shifted from a mechanism to coordinate updates for [Container Linux](https://coreos.com/why/#distro) (CoreOS) into a key value store with JSON endpoints. But at implimentations grew JSON proved too chatty and the need to manage millions of keys in a single cluster became a mission critical requirement. As a result of this requirement etcd v3 now uses [gRPC](https://grpc.io/) over HTTP2 instead of JSON which provided a 2x message processing improvement.

But as the v2 API was JSON and some languages such as Perl do not yet have full gRPC support. A decision early on was made to continue JSON support via a project called [gRPC gateway](https://github.com/grpc-ecosystem/grpc-gateway). Adding gRPC gateway support allows for both gRPC and JSON endpoints to be available at the same time.

![alt text](https://raw.githubusercontent.com/hexfusion/end-point-blog/master/2017/11/29/dancer2-etcd-support-via-grpc-gateway/grpc-gateway.png?raw=true "gRPC Gateway")

### Net::Etcd
Net::Etcd is a Perl client supporting the etcd v3 REST API exposed by the gRPC gateway. Creating this moduled proved to be much more challenging then expected. To highlight a few of the larger challenges, utilizing features of etcd such as key watches and lease keep-alives. These requests require asynchronous non blocking calls to the JSON API. Unlike Go, concurrency is not a core functionality of Perl. Perl does support asynchronous transactions through a few modules, I went with AnyEvent [AnyEvent::HTTP](https://metacpan.org/pod/AnyEvent::HTTP). Programming using asynchronous non blocking code takes a little getting used to.


```perl
# create a watch on key foo
$watch = $etcd->watch( { key => 'foo'}, sub {
    my ($result) =  @_;
    push @events, $result;
})->create;

# put key foo
$etcd->put({ key => 'foo', value => 'bar' });

# get key foo
$etcd->range({ key => 'foo' });

print scalar @events . "\n";

```
The result of this script prints '2'. So what happened here? This is the magic of using an asynconous callback. At the start the watch is created and the connection to etcd stays open. A watch literally creates a connection to the etcd cluster and watches for any action against the key (foo). When the put is made in the next line the event triggers a reply from the watch and pushes that result into the event array, the same for the get. This is a trivial example but really interesting if you think about it. The code below the watch triggered actions against code that has already run. Watches can be very useful for many reasons. Lets change the example to use a key ip_address. Now if the watch sees a change in that key. We have can perform another action such as update a DNS entry for example. It shouldn't be too hard to think of lots of data that you would want to take action on if it changed.

Once I got the hang of using callbacks and wrapped my head around writing non-blocking code I made nice progress and had asynchronous tests passing. At this point I really thought the war was won.
But then I had the startling realization that there was no support for authentication via grpc-gateway. [mike-drop](https://media.giphy.com/media/qlwnHTKCPeak0/giphy.gif). Not only was the support not availabe through etcd but no support existed for authentication at all through grpc-gateway. This was a complicated problem. The grpc-gateway reads a TODO So out of pure stubbornness I polished up my Go skills and added the support to etcd via [#7999](https://github.com/coreos/etcd/pull/7999). As of etcd v3.3+ header based token authentication via grpc-gateway is supported. Please give it a try!

### Testdrive



### Dancer

[dancer](https://github.com/PerlDancer/Dancer2) is a open source light weight framework for Perl. Dancer allows you to create Perl apps quickly. The plugin infrastructure is very powerful and allows for easy access to plugin code via hooks and keywords.


### Dancer::Plugin::Etcd

My motivation for creating the Dancer2 plugin was to have a simple method of deploying configuration data for Dancer2 instance. Although there are many ways to handle configuration deployment and population I felt etcd would simplify the process and allow for rolling updates to a container without the need to update the underlying image itself. The goal, updating a container would simply require it to restart to pull in the newest changes.

Dancer allows for configurations to be stored in both YAML and JSON. I find YAML much nicer to use because of it's human readability and support for comments.

### Comments can be useful additions to a configuration file.

```yaml
# include timestamp in log format
logger_format: "%t [%P] %L @%D> %m in %f l. %l" 
```

etcd is a perfect database for small amounts of data, even more interesting in a distributed ENV such as a Kubernetes deployment. So allowing a Dancer app easy access to the etcd datastore was something worht exploring.

Dancer::Plugin::Etcd contains a script called shepherd allows you to save Dancer App YAML configs to etcd by line as key/value. Even more interesting is that it maintains not just comments but whitespace.


### Dancer::Plugin::Etcd setup

To utilize shepherd with the plugin you will need to setup the plugin stub of your config. Now I know what your saying the config contains the root user/pass for etcd. That is why TLS is so important, and yes using TLS means that the key must be in the container. But [AWS](https://aws.amazon.com/blogs/security/how-to-manage-secrets-for-amazon-ec2-container-service-based-applications-by-using-amazon-s3-and-docker/), [Google Cloud Platform](https://cloud.google.com/kms/docs/store-secrets) and even [Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/) itself offer ways to share secrets with Docker containers as safely as possible.

The snippets below outline how this might look inside of a container.

### config.yml

```yaml
appname: "DanceShop"

plugins:
  Etcd:
```

### systemd unit files
```
#cloud-config

coreos:
  units:
    - name: plackup.service
      command: start
      enable: true
      content: |
        [Unit]
        Description=Plackup
        After=network.target

        [Service]
        Type=simple
        Environment='PLACKUP_OPTS=-E ${DANCER_ENVIRONMENT} -p 3000 -s Starman --pid=/var/run/plackup/placlup.pid --workers 1 -D -a bin/app.psgi'
        ExecStart=/usr/local/bin/plackup ${PLACKUP_OPTS}
        WorkingDirectory=/home/dancer_user/app/DanceShop
        Restart=always

        [Install]
        WantedBy=multi-user.target
    - name: dancer_config_init.service 
      command: start
      enable: true
      content: |
        [Unit]
        Description=Initialize Dancer configs
        Before=plackup.service
        dancer_config_init.service
        [Service]
        Environment='DANCER_ENVIRONMENT=staging'
        Type=oneshot
        WorkingDirectory=/home/dancer_user/app/DanceShop
        User=dancer_user
        Group=dancer_user

        ExecStart=/bin/bash /home/dancer_user/app/DanceShop/bin/config_init.sh
        Restart=no

        [Install]
        WantedBy=plackup.service

### config_init.sh
```
#!/bin/bash

shepherd --env ${DANCER_ENVIRONMENT} get
```

### environments/staging.yml

```
plugins:
  Etcd:
    host: 127.0.0.1
    port: 4001
    user: root
    password: h3xFu5ion
    cacert: path/to/cert.pem

```

### self signed certs
https://coreos.com/os/docs/latest/generate-self-signed-certificates.html

### Conclusion
Blah blah
