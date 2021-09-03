# Introduction
You are here to learn to setup your own jenkins instance and my experience with jenkins in setting up continuous integration framework for Linux kernel on IBM POWER system

# Prerequisites:
  * Install Linux OS of your choice, I have installed RHEL8.x on PowerVM LPAR
  * Setup yum repositories of the OS installed
  * Install distro supported `java-11-openjdk java-11-openjdk-devel` packages
  * Make sure port 80 and port 8080 are open in your server firewall
 
# Jenkins installation

  * Setup jenkins packages repo and install LTS version of jenkins package, I have `jenkins-2.277.2-1.1` on my server
      ```bash
      $ wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo
      $ rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
      $ yum install jenkins
      ```
  * Edit below parameters in `/etc/syscofing/jenkins`
     ```
      JENKINS_HOME="/home/jenkins"
      JENKINS_USER="root"
     ```

  * Disable firewall and start jenkins daemon
      ```bash
      $ systemctl stop firewalld
      $ systemctl disable firewalld
      $ systemctl start jenkins 
      $ systemctl enable jenkins
      ```
  * Now access the jenkins page on port 8080 using your server ip `http://<jenkins-ip>:8080` via a browser
  * Get the secret key in file `/var/lib/jenkins/secrets/initialAdminPassword` from terminal and provide it in the jenkins page
  * Installed required plugins chosing custom plugin option and complete the jenkins installation following steps [link](https://devopscube.com/install-configure-jenkins-2-centos-redhat-servers)
  * Enable LDAP based security Goto Manage Jenkins > Configure Global Security
  * Enable Project based Matrix Authorization

# Configure HTTPS for jenkins

  * Create self signed certificate using java keytool [details]# Configure HTTPS for jenkins [details](https://aboutssl.org/how-to-create-a-self-signed-certificate-using-java-keytool/)
      ```bash
      keytool -genkey -keyalg RSA  -alias selfsigned -keystore jenkins.jks -storepass <passwd>  -validity 3650 -keysize 2048
      ```
  * Stop the jenkins server `service jenkins stop`
  * Create a direcotry `mkdir -p /etc/jenkins`
  * Copy the .jks file created by keytool command to `/etc/jenkins/`
  * Update jenkins configuration file `/etc/sysconfig/jenkins` with below parameters
      ```
      JENKINS_HTTPS_KEYSTORE="/etc/jenkins/jenkins.jks"
      JENKINS_HTTPS_KEYSTORE_PASSWORD="abcdef"
      JENKINS_PORT="-1"
      JENKINS_HTTPS_PORT="8080"
      ```
  * Restart jenkins and login again `https://<jenkins-ip>:8080` using your intranet userid and password
  * Setup HTTP server on jenkins
      ```
      $ yum install httpd
      Edit /etc/httpd/conf/httpd.conf and add ServerAdmin and Listen port 81
      Enable httpd service : systemctl enable httpd
      Start httpd service : systemctl start http
      ```
  * Setup slack notification in jenkins for your channel follow this [link](https://slack.com/apps/A0F7VRFKN-jenkins-ci?next_id=0&tab=settings)

# CI project specific configurations

  * Install the below needed plugins
      ```
      pipeline, git, envinject, matrix-auth 
      Node and label parameter
      Job and Node ownership
      Build Publisher
      Slack Notification
      Build with Parameters
      Extended choice parameters
      Labeled Test Groups Publisher
      Test Results Analyzer
      Environment Script
      Rebuilder
      Flexible Publish
      Workspace cleanup
      ```

  * Install required pacakges

  ```bash
  $ yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
  $ yum install python-pip
  $ pip install pexpect
  $ pip install pypy3
  $ ln -s /usr/bin/python3.7 /usr/bin/python
  ```

Thanks for reading my blog, hope this helps you setup your jenkins instance !
