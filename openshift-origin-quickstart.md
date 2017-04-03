---
# Pre-Setup
---

Note1: This is setup for any of the quickstarts

- Setting hostname at `/etc/hosts` file, for example:
```
ip-address  domain-name.tld
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

Note1: This section is still being worked on.
```
yum install centos-release-openshift-origin
yum install origin-clients
oc cluster up
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
oc status
curl 172.30.126.164:8080 # (example v1) (Use URL that it gives you for svc/deployment-example)
oc tag deployment-example:v2 deployment-example:latest
curl 172.30.126.164:8080 # (example2 v2)
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


### Screenshots
![login page](https://raw.githubusercontent.com/isnuryusuf/openshift-install/master/image1.png?raw=true)

