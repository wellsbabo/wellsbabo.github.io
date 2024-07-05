I have built a system to deploy Spring Boot projects.
Since I went through some diggine, I summarized the process.

The configuration diagram of the system I built is as follows:
![deploy_spring_boot_web_application_diagram.png]({{site.baseurl}}/_posts/deploy_spring_boot_web_application_diagram.png)

I am planning to set up a server with an SSH certificate, and the server will be using Rocky Linux.

In Linux, Nginx is also installed and sent to each port according to the request domain, and Spring Boot processes the request on that port.

Now I will start explaining step by step.


## 1. Linux Java Installation
First, check whether Java is already installed on the Linux server, and if not, install the desired version.

```bash
# Check if Java is already installed
yum list installed | grep java

# If it is not the version you want, delete the installed version.
sudo yum remove -y [패키지명]

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