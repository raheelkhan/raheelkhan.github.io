---
title: Automate AWS AMI provisioning with Packer and CodeBuild
date: 2023-01-06
categories:
---

[Packer](https://www.packer.io/) is a tool by HashiCorp that can be used to automate the process of creating Amazon Machine Images (AMIs). One of the benefits of using Packer to create AMIs with your application code pre-installed is that it can help speed up the process of launching EC2 instances in order to meet production traffic needs. This is because Packer allows you to build AMIs in a repeatable and automated way, so that you can quickly and easily launch new instances with your application code already installed and configured. This can be particularly useful in scenarios where you need to scale up your infrastructure quickly in order to meet a sudden increase in traffic. Additionally, Packer allows you to test your AMIs before they are deployed to ensure that they are fully functional, which can help reduce the risk of launching instances that are not ready for production use. Overall, using Packer to create AMIs with your application code can help you more efficiently and effectively meet the demands of your production environment.

In this article, I am explaining the process of creating golden ami with a laravel applicaton preconfigured in it. I will also explain how to automate the process using AWS CodeBuild.

## Prerequisite 

Before we begin, you must have an AWS account. This is a level 200 tutorial and I assume you know how to setup an AWS account and IAM users with CLI access. You will also need to setup AWS CodeCommit cli credentials.

- IAM user with access `AdministratorAccess`. In production you will be using PoLP
- Git
- PHP > 7.3
- Composer - PHP package manager

> The use of Laravel is just by choice and is not mandatory

## A word about Packer workflow

Hashicorp Packer is a tool for creating machine and container images for various platforms from a single source configuration. Packer works by creating a temporary environment, known as a "builder," in which it installs the necessary tools and packages for the image, then runs a series of provisioning scripts to configure the system. Finally, Packer creates an image file that contains the contents of the builder environment, which can then be used to create new servers or containers.

Here is a high-level overview of how Packer works:

1. The user specifies the configuration for the image they want to create in a Packer template file. This includes the builder(s) to use, the provisioning scripts to run, and any other necessary settings.

2. Packer reads the template and creates a builder environment for each specified builder. The builder environment is typically a temporary, throw-away environment that is created specifically for the purpose of creating the image.

3. Packer installs any necessary tools and packages in the builder environment, such as the operating system, package manager, and other dependencies required by the provisioning scripts.

4. Packer runs the provisioning scripts in the builder environment to configure the system according to the user's specifications. This can include tasks such as installing and configuring software, setting up users and permissions, and other customizations.

5. Once the provisioning is complete, Packer creates an image file from the contents of the builder environment. This image file can then be used to create new servers or containers on the target platform.

## Lets get to work

We'll being by creating a fresh Laravel project in our local environment

```bash
composer create-project laravel/laravel packer-laravel
```
Once the installer is completed, verify if the application works. You should be able to see Laravel welcome page

```bash
cd packer-laravel && php artisan serve
```

Now that we have application up and running, Lets put the code in a AWS CodeCommit repository. 

```bash
respository=$(aws codecommit create-repository --repository-name packer-laravel --repository-description "Packer Laravel Application" | jq -r ".repositoryMetadata.cloneUrlSsh")
git init .
git add .
git commit -m "Initial Commit"
git remote add origin $repository
git push origin master
```

Most of our work here will be based on CLI, so lets create a folder called `scripts` within your project root directory. The scripts directory will have the following files

```bash
scripts
├── aws-ubuntu.pkr.hcl
├── buildspec.yml
├── codebuild-create-role.json
├── codebuild-put-role-policy.json
├── codebuild.json
└── script.sh
```

The first file we will work is `aws-ubuntu.pkr.hcl`. The first block of code sets a default value for a variable called "source_dir", which is the directory containing the source code for the image. This default value is the directory specified in an environment variable called "CODEBUILD_SRC_DIR" which is where AWS CodeBuild clone the repository of your SOURCE which in our case is AWS CodeCommit.

The next block of code specifies that Packer requires a specific plugin called "amazon-ebs" in order to build the image. This plugin must be version 0.0.2 or higher, and it can be found on GitHub at the specified URL.

The next block of code defines the source for the image, which is an Amazon Elastic Block Store (EBS) volume with an Ubuntu operating system. It specifies various properties of the volume, such as the name of the image, the type of instance it will run on, the region it will be located in, the username for SSH access, and the source Amazon Machine Image (AMI) that it will be based on.

The final block of code defines the build process for the image. It specifies the name of the image and the source for it, which is the Amazon EBS volume defined earlier. It also includes three provisioners: a shell script that creates a directory, a file provisioner that copies the source code directory to this new directory, and another shell script that runs a script called "script.sh" within the temporary EC2 instance which Packer creates for provisioning machine image. This build process will create a machine image with the specified name, based on the specified source AMI, and with the specified source code and scripts included in it.

Now lets go through the content of `script.sh` file. This file contains comment that explains the purpose of each step

```bash
#!/bin/bash

# Give some time for the temporary instance to boot
sleep 30

# Lets get latest updates
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update -y

# Install our webserver and version control
sudo apt-get install -y git nginx php7.3-fpm

# Test if nginx is actually serving on port 80
if curl -s http://localhost:80 | grep -q "Welcome to nginx!"; then
  echo "Nginx working"
else
  echo "Ngnix not working"
fi

# Copy our app into web directory
sudo mv /tmp/packer-laravel /var/www/
ls -l /var/www

# Laravel bootstrapping
sudo chown -R www-data:www-data /var/www/packer-laravel/storage/
sudo cp /var/www/packer-laravel/.env.example /var/www/packer-laravel/.env
sudo php /var/www/packer-laravel/artisan key:generate

# Nginx configuration for our laravel site
cat << EOF | sudo tee -a /etc/nginx/sites-available/packer-laravel
server {
    listen 80;
    listen [::]:80;
    server_name _;
    root /var/www/packer-laravel/public;
 
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
 
    index index.php;
 
    charset utf-8;
 
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
 
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
 
    error_page 404 /index.php;
 
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
 
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
EOF
sudo rm /etc/nginx/sites-enabled/default /etc/nginx/sites-available/default
sudo ln -s /etc/nginx/sites-available/packer-laravel /etc/nginx/sites-enabled/
sudo systemctl restart nginx

# Start nginx server and php-fpm and enable it for reboot
sudo systemctl enable nginx
sudo systemctl enable php7.3-fpm.service
```

We have scripted the steps for Packer to work on. Its time to automate the process by using AWS CodeBuild. Ofcourse, you can go a step further and put a end to end pipeline in place using AWS Codepipeline on your own.

In order to create a CodeBuild project lets create a file `codebuild.json`. We will use AWS CLI to create CodeBuild project which is defined in the json format in this file

```json
{
    "name": "packer-laravel",
    "description": "A dummy laravel application built with packer",
    "source": {
        "type": "CODECOMMIT",
        "location": "https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/packer-laravel",
        "buildspec": "scripts/buildspec.yml"
    },
    "artifacts": {
        "type": "S3",
        "location": "your-bucket-name",
        "namespaceType": "NONE"
    },
    "environment": {
        "type": "LINUX_CONTAINER",
        "image": "aws/codebuild/standard:4.0",
        "computeType": "BUILD_GENERAL1_SMALL"
    },
    "serviceRole": "arn:aws:iam::your_aws_account_id:role/CodeBuildServiceRole",
    "logsConfig": {
        "cloudWatchLogs": {
            "status": "ENABLED",
            "groupName": "packer-laravel"
        }
    }
}
```

Notice that CodeBuild requires a S3 Bucket for storing artifacts. You can pass any bucket that you own, or you can create a new bucket by running the following command

```bash
aws s3api create-bucket \
    --bucket your-bucket-name \
    --region eu-west-1 \
    --create-bucket-configuration LocationConstraint=eu-west-1
```

Because we need to provide CodeBuild a service role to interact with other AWS services, lets create `CodeBuildServiceRole` and `CodeBuildServiceRolePolicy`

In file `codebuild-create-role.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

and file `codebuild-put-role-policy.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogsPolicy",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CodeCommitPolicy",
      "Effect": "Allow",
      "Action": [
        "codecommit:GitPull"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3GetObjectPolicy",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3PutObjectPolicy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3BucketIdentity",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketAcl",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CopyImage",
        "ec2:CreateImage",
        "ec2:CreateKeypair",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteKeyPair",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DeregisterImage",
        "ec2:DescribeImageAttribute",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSnapshots",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume",
        "ec2:GetPasswordData",
        "ec2:ModifyImageAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifySnapshotAttribute",
        "ec2:RegisterImage",
        "ec2:RunInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

Run the following command to create AWS CodeBuild project
```bash
aws codebuild create-project --cli-input-json file://scripts/codebuild.json
```

We will define instructions for CodeBuild in file `buildspec.yml`

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      php: 7.3
  pre_build:
    commands:
      - php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
      - php composer-setup.php --install-dir=/usr/local/bin --filename=composer
      - php -r "unlink('composer-setup.php');"
      - curl -qL -o jq https://stedolan.github.io/jq/download/linux64/jq && chmod +x ./jq
      - curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
      - sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
      - sudo apt-get update && sudo apt-get install packer
  build:
    commands:
      - cd $CODEBUILD_SRC_DIR
      - composer install
      - echo "Configuring AWS credentials"
      - curl -qL -o aws_credentials.json http://169.254.170.2/$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI > aws_credentials.json
      - aws configure set region $AWS_REGION
      - aws configure set aws_access_key_id `./jq -r '.AccessKeyId' aws_credentials.json`
      - aws configure set aws_secret_access_key `./jq -r '.SecretAccessKey' aws_credentials.json`
      - aws configure set aws_session_token `./jq -r '.Token' aws_credentials.json`
      - cd $CODEBUILD_SRC_DIR/scripts
      - packer init .
      - packer fmt .
      - packer validate .
      - packer build aws-ubuntu.pkr.hcl
  post_build:
    commands:
      - echo Build completed on `date`
```

Thats it, you can go to AWS console CodeBuild servie. You should see your project there in the region you specified. Try hitting the `Start Build` button and see the logs. If every thing works (Hopefully), then you will have a custom ami name `packer-laravel` created in your account.

For the curious out there. You can try creating your Autoscaling Group and in the Launch Template, specify the ami id of your newly generated image. You will notice that with this approach the time it takes for EC2 instance to bootup is reduced.

Happy Provisioning ! 