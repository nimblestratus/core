core
====

Core services for Nimblestratus.

The idea is to provide a scalable set of core services which are HA and resilient.

The identified services include:

* etcd discovery service
* DNS
* Docker Registry
* Node Management
* Fleet Cluster Management
* Routing & Service Discovery
* Metrics & Logs Gathering & Aggregation 

## Background ##

In order to resolve the chicken and egg problem of services needed to
bootstrap a PAAS cluster, some services need to be "pets" &mdash;
handcrafted. However, they need to be recreatable on demand; whether
due to expansion or a failure the services need to be well defined and
maintainable.

It is assumed that the services will be HA in order to prevent
a service failure from causing an entire cluster to fail.

## Philosophy and Motivators ##

In general and as much as possible, the driving motivator is to have a
basis upon which to build a stable, fault tolerant platform.  To this
end, the following design criteria are assumed:

1. Simple
2. Reuse
3. Well defined
4. Repeatable
5. Highly Available &mdash; there is no one copy of a service.

In order to support reuse, etcd is leveraged as a backing service
throughout the design.

## Services ##

### Etcd Discovery Service###

[Etcd](https://coreos.com/using-coreos/etcd/) is an open-source
distributed key-value store used by a number of
technologies including:

* [CoreOS](http://coreos.com)
* [Fleet](https://github.com/coreos/fleet) &mdash; a distributed init
  system

CoreOS provides a global cluster discovery service at
[https://discovery.etcd.io/new](https://discovery.etcd.io/new) which
returns something like:
[https://discovery.etcd.io/6c1f17ed3e6c866406857f90690a3d43](https://discovery.etcd.io/6c1f17ed3e6c866406857f90690a3d43).
This returns a json formatted object which lists the members of the
cluster.  An empty cluster looks like:

`{"action":"get","node":{"key":"/_etcd/registry/6c1f17ed3e6c866406857f90690a3d43","dir":true,"modifiedIndex":86931009,"createdIndex":86931009}}`

In order to simplify the use of the etcd discovery service and to
remove dependencies on outside links, a discovery service will be run
at a canonical name such as `http://discovery.local` which
mimics the behaviour of the global service.  The source for the global
repository is at
[https://github.com/coreos/discovery.etcd.io](https://github.com/coreos/discovery.etcd.io);
it has `https://discovery.etcd.io` hard coded in its source.  In order
to make it useful and be more [12 Factor](http://12factor.net)
compliant, the code should be modified to take an argument for the host and preferably read from the environment.

Additionally, DNS should maintain service records for the core etcd
servers and discovery servers.


### DNS ###

skydns is built upon etcd.

It needs to be started in "discovery" mode.

### Docker Registry ###

* Using a local registry eliminates the need to connect to outside
  resources for starting docker containers.  Additionally it allows
  for the use of flattened containers &mdash; a container in which all of
  the layers have been flattened and consequently it is
  deterministic.  Version 1.0 of the Gnomemobile container contains
  the entire image of Gnomemobile and its dependencies.  This
  alleviates potential issues with dependencies on the "latest"
  version of container.
* Runs on a known DNS location such as `registry.DOMAIN`.
  This assumes that the DNS search defaults to `DOMAIN`
* Exposes port 80.  This simplifies matters &mdash; instead of being on the
  "default" port of 5000 whereby containers are referenced by
  `registry:5000/redis:latest` the `:5000` can be left off, simplifying matters.
* Backed by [S3](http://en.wikipedia.org/wiki/Amazon_S3) instead of
  local storage.  If a server goes down, then
  the images will still be available.  The S3 storage can be provided
  by any service which understands the S3 protocol, such as [riak](http://basho.com/riak/). In particular riak-cs needs to be running. The [hectcastro/riak-cs](https://registry.hub.docker.com/u/hectcastro/riak-cs/) container works well. However, the docker daemon needs to be started with `-H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock` for the container to work as the makefile advertises.

### Fleet Cluster Management ###

* flit

### Routing ###

(eg haproxy)

### Service Discovery ###

(skydns supports this via service records)

## Core Servers OS ##

CoreOS will be used for the core services OS.  Services will either be
controlled by systemd or docker containers spawned by fleet.

Host metrics will be collected by metrix.

## Bootstrap Order ##

0. Have S3 store available for docker registry.
1. Bring up first etcd instance telling it that it is a "master" and
   set for auto-discovery. (cloud-config)
2. Bring up other etcd instances, telling them about the master and
   set them for auto-discovery. (cloud-config)
3. Bring up first docker registry instance on etcd master (it is a
   docker container).  It will need to be bootstrapped as it is a
   docker container and the registry does not yet exist.  Save its
   details to put into DNS.  This can be accomplished via cloud-config.
4. Use fleet to bring up DNS & register etcd instances in DNS.
5. Bring up additional docker registries, bootstrapped off of the
   initial registry instance.
6. Bring up metrics collector & register in DNS
7. Start metrics service on the core hosts
8. Use fleet to bring up Etcd Discovery service instances &
   register in DNS.
9. Fleet cluster management tool
