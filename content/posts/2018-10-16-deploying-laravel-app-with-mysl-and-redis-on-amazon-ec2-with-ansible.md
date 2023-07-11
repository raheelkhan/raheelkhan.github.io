---
title: Deploying Laravel app with MySQL and Redis on Amazon EC2 with Ansible
date: 2018-10-16
categores:
---

Ansible is a configuration management tool that automates the deployment process and can be easily scaled. The way Ansible works it connect to your remote host(s) via SSH and runs your deployment steps there. You can add up multiple hosts depends on how many forks can be created on your control machine (i.e from where you are executing Ansible commands).

This tutorial is a step by step guide to deploy Laravel application on AWS EC2 instance with Ansible.

For the sake of this tutorial, I will be created 3 EC2 instances

1. web – This instance will have PHP, Nginx install along with our Laravel application
2. database – This instance will have MySQL installation
3. cache – This instance will contain Redis installation

### Creating EC2 instances on AWS

For learning purpose we can use AWS free tier . Since this post is more focused on using Ansible,I leave it on you to spin up your instances. For reference please see the video lesson Video Lesson on AWS Free Tier

> Ubuntu 18.04 comes with Python3 as default. If you have install Ansible in your control machine through pip3, you should be able to connect without hassle. If your Ansible is configured to use Python2 then you have to install Python2 in your host machines on EC2.

You can install Python2 on Ubuntu 18.04 by running

```
sudo apt install python-minimal
```

I have installed Ansible via brew package manager that by default uses Python2 so I had to install Python2 manually on my hosts machines

### Creating Laravel Project Locally

I prefer to install Laravel through Composer. If you have PHP & Composer installed in your machine you can run the below command to create a Laravel project.

```
composer create-project --prefer-dist laravel/laravel laravel-ansible "5.6.*"
```

In order to verify if the installation is successful you can run

```
cd laravel-ansible; php -S localhost:8080
```

This will spin up a builtin PHP development server. Visit http://localhost:8080 and you should be able to see Laravel home page.

### Installing Ansible Locally

Usually in continuous delivery environment you should have a dedicated machine responsible for running your Ansible playbooks typically triggered by your version control software. For example when some new code is merged into master. For this tutorial we will run Ansible from our local machine.

In order to install Ansible in mac we can run

```
brew install ansible
```

You can verify your Ansible version like this

![Ansible](/img/ansible-sc.png)

### Creating Directory Structure for Ansible files

In your project root create a folder arbitrary named deployment. This folder will be part of our version control. We will be dividing our deployment tasks by [Ansible Roles](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_reuse_roles.html)

Recall our hosts are divided into web, database, cache. Hence we will create three roles for each purpose. The final directory tree will look like

![Laravel Project With Ansible](/img/ansible-laravel-tree-sc.png)

### Defining Hosts

Our Inventory file is the hosts file in deployment folder. Here we will list our 3 EC2 instances and alias it accordingly.

```
web ansible_host=ec2-13-250-113-129.ap-southeast-1.compute.amazonaws.com ansible_port=22
database ansible_host=ec2-54-169-203-141.ap-southeast-1.compute.amazonaws.com ansible_port=22
cache ansible_host=ec2-52-77-251-193.ap-southeast-1.compute.amazonaws.com ansible_port=22
```

### Ansible Config

Ansible assumes that the config file ansible.cfg is adjacent to the playbook being run. If not, it will search other paths as well such as ~/etc/ansible/ansible.cfg. Here we will define some basic configurations

```ini
[defaults]
inventory=hosts
remote_user=ubuntu
host_key_checking=True
private_key_file=~/.ssh/laravel-ansible.pem
```

Lets quickly see whats in the config. We told Ansible which file to look for inventory which in this case a file in same directory called hosts. Next we defined that the user for SSH connection to our EC2 instances should be ubuntu (I am using ubuntu 18.04 t2.micro instances for this demo). If we don’t explicitly define the remote user, Ansbile will attempt to connect to remote machines using your local machine name which in my case is raheel. Further, We set the host_key_checking flag to True. This will help us to pass through the interactive session when the EC2 instance is not present in your machine’s known_hosts file. For this tutorial, I have generated the same private key for my all three instances and moved into my machine’s ~/.ssh directory. Ansible will use this key to connect to the EC2 machines.

