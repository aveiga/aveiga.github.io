---
title: Setting up a Kubernetes cluster using Raspberry Pi Zero 2 W's
date: "2022-05-17T19:00:00.000Z"
description: "Deploying k3s to create a Kubernetes cluster running on top of 3 Raspberry Pi Zero 2 W's and using external Kubernetes control plane"
---

# Introduction

This week I wanted to challenge myself and try to create a Kubernetes cluster on 3 of my Raspberry Pi Zero 2 W's. These Pi's are somewhat underpowered for the task, especially in regards to their RAM, and I hoped to learn a trick or two on setting up High Availability on such a constrained environment.

If you've read my previous posts, you'll know that I'm using [k3s](https://k3s.io), from Rancher Labs. It's light-weight, full-featured, and highly configurable. The perfect fit for this use case.

# Solution

Highly available cluster will always need 3 Kubernetes nodes, at the very least, as per the [Raft Algorithm](https://raft.github.io) used by etcd, the Kubernetes Control Plane database. This is, of course, assuming that application workloads are schedulable to the master nodes. If that's not the case, as best practices dictate, then you need an Highly Available Control Plane (3 or more nodes) plus a number of worker nodes, depending on your own workload.

In this exercise, for simplicity purposes, I'm using three Raspberry Pi Zero W's that will run both the Control Plane as well as the application workloads. At least that was the plan...

## Attempt #1: Running the standard k3s installation - Failed

My first approach was to ignore k3s' minimum hardware requirements and simply run the installer with no further deployment customizations. This proved to be impossible, as the k3s daemon didn't even start due to insufficient memory on the Raspberry Pi. Nonetheless, this is how I went about it:

The first node is pretty simple, just run the regular k3s install script:

```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init
```

The next two Kubernetes nodes are setup by running:

```bash
curl -sfL https://get.k3s.io | sh -s - server --server https://<ip or hostname of server1>:6443
```

This should give you an HA cluster where the master nodes are schedulable. Not possible with Pi Zero 2's, apparently, but more than possible with Pi 4's (or even Pi 3's).

## Attempt #2: Externalizing the Control Plane DB

By default, k3s will run an embedded etcd as the Control Plane DB. It's well documented that this solution may have ["performance issues on slower disks such as Raspberry Pis running with SD cards"](https://rancher.com/docs/k3s/latest/en/installation/ha-embedded/), so I though running an external DB to act as the Kubernetes Control Plane would maybe easy the load on the Pi Zero's... Luckily, this is supported out-of-the-box by k3s.

First of all, you have to choose between using PostgreSQL, MySQL, MariaDB and (external) etcd. I went with PostgreSQL as I'm more familiar with it.

Since this needs to run externally (and the point is to remove the extra load from the Pi's), I'm running PostreSQL in Docker in my own Mac. This is the docker-compose file I've used:

```yaml
version: "3.0"
services:
  postgres:
    image: postgres
    restart: always
    volumes:
      - pg:/var/lib/postgresql/data/pgdata
    ports:
      - "5450:5432"
    environment:
      POSTGRES_PASSWORD: postgres
      PGDATA: /var/lib/postgresql/data/pgdata

volumes:
  pg:
```

After spinning up the PostgreSQL container, you'll need to create a DB called `kubernetes`.

We now need to pass the DB connection string to the k3s server setup commands.

In this setup, since etcd isn't really running in the Kubernetes cluster, we only really need two nodes to achieve High Availability (even though we'd have to deal with PostgreSQL high availability as well... but that's a different topic).

The following command can be run in all nodes to create a cluster:

```bash
curl -sfL https://get.k3s.io | sh -s - server --token=<SECRET> --datastore-endpoint="postgres://postgres:postgres@<PostgreSQL Server IP Address>:5450/kubernetes?sslmode=disable"
```

In the end, this proved not to be enough. The rest of the Control Plane components are still too heavy for the Pi Zero's...

## Attempt #3: Splitting the control plane components over the different Pi's - Also failed

k3s provides a few useful flags to disable the Kubernetes Control Plane components. Plus, remember when I mentioned that k3s is a full-featured Kubernetes distro? What that means is that k3s comes bundled with a number of useful components that are commonly needed to run a production cluster: Load Balancer, Ingress Controller, Local Storage Provider and a Metrics Server.

While these are indeed needed in most situations where k3s is meant to be run, that's not really the case for my exercise where I'm learning about k3s sizing and cluster stability. Thus, my next though was to:

1. Disable all unnecessary components
1. Run the different Control Plane components on different Pi's

This has a number of disadvantages, namely the complete lack of High Availability with just 3 servers (what if the Pi running etcd goes down? Or the Pi running the API Server?). This is how you can run k3s, disabling, for example, the Scheduler and Traefik (the Ingress Controller):

```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --disable-scheduler --disable traefik
```

Even in this scenario it was still too much for the poor Pi Zero's. You can learn more about k3s feature flags [here](https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/) and about etcd only nodes [here](https://rancher.com/docs/k3s/latest/en/installation/disable-flags/)

## Attempt #4: Running the control plane on an external VM - Great Success 🚀

So... that was it. I had to accept that running the Control Plane on the Pi Zero's was a bit too much, for now... Which shouldn't be a surprise, given [k3s' minimum hardware requirements](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/resource-profiling/).

Nonetheless, I wanted to follow thorough with this. A highly available app deployment will still be running even if the Control Plane goes down! This means that we can run an entire master node externally and use the Pi's as worker nodes, to deploy highly available app workloads.

I've then created a VM on my Mac where I've installed a k3s master node:

```bash
curl -fL https://get.k3s.io | sh -s - server
```

And run the agent node installation on the Raspberry Pi's as follows:

```bash
curl -fL https://get.k3s.io | sh -s - agent
```

There is, of course, a bit more to this. k3s can consume installation configuration from a `config.yaml` file which is preemptively placed at `/etc/rancher/k3s/`.

The master node config file looks like this:

```bash
node-name: {{ inventory_hostname }}
cluster-init: true
node-taint: "CriticalAddonsOnly=true:NoExecute"
```

and the agent node config file looks like this:

```bash
node-name: {{ inventory_hostname }}
token: "{{ hostvars['vm']['token'].stdout }}"
server: "https://{{ hostvars['vm']['ansible_host'] }}:6443"
```

Also, a few boot command line options need to be set on all Raspberry Pi's. The `/boot/cmdline.txt` should look like this (notice the `cgroup_memory=1 cgroup_enable=memory` options at the end of the line):

```
console=serial-1,115200 console=tty1 root=PARTUUID=01decd83-02 rootfstype=ext4 fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory
```

By the way, did you notice the strange `{{ ... }}` syntax in some of the code sample above? This leads me to the next section...

# Automation

As always, my true documentation is on GitHub, delivery as a set of Ansible Playbooks. You can [take a look at it, clone it and/or fork it here](https://github.com/aveiga/rpizero-k3s-cluster).

This has been thoroughly tested running the Playbooks from my M1 Macbook Air, and will result on a VM running the Kubernetes cluster master node (unschedulable for app workloads) and the three Raspberry Pi's running the cluster's worker nodes.

Don't forget to adapt the [hosts.yaml](https://github.com/aveiga/rpizero-k3s-cluster/blob/main/hosts.yaml) file to your own environment, as well as recreating the Ansible [Vault](https://github.com/aveiga/rpizero-k3s-cluster/blob/main/group_vars/all/vault) defining your own variables:

```yaml
vaulted_become_password: TBD # The password used to run commands with elevated privileges
vaulted_ssh_user: TBD # The user used to SSH into the Pi's
```
