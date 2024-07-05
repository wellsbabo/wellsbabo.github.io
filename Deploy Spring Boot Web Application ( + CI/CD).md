
I have built a system to deploy Spring Boot projects.
Since I went through some diggine, I summarized the process.

The configuration diagram of the system I built is as follows:
![deploy_spring_boot_web_application_diagram.png]({{site.baseurl}}/Deploy Spring Boot Web Application ( + CI/deploy_spring_boot_web_application_diagram.png)

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
# etc/profile
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