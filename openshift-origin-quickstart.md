---
# Pre-Setup
---

Note1: This is setup for any of the quickstarts

- Setting hostname at `/etc/hosts` file, for example:
```
ip-address  domain-name.tld
```

- Setting hostname at server:
```
hostnamectl set-hostname domain-name.tld
hostname
```


- Install needed packages
```
  yum install centos-release-openshift-origin
  yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion origin-clients 
```


- Install and setup docker
```
  yum install docker
```

Edit `/etc/sysconfig/docker` file and add --insecure-registry 172.30.0.0/16 to the OPTIONS parameter.
```
  sed -i '/OPTIONS=.*/c\OPTIONS="--selinux-enabled --insecure-registry 172.30.0.0/16"' /etc/sysconfig/docker
  systemctl is-active docker
  systemctl enable docker
  systemctl start docker
```

---
# Setup
---

Pick One, don't do all four


OC CLUSTER
----

The oc cluster up command starts a local OpenShift all-in-one cluster with a configured registry, router, image streams, and default templates. By default, the command requires a working Docker connection

Note1: This section is still being worked on.
```
yum install centos-release-openshift-origin
yum install origin-clients
oc cluster up
```
Example Output:
```
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... OK
-- Checking for openshift/origin:v1.4.1 image ... 
   Pulling image openshift/origin:v1.4.1
   Pulled 0/3 layers, 3% complete
   Pulled 1/3 layers, 33% complete
   Pulled 2/3 layers, 86% complete
   Pulled 3/3 layers, 100% complete
   Extracting
   Image pull complete
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ... 
   Using nsenter mounter for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ... 
   Using 139.xx.xx.xxx as the server IP
-- Starting OpenShift container ... 
   Creating initial OpenShift configuration
   Starting OpenShift using container 'origin'
   Waiting for API server to start listening
   OpenShift server started
-- Adding default OAuthClient redirect URIs ... OK
-- Installing registry ... OK
-- Installing router ... OK
-- Importing image streams ... OK
-- Importing templates ... OK
-- Login to server ... OK
-- Creating initial project "myproject" ... OK
-- Removing temporary directory ... OK
-- Server Information ... 
   OpenShift server started.
   The server is accessible via web console at:
       https://139.xx.xx.xxx:8443

   You are logged in as:
       User:     developer
       Password: developer

   To login as administrator:
       oc login -u system:admin
```



Running in a Docker Container
----
```
docker run -d --name "origin" \
--privileged --pid=host --net=host \ -v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys -v /var/lib/docker:/var/lib/docker:rw \ -v /var/lib/origin/openshift.local.volumes:/var/lib/origin/openshift.local.volumes \ registry.centos.org/openshift/origin start
```

Running from a rpm
----

```
yum install origin
openshift start
export KUBECONFIG="$(pwd)"/openshift.local.config/master/admin.kubeconfig
export CURL_CA_BUNDLE="$(pwd)"/openshift.local.config/master/ca.crt
sudo chmod +r "$(pwd)"/openshift.local.config/master/admin.kubeconfig
```

Installer Installation Steps
----
Note1: This section is still being worked on. You will hit errors. It will fail. But we're working on it.
Note2: openshift-ansible can work on ansible 1.9.4 and above.

```
yum install centos-release-openshift-origin
yum --enablerepo=centos-openshift-origin-testing clean all
yum --enablerepo=centos-openshift-origin-testing install atomic-openshift-utils
atomic-openshift-installer install
```
  - answer questions as it steps you through an installation
  - prerequisites - https://docs.openshift.com/enterprise/3.2/install_config/install/prerequisites.html

---
# Testing
---


Quick Test 1
----

```
oc login
Username: test
Password: test
oc new-project test
oc new-app openshift/deployment-example
--> Found Docker image 1c839d8 (20 months old) from Docker Hub for "openshift/deployment-example"

    * An image stream will be created as "deployment-example:latest" that will track this image
    * This image will be deployed in deployment config "deployment-example"
    * Port 8080/tcp will be load balanced by service "deployment-example"
      * Other containers can access this service through the hostname "deployment-example"
    * WARNING: Image "openshift/deployment-example" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources ...
    imagestream "deployment-example" created
    deploymentconfig "deployment-example" created
    service "deployment-example" created
--> Success
    Run 'oc status' to view your app.
```

Cek Deployment status:
```
oc status

In project test on server https://139.59.243.79:8443

svc/deployment-example - 172.30.235.55:8080
  dc/deployment-example deploys istag/deployment-example:latest 
    deployment #1 deployed about a minute ago - 1 pod

2 warnings identified, use 'oc status -v' to see details.
```

Test app:
```
curl 172.30.126.164:8080 # (example v1) (Use URL that it gives you for svc/deployment-example)
oc tag deployment-example:v2 deployment-example:latest
curl 172.30.126.164:8080 # (example2 v2)
```
Output Test app:
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Deployment Demonstration</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    HTML{height:100%;}
    BODY{font-family:Helvetica,Arial;display:flex;display:-webkit-flex;align-items:center;justify-content:center;-webkit-align-items:center;-webkit-box-align:center;-webkit-justify-content:center;height:100%;}
    .box{background:#006e9c;color:white;text-align:center;border-radius:10px;display:inline-block;}
    H1{font-size:10em;line-height:1.5em;margin:0 0.5em;}
    H2{margin-top:0;}
  </style>
</head>
<body>
<div class="box"><h1>v1</h1><h2></h2></div>
</body>
</html>
```


Quick Test 2
----
```
oc login -u system:admin
oc project default
oadm registry --credentials=./openshift.local.config/master/openshift-registry.kubeconfig
oc login -u test
oc project test
oc new-app openshift/nodejs-010-centos7~https://github.com/openshift/nodejs-ex.git
oc status
curl 172.30.126.164:8080 # (RIGHT PLACE??)
```





---
# Basic Configuration
---


OpenShift "oc cluster up" Wrapper script
----

This script provides the following enhancements:
- cluster profiles
  - cluster management lifecycle
  - convenience methods for working with persistent volumes
  - convenience methods for adding common software to your cluster (This will be rewritten to be a plugin like mechanism)
```
git clone https://github.com/openshift-evangelists/oc-cluster-wrapper
echo 'PATH=$HOME/oc-cluster-wrapper:$PATH' >> $HOME/.bash_profile
echo 'export PATH' >> $HOME/.bash_profile
```


Change apps deployment domain name
----

Shutdown Cluster
```
oc cluster down
```
- Configuration location
    - --master-config=`/var/lib/origin/openshift.local.config/master/master-config.yaml`
    - --node-config=`/var/lib/origin/openshift.local.config/node-139.xxx.xxx.xxx/node-config.yaml`

Line 204 edit:
```
routingConfig:
  subdomain: 139.xxx.xxx.xxx.xip.io
 ```
to:
```
routingConfig:
  subdomain: demo.i3-cloud.com
```
Start cluster:
```
oc cluster up
```

---
# Screenshots
---

![login page](https://raw.githubusercontent.com/isnuryusuf/openshift-install/master/image1.png?raw=true)
![deploy1](https://github.com/isnuryusuf/openshift-install/blob/master/image2.png?raw=true)
![deploy2](https://github.com/isnuryusuf/openshift-install/blob/master/image3.png?raw=true)