### Working on roles

As mentioned we have three roles web, database, cache we will start off with the web role.

#### Web

create a file in `deployment/roles/web/tasks` called `main.yml` and paste the following tasks

```yaml
- name: install php with required extensions and nginx
  apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
  with_items:
    - php
    - php-common
    - php-fpm
    - php-mbstring
    - php-xml
    - php-zip
    - nginx
    - git
    - composer
  become: True

- name: remove apache2 installation from ubuntu 18.04
  apt: pkg=apache2 state=absent purge=yes
  become: True

- name: copy nginx config file
  copy: src=nginx.conf dest=/etc/nginx/sites-available/default
  become: True

- name: enable nginx configuration
  file: >
    dest=/etc/nginx/sites-enabled/default
    src=/etc/nginx/sites-available/default
    state=link
  notify: restart nginx

- name: create project directory
  file: path={{ remote_source_path }} state=directory

- name: checkout latest code from github
  git: >
    repo=git@github.com:raheelkhan/laravel-ansible.git
    dest={{ remote_source_path }}
    force=yes
    accept_hostkey=yes

- name: install laravel dependencies via composer
  composer: command=install working_dir={{ remote_source_path }}

- name: copy .env.production to .env
  copy: >
    src="{{ remote_source_path}}/.env.production"
    dest="{{ remote_source_path}}/.env"
    remote_src=yes

- name: give writeable permissions to laravel storage directory
  file: >
    path={{ remote_storage_path }}
    mode=0775
    owner=ubuntu
    group=www-data
    state=directory
    recurse=yes
  become: True
```

Lets go quickly what’s going on here.

