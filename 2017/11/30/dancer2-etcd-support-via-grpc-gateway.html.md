---
author: Sam Batschelet 
title: "Harnessing etcd's v3 API with Net::Etcd and Dancer2"
tags: perl, dancer2, etcd, ansible
---

### etcd

[etcd](https://github.com/coreos/etcd) is a open source leader-based distributed fault-tolerant key/value store written in Go by the team at CoreOS. etcd utilities the [raft](https://raft.github.io/) protocaol to maintain consensus among it's nodes. The most common use-case is for persistent storage of container metadata and API objects for distributed container scheduling systems such as [Kubernetes](https://kubernetes.io/). [Getting started with etcd](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/local_cluster.md).

### etcd v3 API
Many lessons were learned with etcd2. Quickly it's primary usecase shifted from a mechanism to coordinate updates for [Container Linux](https://coreos.com/why/#distro) (CoreOS) into a key value store with JSON endpoints. But at scale JSON proved much too chatty and the need to manage millions of keys in a single cluster became a requirement. As a result etcd v3 now uses gRPC via HTTP2 instead of JSON which gave a 2x message processing improvement.

But as the v2 API was JSON and some languages such as Perl do not yet have full gRPC support. It was very nice to see JSON was still to be supported via the [gRPC gateway](https://github.com/grpc-ecosystem/grpc-gateway)

![alt text](https://raw.githubusercontent.com/hexfusion/end-point-blog/master/2017/11/29/dancer2-etcd-support-via-grpc-gateway/grpc-gateway.png?raw=true "gRPC Gateway")

### Net::Etcd
I created Net::Etcd to provide Perl support for the etcd gRPC gateway. This proved to be much more challenging then expected. To highlight a few of the larger challenges, utilizing features of etcd such as key watches and lease keep-alives. These requests require asynchronous non blocking calls to the JSON API. Unlike Go, concurrency is not a core functionality of Perl. My solution was to use [AnyEvent::HTTP](https://metacpan.org/pod/AnyEvent::HTTP). Once I got the hang of using callbacks and had asynchronous tests passing I thought the war was won.
But then I had the startling realization that there was no support for authentication via grpc-gateway. <mike-drop>. I had quite a bit of time into this project at this point. So instead of giving up I polished up my Go skills and added the support to etcd via [#7999](https://github.com/coreos/etcd/pull/7999). So as of etcd v3.3+ header based authentication via grpc-gateway is supported. Please give it a try!

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


### config.yml

```yaml
appname: "DanceShop"

plugins:
  Etcd:
```

### systemd: define staging ENV
```
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

```

### systemd: dancer_config_init.service
```
[Unit]
Description=Initialize Dancer configs
Before=plackup.service

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



### Conclusion
Blah blah
