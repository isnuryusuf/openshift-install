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

Configure the Docker daemon with an insecure registry parameter of 172.30.0.0/16
In RHEL and Fedora, edit the /etc/containers/registries.conf file and add the following lines:

```
[registries.insecure]
registries = ['172.30.0.0/16']

 systemctl daemon-reload
 systemctl restart docker
```

Determine the Docker bridge network container subnet:
```
docker network inspect -f "{{range .IPAM.Config }}{{ .Subnet }}{{end}}" bridge
```
You should get a subnet like: 172.17.0.0/16

Create a new firewalld zone for the subnet and grant it access to the API and DNS ports:
```
firewall-cmd --permanent --new-zone dockerc
firewall-cmd --permanent --zone dockerc --add-source 172.17.0.0/16
firewall-cmd --permanent --zone dockerc --add-port 8443/tcp
firewall-cmd --permanent --zone dockerc --add-port 53/udp
firewall-cmd --permanent --zone dockerc --add-port 8053/udp
firewall-cmd --reload
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
or
oc cluster up --metrics
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
openshift start --write-config=/openshift.local.config
openshift start --master-config=/openshift.local.config/master/master-config.yaml --node-config=/openshift.local.config/node-openshit.i3-cloud.com/node-config.yaml
#openshift start
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

Login as system:admin
----
```
oc login -u system:admin -n default
Logged into "https://139.59.243.79:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-system
    myproject
    openshift
    openshift-infra
    test
    test-project1
    test-project2
    test2

Using project "default".
```

```
oc status
In project default on server https://139.59.243.79:8443

svc/docker-registry - 172.30.248.225:5000
  dc/docker-registry deploys docker.io/openshift/origin-docker-registry:v1.4.1 
    deployment #1 deployed 35 minutes ago - 1 pod

svc/kubernetes - 172.30.0.1 ports 443, 53->8053, 53->8053

svc/router - 172.30.4.117 ports 80, 443, 1936
  dc/router deploys docker.io/openshift/origin-haproxy-router:v1.4.1 
    deployment #1 deployed 35 minutes ago - 1 pod

View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'
```



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
source .bash_profile
```


Change apps deployment domain name and persistent cluster data
----

Shutdown Cluster
```
oc cluster down
```
- Configuration location
    - --master-config=`/var/lib/origin/openshift.local.config/master/master-config.yaml`
    - --node-config=`/var/lib/origin/openshift.local.config/node-139.xxx.xxx.xxx/node-config.yaml`
    - --host-data-dir=`/root/cluster1` (you free to specify)

Edit `/var/lib/origin/openshift.local.config/master/master-config.yaml` Line 204 and change:
```
routingConfig:
  subdomain: 139.xxx.xxx.xxx.xip.io
 ```
to:
```
routingConfig:
  subdomain: demo.i3-cloud.com
```

Start cluster with specific --host-data-dir:
```
oc cluster up --host-data-dir=/root/cluster1 --public-hostname=demo.i3-cloud.com --routing-suffix=demo.i3-cloud.com --use-existing-config
oc cluster status
The OpenShift cluster was started 2 minutes ago

Web console URL: https://139.59.243.79:8443

Config is at host directory /var/lib/origin/openshift.local.config
Volumes are at host directory /var/lib/origin/openshift.local.volumes
Data is at host directory /root/cluster1
```

Expose Application to public
```
oc expose service deployment-example
route "deployment-example" exposed
```

```
oc get svc
NAME                 CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
deployment-example   172.30.198.162   <none>        8080/TCP   8m

NAME                 HOST/PORT                                            PATH      SERVICES             PORT       TERMINATION
deployment-example   deployment-example-test-project2.demo.i3-cloud.com             deployment-example   8080-tcp   

```

Change PasswordIdentityProvider from anypassword to httpasswd 
----
By default Openshift running anypassword for PasswordIdentityProvider, to secure openshift login we must change to non-anypassowrd identityprovider, like httpasswd

First install dependency and create user database
```
yum install httpd-tools -y

htpasswd -c /var/lib/origin/openshift.local.config/master/users.htpasswd yusuf
New password: 
Re-type new password: 
Adding password for user yusuf

htpasswd -m /var/lib/origin/openshift.local.config/master/users.htpasswd admin
New password: 
Re-type new password: 
Adding password for user admin

```

Edit file `/var/lib/origin/openshift.local.config/master/master-config.yaml` and find:
```
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: anypassword
    provider:
      apiVersion: v1
      kind: AllowAllPasswordIdentityProvider
```

Change to:
```
  identityProviders:
  - name: my_htpasswd_provider 
    challenge: true 
    login: true 
    provider:
      apiVersion: v1
      kind: HTPasswdPasswordIdentityProvider
      file: /root/cluster1/users.htpasswd
```

After change we must restart openshift
```
oc cluster down
oc cluster up --host-data-dir=/root/cluster1 --public-hostname=demo.i3-cloud.com --routing-suffix=demo.i3-cloud.com --use-existing-config
```

Adding user yusuf as cluster-admin
```
yum -y install origin
oc login -u system:admin -n default
oadm policy add-cluster-role-to-user cluster-admin yusuf
```


---
# Screenshots
---

![login page](https://github.com/isnuryusuf/openshift-install/blob/master/images/image1.png?raw=true)
![deploy1](https://github.com/isnuryusuf/openshift-install/blob/master/images/image2.png?raw=true)
![deploy2](https://github.com/isnuryusuf/openshift-install/blob/master/images/image3.png?raw=true)