We started with installing Nginx, PHP with required extensions as mentioned in [Laravel Dependencies](https://laravel.com/docs/5.7/installation#server-requirements) . Notice we missed some of the extensions as they comes by default in PHP installations unless you are not recompiling php your self. Ubuntu 18.04 comes with PHP 7.2.10 as of now.
Next we removed and purge the default apache2 installation from our remote server so that it does not conflict with our Nginx installation. Make sure you use the purge option otherwise you may see Apache’s landing page when visiting your server port 80.
Then we copied Nginx configuration file from control machine to host in order to serve our Laravel application followed by linking it to the /etc/nginx/sites-enabled/default file. You need to create this config file in directory deployment/role/web/files/nginx.conf with the following content

```conf
server {

    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /home/ubuntu/laravel-ansible/public;

    index index.php index.html index.htm;

    server_name localhost;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

}
```

After this, we create a directory where our code will reside. The command for this is self explanatory except we are using a variable for the directory path **{{ remote_source_path }}**. This variable you will have to define at **deployment/role/web/vars/main.yml**

```yaml
remote_source_path: /home/ubuntu/laravel-ansible
remote_storage_path: "{{ remote_source_path }}/storage"
```

> Github below refers to all VCS services such as Bitbucket or Gitlab

After creating the directory the next step is to pull the latest code from your VCS into your host machine. This is a bit tricky step. Since in private repos you need to authenticate your machine on Github etc in order to clone / push code. Otherwise services like Github or Bitbucket will throw the access denied error. Authentication can be done either by using username and password i.e using HTTPS or by adding your development machines public key in the settings of Git service.

The problem now is Ansible runs the deployment script on your host machine. When it tries to fetch the code via git module, the Github server will reject as it will not be able to recognise the host machine.

This problem can be solved by different ways

1. Generate public key for all your host machines and add them into your Github SSH settings. This could be very time taking. Imagine if you are keep on changing your instances.
2. Copy your control machine’s public key can be found in ~/.ssh/id_rsa.pub on all your host machines. This will have a potential risk. If the host machine’s somehow compromised, It will expose your public key as well.
3. Use SSH Forwarding. This is the most easy and secure way. By allowing SSH Forwarding, you forward your public key on host machine for the runtime of Ansible scripts. The host machine will communicate with Github as if it is from your own machine. In order to enable SSH Forwarding you need to add your identity to the running ssh-agent by running the ​ssh-add command. Verify the identity by running commad ssh-add -l. You should see a SHA265 string. Also notice the ansible.cfg file content above to let Ansible use th SSH Forwarding.

Rest of the steps are self explanatory.

In order to run this role, we need to reference this web role inside our playbook.

In the deployment folder as shown in the screenshot of directory tree, create file called `production.yml` and add the following content.

```yaml
- name: web plays
  hosts: web
  gather_facts: True
  roles:
    - web
```

Go on and run this playbook from inside the deployment folder

If there is no error, you should see ansible started to run your plays sequentially.

![Anisble Playbook Running](/img/ansible-playbook-sc.png)

#### Database

For the database we are going to create a role that will install MySQL on our debian server as well as run the [MySQL Secure Installation](https://dev.mysql.com/doc/refman/5.7/en/mysql-secure-installation.html) steps via our tasks.

Create file `deployment/database/tasks/main.yml` with the following content

```yaml
- name: update apt cache and install MySQL
  apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
  with_items:
    - mysql-server
    - python-pip
    - default-libmysqlclient-dev
  become: True

- name: install MySQL-python module
  pip: name=MySQL-python

- name: start mysql service
  service: name=mysql state=started enabled=yes

- name: set the root password
  mysql_user: >
    name=root
    password="{{ mysql_root_password }}"
    check_implicit_admin=yes
    host={{ item }}
    priv="*.*:ALL,GRANT"
  with_items:
    - "{{ ansible_host }}"
    - 127.0.0.1
    - ::1
    - localhost
  become: True

- name: copy root credentials to /root/.my.cnf
  template: src=.my.cnf.j2 dest=/root/.my.cnf owner=root mode=0600
  become: True

- name: delete anonymous user
  mysql_user: name="" state=absent host={{ item }}
  with_items:
    - "{{ ansible_host }}"
    - 127.0.0.1
    - ::1
    - localhost
  become: True

- name: delete test database
  mysql_db: name=test state=absent
  become: True
```

Following are contents of variable file and `.my.cnf.j2` template respectively.

create file `deployment/database/defaults/main.yml` with

```yaml
mysql_root_password: mysecretpassword
```

and `deployment/database/templates/.my.cnf.j2` with

```ini
[client]
user=root
password={{ mysql_root_password }}
```

As usual, we have to reference this role in our main playbook as

```yaml
- name: web plays
  hosts: web
  gather_facts: True
  roles:
    - web

- name: database plays
  hosts: database
  gather_facts: True
  roles:
    - database
```

In order to run only database role, we can run the command

```
ansible-playbook -l database production.yml
```

For detailed reference of mysql modules visit [here](https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html) and [here](https://docs.ansible.com/ansible/2.5/modules/mysql_db_module.html)

#### Cache

While I was writing this post, I changed my mind to not to write Redis role myself rather use a role provided by ansible community via [Ansible Galaxy](https://galaxy.ansible.com/)

There are obvious benefits of using community provided roles as they cover major OS as well as we get to learn how to write generic roles.

Ansible ships with a command line tool ansible-galaxy. We can go to the search page of ansible and find our desired roles, modules etc.

I decided to use [https://galaxy.ansible.com/geerlingguy/redis]

Delete the cache role inside deployment folder and run the following command from within deployment directory.

```
ansible-galaxy install -p ./roles geerlingguy.redis
```

The -p argument tells ansible galaxy to install the role insides roles folder in currenty directory. Otherwise, it will install it globally.

Once done, you will see a folder name `geerlinguy.redis` inside your roles folder. If you wish you can rename it to `cache` again.

You can view the contents of this role and tweak it as per your needs. In my case, I have to put `become: True` in some tasks as I do not run the whole playbook as root.

As usual, we will reference this role in our playbook as

```yaml
- name: web plays
  hosts: web
  gather_facts: True
  roles:
    - web

- name: database plays
  hosts: database
  gather_facts: False
  roles:
    - database

- name: cache plays
  hosts: cache
  gather_facts: True
  roles:
    - cache
```

We are done with our playbook. If all goes well, you will be able to successfully run it and have your Laravel application deployed with MySQL and Redis on different servers.

> [Ansible Up & Running 2nd Edition](http://shop.oreilly.com/product/0636920065500.do) Highly Recommended
