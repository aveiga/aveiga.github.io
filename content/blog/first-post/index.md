---
title: Installing K3s and Knative on ARM
date: "2022-01-16T19:00:00.000Z"
description: "Setting up Knative on Ubuntu Server 20.04 running on ARM"
---

## Installation

This past week I wanted to try out [Knative](https://knative.dev/docs/), a "Kubernetes-based platform to deploy and manage modern serverless workloads". In essence, what I'm looking forward is a higher level abstration for running applications/microservices without worrying with Kubernetes inherent complexities.

I've been using [K3s](https://k3s.io), a light-weight Kubernetes distribution built by the team behind [Rancher](https://rancher.com) for all of my deployments for a while. It's easy and fast to install, highly scalable and has a minimal footprint, perfect for on-premises and self-managed deployments.

K3s comes with it's own network layer, provided by Traefik. However, since Knative needs to takeover the networking layer in order to successfully run applications as Knative Services, Traefik needs to be disabled. It's possible to install K3s without Traefik by running the following command:

```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

After this, to install Knative, just follow the [Knative installation guide](https://knative.dev/docs/install/serving/install-serving-with-yaml/).

Finally, to try out and "smoke test" the installation, I've created the following Knative Service definition (which deploys Nginx as a Knative Service)...

```yaml
# hello.yaml
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      # This is the name of our new "Revision," it must follow the convention {service-name}-{revision-name}
      name: hello-nginx
    spec:
      containers:
        - image: nginx
          ports:
            - containerPort: 80
```

...and applied it with the the following command:

```bash
sudo kubectl apply -f hello.world
```

You can then get the application URL by listing the Knative Services (note that you'll need the Knative CLI, `kn`, installed):

```bash
kn service list
```

## Automation

I've created a set of [Ansible](https://www.ansible.com) playbooks to deploy K3s, Knative (and a "smoke test" application) on ARM and made them available in [this GitHub repo](https://github.com/aveiga/k3s-knative-arm). Hope it makes your life easier, when testing Knative.
