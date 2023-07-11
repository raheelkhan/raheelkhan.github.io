---
title: Why I used Podman for Ansible practice
date: 2023-01-17
categories:
---

I was trying to do some ansible stuff for learning purpose. My tasks was to do some provisioning on multiple hosts. At first I thought to spin up some t2.micro instances but later I realized that I do not have free tier anymore. Naturally I decided to run multiple containers and set them as hosts for my ansible project. Because I wanted to try Redhat based systems I pulled Redhat official image from docker hub [Redhat Universal Base Image](https://hub.docker.com/r/redhat/ubi8).

Ansible has integration with docker and it can simply be configured by telling ansible to use `docker` as a connection mechanism for a certain host.

```ini
[containers]
redhat

[containers:vars]
ansible_connection=docker
```

To try out my connection I wrote a simple playbook to test out if my control node can actually ping docker container.

```yaml
---
- hosts: containers
  tasks:
    - name: Ping docker container
      ansible.builtin.ping:
```

And it worked fine.  I then started writing my first play which is to install nginx webserver 

```yaml
---
- hosts: containers
  tasks:
    - name: Installing nginx server
      ansible.builtin.yum:
        name:
          - nginx
        state: present
    - name: Make sure nginx is started
      ansible.builtin.service:
        name: nginx
        state: started
``` 

And to my surpirse I got an error in the second tasks

> Service status unknown

I logged in inside container and to debug what is the issue with nginx I ran command `systemctl status nginx` and it gives me error 

> System has not been booted with systemd as init system (PID 1). 
Can't operate. Failed to connect to bus: Host is down

I was seeing this error first time and had no idea what is it about. But thanks to online forums and documentations I understood what is happeing.

## What is systemctl anyway
systemctl is a command line utility used to control and manage system services and daemons on Linux operating systems. It allows you to start, stop, and check the status of services, as well as enable or disable them to start automatically at boot time. It is part of the systemd system and service manager.

So the question is why my docker container is not booted with systemd is the first place ?

## Docker philosophy
From the docs, A container’s main running process is the `ENTRYPOINT` and/or `CMD` at the end of the Dockerfile. It is generally recommended that you separate areas of concern by using one service per container. That service may fork into multiple processes (for example, Apache web server starts multiple worker processes). It’s ok to have multiple processes, but to get the most benefit out of Docker, avoid one container being responsible for multiple aspects of your overall application. You can connect multiple containers using user-defined networks and shared volumes.

Because a docker container generally supposed to do one task i.e one service, it by design does not run init services in background and hence we see the above error where it complains about system not being booted with systemd as init service. PID 1 is generally systemd in Linux OS but in this case of docker the PID 1 is assigned to the command or entrypoint you provide when running the container. You can verify this by running following command

```sh
docker pull redhat/ubi8
docker run -it redhat/ubi8 bash
yum install procps-ng
ps

PID TTY          TIME CMD
  1 pts/0    00:00:00 bash
  48 pts/0    00:00:00 ps
```

Notice the PID 1 here is `bash` we run the container with `bash` as entrypoint.

## Multi service container

Now that we understand what is blocking us to start nginx on the docker container, It now becomes obvious reason to use multi service container. 

Even discourged, Docker still supports multi service container but they are not straight forward. Specially that I am trying to focus on ansbile, I really did not want to configure by hand all the stuff to run systemd in my docker container. [Multi Service Container](https://docs.docker.com/config/containers/multi-service_container/)

## Using Podman
Podman is a containerization tool similar to Docker, but it does not require a daemon to run, it does not require root privileges and has built-in support for pods, rootless containers, **and multiple services in a single container**. It also supports IPv6, IPsec, VXLAN and has built-in support for Buildah, Skopeo and can be integrated with Kubernetes and OpenShift. It also has better handling of container storage compared to Docker. It can coexist with Docker on the same machine.


## Running Podman
```bash
brew install podman
podman machine start
podman pull registry.access.redhat.com/ubi8/ubi-init:latest
```

Now if you run the container you will notice that the systemd is actually already running in the container. Thanks to the specialized `ubi8/ubi-init:8.7-10` images by Redhat that comes handly for running multi service container.


### Ansible with Podman

Now that the container stuff is sorted, I tried running ansible to manage podman container 

```ini
[containers]
redhat

[containers:vars]
ansible_connection=podman
```

The only difference here is the `ansible_connection=podman`. Luckily Ansible comes with the connection plugin in its core installation. Otherwise you may have to pull it from Ansible Galaxy.

This time I tried my ansible playbook again and it was a success

```bash
podman run -it --name redhat -p 8080:80/tcp registry.access.redhat.com/ubi8/ubi-init
```

After running the playbook against my redhat container, now I get a success message. Also verified that the nginx welcome page is actually accessible on `localhost:8080`

```sh
PLAY [containers] ********************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
[WARNING]: Platform linux on host redhat is using the discovered Python interpreter at /usr/libexec/platform-python, but future installation of another
Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.13/reference_appendices/interpreter_discovery.html for
more information.
ok: [redhat]

TASK [Installing nginx server] *******************************************************************************************************************************
changed: [redhat]

TASK [Make sure nginx is started] ****************************************************************************************************************************
changed: [redhat]

PLAY RECAP ***************************************************************************************************************************************************
redhat                     : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

