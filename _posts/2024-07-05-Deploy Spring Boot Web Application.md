---
layout: post
title:  Deploy spring boot web application
categories: [spring-boot, Backend Development]
tags: [spring-boot, deploy, Nginx]
description: Deploying Spring Boot with Nginx (Including Simple CI/CD)
---

I have built a system to deploy Spring Boot projects.
Since I went through some diggine, I summarized the process.

The configuration diagram of the system I built is as follows:
![deploy_spring_boot_web_application_diagram.png](assets/img/posts/240916/deploy_spring_boot_web_application_diagram.png)

I am planning to set up a server with an SSH certificate, and the server will be using Rocky Linux.

In Linux, Nginx is also installed and sent to each port according to the request domain, and Spring Boot processes the request on that port.

Now I will start explaining step by step.


## 1. Linux Java Installation
First, check whether Java is already installed on the Linux server, and if not, install the desired version.

```bash
# Check if Java is already installed
yum list installed | grep java

# If it is not the version you want, delete the installed version.
sudo yum remove -y [package name]

# Check the list of jdk versions that can be installed using yum
yum list java*jdk-devel

# Install the version you want (I chose Java 17)
sudo yum install java-17-openjdk-devel.x86\_64
```

And set the environment variables.

```bash
# Edit /etc/profile
sudo vi /etc/profile
```
```bash
# Add to etc/profile
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.9.0.9-2.el8_8.x86_64
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar
```
```bash
# Apply modified profile
source /etc/profile

# Check environment variables
echo $JAVA_HOME
```

## 2. Install maven
Since I wrote Spring Boot based on Maven, I will install Maven on the server.

[maven install link](https://archive.apache.org/dist/maven/?source=post_page-----e1e1b96298d4--------------------------------)

Access the link above, find the desired maven version, and copy the path to the tar.gz file. Then proceed with installation by following the steps below.

```bash
# The installation path may change, so check the URL before downloading.
wget https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz -P /tmp

# Unzip the downloaded file
sudo tar xf /tmp/apache-maven-3.9.6-bin.tar.gz -C /opt

# Create links for ease of use
sudo ln -s /opt/apache-maven-3.9.6 /opt/maven
```

And set the environment variables as you did when installing Java.

This time, we will create a separate file called `maven.sh` in the `/etc/profile.d `directory and set the environment variables.

Since everything in `/etc/profile` and `/etc/profile.d` is called anyway, it is okay to set it like this.

In Linux, if possible, it is recommended to create a new file in `/etc/profile.d/` and not to touch `/etc/profile` itself.

```bash
# Create maven.sh file
sudo vi /etc/profile.d/maven.sh
```
```bash
# Add to maven.sh
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.9.0.9-2.el8_8.x86_64
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
```

Then, give execution permission to `maven.sh` and apply environment variables.

```bash
sudo chmod +x /etc/profile.d/maven.sh

source /etc/profile.d/maven.sh
```

## 3. Install Nginx

As briefly mentioned at the beginning, Nginx is used as a reverse proxy server to classify requests by domain and deliver them to the corresponding internal Spring boot services.

First, install nginx via dnf using the commands below.

```bash
dnf update
dnf module list nginx
dnf install nginx
```

Once the installation is complete, we now need to modify the nginx configuration file.

The default configuration file location is `/etc/nginx`. 

`nginx.conf` here is the default configuration file, but in case you need to configure multiple services or create a configuration file for each service, comment out the server content in the default configuration file and create a new configuration file in the `conf.d` directory. to add content.

![nginx_config.png](assets/img/posts/240916/nginx_config.png)

Due to the blue underlined part in the picture above, if you create a configuration file with the `.conf` extension in the `conf.d` directory in the future, it will be read. (The default settings have been commented out.)

Let's assume that you created a configuration file called `default.conf` under the `conf.d` directory. The content below is just an example, so when actually writing it, be sure to understand the domain and settings you use well.

```bash
# When a request comes in on port 443 with SSL applied
server{
        listen 443 ssl;
        server_name     wellsbabo.com; # Change based on domain
        ssl_protocols   TLSv1.2;
        # Change according to ssl file name
        ssl_certificate /usr/ssl/wellsbabo.pem;
        ssl_certificate_key /usr/ssl/wellsbabo.key;
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 5m;
        ssl_ciphers HIGH:MEDIUM:!SSLv2:!PSK:!SRP:!ADH:!AECDH;
        ssl_prefer_server_ciphers on;
        server_tokens off;
        # Separated according to the domain from which the request came
        # In the example below, when a request is sent to wellsbabo.com/aaa/, the request is forwarded to the internal local server localhost:10001/aaa/.
        location ^~ /aaa/ {
                proxy_pass http://localhost:10001/aaa/;
        }

        location / {
                return 404;
        }
}

# When a request comes in on port 80 where SSL is not applied
server{
        listen 80;
        server_name _;
        server_tokens off;
        location ^~ /aaa/ {
                proxy_pass http://localhost:10001/aaa/;
        }
        location / {
                return 404;
        }
}
```

Once you have completed writing the configuration file, you can finally start the nginx service using the command below.

```bash
systemctl enable nginx
systemctl start nginx
systemctl status nginx # Check nginx status
```

## 4. Install git and build CICD (briefly)

First, install git on the server using the command below.
```bash
sudo dnf install git
```

After installing git, create an Ansible script according to the project's build process, create a repository, and upload the script.

Before uploading the script, build and deploy it in the order of the Ansible script you wrote, and test whether it works properly.

Once the test is complete, create a new Item in Jenkins and proceed with the build using the previously written script.

I wrote it very briefly, but I will write more details later.