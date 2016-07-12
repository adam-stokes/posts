---
published: false
layout: post
title: deploy to lxc AND kvm in the local provider
date: 2014-03-20T02:09Z
tags: juju, ubuntu
---
While messing around with juju 1.18.x I managed to stumble across a setup that allows me to deploy both LXC and KVM containers in a single environment.

## Pre-reqs

*   Juju v1.18 or higher
*   libvirt-bin
*   lxc
*   Ubuntu Trusty or higher (only one I tested this on)

## The juju environments.yaml

Very simple configuration for this

    default: local

    environments:
      local:
        type: local
        lxc-use-clone: true
        container: kvm

**Note** _lxc-use-clone_ was introduced in juju 1.18.3 and 1.19.2\. This will greatly reduce the wait time it takes to deploy systems into containers no matter the provider used (as long as it supports lxc)

## Networking modification

In order for the machines to talk to one another we need to make sure that the KVM machine that's housing the lxc containers are utilizing the same network that the KVM uses. So if we didn't modify the network and attempted to deploy you would have **machine 1** with an ip of like **10.0.3.1** and when you deploy a service to an lxc container within that KVM it would assign an address of **10.0.4.x** or whatever the lxcbr0 is using for its network.

To fix this you'll want to make sure that the KVM machine is using a host only network where lxcbr0 is bridged to the **eth0** device. The output below shows a simple way to make this work:

In your **/etc/network/interfaces.d/lxcbr0.cfg** file put the following:

    auto eth0
    iface eth0 inet manual

    auto lxcbr0
    iface lxcbr0 inet dhcp
        bridge_ports eth0

From there you can remove the existing **eth0.cfg** file and reboot your kvm instance. Once rebooted you can `juju deploy mysql --to lxc:1` and it'll have the correct ip associated with it so that all kvm/lxc containers can communicate with one another.

## Bootstrap and Deploy

    $ juju bootstrap
    $ juju set-constraints mem=2G
    $ juju add-machine
    $ juju deploy openstack-dashboard --to lxc:1
    $ juju deploy mysql --to lxc:1
    $ juju deploy keystone --to lxc:1
    $ juju add-machine
    $ juju deploy nova-compute

What this does is deploy a few openstack components to kvm and LXC where the KVM instances are your separate machines and lxc being containers within the KVM machine.

## Output from a Openstack cloud deployed on a single machine using both lxc and kvm

    environment: local
    machines:
      "0":
        agent-state: started
        agent-version: 1.18.1.1
        dns-name: localhost
        instance-id: localhost
        series: trusty
      "1":
        agent-state: started
        agent-version: 1.18.1.1
        dns-name: 10.0.3.64
        instance-id: poe-local-machine-1
        series: trusty
        containers:
          1/lxc/0:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.177
            instance-id: juju-machine-1-lxc-0
            series: trusty
            hardware: arch=amd64
          1/lxc/1:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.230
            instance-id: juju-machine-1-lxc-1
            series: trusty
            hardware: arch=amd64
          1/lxc/2:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.67
            instance-id: juju-machine-1-lxc-2
            series: trusty
            hardware: arch=amd64
          1/lxc/3:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.178
            instance-id: juju-machine-1-lxc-3
            series: trusty
            hardware: arch=amd64
          1/lxc/4:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.84
            instance-id: juju-machine-1-lxc-4
            series: trusty
            hardware: arch=amd64
          1/lxc/5:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.14
            instance-id: juju-machine-1-lxc-5
            series: trusty
            hardware: arch=amd64
          1/lxc/6:
            agent-state: started
            agent-version: 1.18.1.1
            dns-name: 10.0.3.66
            instance-id: juju-machine-1-lxc-6
            series: trusty
            hardware: arch=amd64
        hardware: arch=amd64 cpu-cores=1 mem=2048M root-disk=8192M
      "2":
        agent-state: started
        agent-version: 1.18.1.1
        dns-name: 10.0.3.219
        instance-id: poe-local-machine-2
        series: trusty
        hardware: arch=amd64 cpu-cores=1 mem=2048M root-disk=8192M
      "3":
        agent-state: started
        agent-version: 1.18.1.1
        dns-name: 10.0.3.237
        instance-id: poe-local-machine-3
        series: trusty
        hardware: arch=amd64 cpu-cores=1 mem=2048M root-disk=8192M
    services:
      glance:
        charm: cs:trusty/glance-0
        exposed: false
        relations:
          cluster:
          - glance
          identity-service:
          - keystone
          image-service:
          - nova-cloud-controller
          - nova-compute
          shared-db:
          - mysql
        units:
          glance/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/2
            open-ports:
            - 9292/tcp
            public-address: 10.0.3.67
      juju-gui:
        charm: cs:trusty/juju-gui-2
        exposed: false
        units:
          juju-gui/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/4
            open-ports:
            - 80/tcp
            - 443/tcp
            public-address: 10.0.3.84
      keystone:
        charm: cs:trusty/keystone-2
        exposed: false
        relations:
          cluster:
          - keystone
          identity-service:
          - glance
          - nova-cloud-controller
          - openstack-dashboard
          shared-db:
          - mysql
        units:
          keystone/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/5
            public-address: 10.0.3.14
      mysql:
        charm: cs:trusty/mysql-0
        exposed: false
        relations:
          cluster:
          - mysql
          shared-db:
          - glance
          - keystone
          - nova-cloud-controller
          - nova-compute
        units:
          mysql/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: "3"
            public-address: 10.0.3.237
      nova-cloud-controller:
        charm: cs:trusty/nova-cloud-controller-36
        exposed: false
        relations:
          amqp:
          - rabbitmq-server
          cloud-compute:
          - nova-compute
          cluster:
          - nova-cloud-controller
          identity-service:
          - keystone
          image-service:
          - glance
          shared-db:
          - mysql
        units:
          nova-cloud-controller/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/1
            open-ports:
            - 3333/tcp
            - 8773/tcp
            - 8774/tcp
            public-address: 10.0.3.230
      nova-compute:
        charm: cs:trusty/nova-compute-0
        exposed: false
        relations:
          amqp:
          - rabbitmq-server
          cloud-compute:
          - nova-cloud-controller
          compute-peer:
          - nova-compute
          image-service:
          - glance
          shared-db:
          - mysql
        units:
          nova-compute/1:
            agent-state: started
            agent-version: 1.18.1.1
            machine: "2"
            public-address: 10.0.3.219
      openstack-dashboard:
        charm: cs:trusty/openstack-dashboard-0
        exposed: false
        relations:
          cluster:
          - openstack-dashboard
          identity-service:
          - keystone
        units:
          openstack-dashboard/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/3
            open-ports:
            - 80/tcp
            - 443/tcp
            public-address: 10.0.3.178
      rabbitmq-server:
        charm: cs:trusty/rabbitmq-server-1
        exposed: false
        relations:
          amqp:
          - nova-cloud-controller
          - nova-compute
          cluster:
          - rabbitmq-server
        units:
          rabbitmq-server/0:
            agent-state: started
            agent-version: 1.18.1.1
            machine: 1/lxc/6
            open-ports:
            - 5672/tcp
            public-address: 10.0.3.66

## Disclaimer

*   At this time it is not currently recommended to do it this way.
*   I say do it anyway and see what you can deploy.
