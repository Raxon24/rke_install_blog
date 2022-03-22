---
title: Install RKE2 + Rancher + Longhorn
author: Andy Clemenko, @clemenko, andy.clemenko@rancherfederal.com
---

# Simple RKE2, Longhorn, and Rancher Install

![logp](img/logo_long.jpg)

Throughout my career there has always been a disconnect between the documentation and the practical implementation. The Kubernetes (k8s) ecosystem is no stranger to this problem. This guide is a simple approach to installing Kubernetes and some REALLY useful tools. We will walk through installing all the following.

- [RKE2](https://docs.rke2.io) - Security focused Kubernetes
- [Rancher](https://www.suse.com/products/suse-rancher/) - Multi-Cluster Kubernetes Management
- [Longhorn](https://longhorn.io) - Unified storage layer

We will need a few tools for this guide. We will walk through how to install `helm` and `kubectl`.

---

> **Table of Contents**:
>
> * [Whoami](#whoami)
> * [Prerequisites](#prerequisites)
> * [Linux Servers](#linux-servers)
> * [RKE2 Install](#rke2-install)
>   * [RKE2 Server Install](#rke2-server-install)
>   * [RKE2 Agent Install](#rke2-agent-install)
> * [Rancher](#rancher)
>   * [Rancher Install](#rancher-install)
>   * [Rancher Gui](#rancher-gui)
> * [Longhorn](#longhorn)
>   * [Longhorn Install](#longhorn-install)
>   * [Longhorn Gui](#longhorn-gui)
> * [Automation](#automation)
> * [Conclusion](#conclusion)

---

## Whoami

Just a geek - Andy Clemenko - @clemenko - andy.clemenko@rancherfederal.com

## Prerequisites

The prerequisites are fairly simple. We need 3 linux servers with access to the internet. They can be bare metal, or in the cloud provider of your choice. I prefer [Digital Ocean](https://digitalocean.com). We need an `ssh` client to connect to the servers. And finally DNS to make things simple. Ideally we need a URL for the Rancher interface. For the purpose of the this guide let's use `rancher.dockr.life`. We will need to point that name to the first server of the cluster. Here are all the files: https://github.com/clemenko/rke_install_blog.

## Linux Servers

For the sake of this guide we are going to use [Ubuntu](https://ubuntu.com). Our goal is a simple deployment. The recommended size of each node is 4 Cores and 8GB of memory with at least 60GB of storage. One of the nice things about [Longhorn](https://longhorn.io) is that we do not need to attach additional storage. Here is an example list of servers. Please keep in mind that your server names can be anything. Just keep in mind which ones are the "server" and "agents".

```text
rancher1     142.93.189.52      8192   4   160   Ubuntu 21.10 x64
rancher2     68.183.150.214     8192   4   160   Ubuntu 21.10 x64
rancher3     167.71.188.101     8192   4   160   Ubuntu 21.10 x64
```

For Kubernetes we will need to "set" one of the nodes as the control plane. Rancher1 looks like a winner for this. First we need to `ssh` into all three nodes and make sure we have all the updates. For the record I am not a fan of software firewalls. Please feel free to reach to me to discuss. :D

```bash
# stop the software firewall
systemctl stop ufw
systemctl disable ufw

# get updates, install nfs, and apply
apt update
apt install nfs-common -y  
apt upgrade -y

# clean up
apt autoremove -y
```

Cool, lets move on to the RKE2.

## RKE2 Install

### RKE2 Server Install

Now that we have all the nodes up to date, let's focus on `rancher1`. While this might seem controversial, `curl | bash` does work nicely. There is one option we should set, `INSTALL_RKE2_CHANNEL`. If we look at the [Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-6-3/) we will need to use Kubernetes `1.21.x`. Here is how we do that with the tarball version. Please be patient with the start command. It is unpacking the components and it takes a minute. Here are the [rke2 docs](https://docs.rke2.io/install/methods/) for reference.

```bash
# On rancher1
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - 

# start and enable for restarts - 
systemctl enable rke2-server.service 
systemctl start rke2-server.service
```

Here is what is should look like:

```text
root@rancher1:~# curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 sh - 
[INFO]  finding release for channel v1.21
[INFO]  using v1.21.10+rke2r2 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.21.10+rke2r2/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.21.10+rke2r2/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
root@rancher1:~# systemctl enable rke2-server.service 
Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /usr/local/lib/systemd/system/rke2-server.service.
root@rancher1:~# systemctl start rke2-server.service
root@rancher1:~#
```

Let's validate everything worked as expected. Run a `systemctl status rke2-server` and make sure it is `active`. 

```text
root@rancher1:~# systemctl status rke2-server.service 
● rke2-server.service - Rancher Kubernetes Engine v2 (server)
     Loaded: loaded (/usr/local/lib/systemd/system/rke2-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-03-21 13:21:19 UTC; 2min 11s ago
```

Perfect! Now we can start talking Kubernetes. We need to install the `kubectl` cli on `rancher1`. FYI, We can install a newer `kubectl` without any issues.

```bash
# another curl...
curl -L# https://dl.k8s.io/release/v1.23.0/bin/linux/amd64/kubectl -o /usr/local/bin/kubectl

# make it executable
chmod 755 /usr/local/bin/kubectl

# add kubectl conf
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml 

# check node status
kubectl  get node
```

Hopefully everything looks good! Here is an example.

```
root@rancher1:~# kubectl  get node
NAME       STATUS   ROLES                       AGE     VERSION
rancher1   Ready    control-plane,etcd,master   7m56s   v1.21.10+rke2r2
```

For those that are not TOO familiar with k8s, the config file is what `kubectl` uses to authenticate to the api service. If you want to use a workstation, jump box, or any other machine you will want to copy `/etc/rancher/rke2/rke2.yaml`. You will want to modify the file to change the ip address. We will need one more file from `rancher1`, aka the server, the agent join token. Copy `/var/lib/rancher/rke2/server/node-token`, we will need it for the agent install.

### RKE2 Agent Install

The agent install is VERY similar to the server install. Except that we need an agent config file before starting. We will start with `rancher2`. We need to install the agent and setup the configuration file.

```bash
# we add INSTALL_RKE2_TYPE=agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 INSTALL_RKE2_TYPE=agent sh -  

# create config file
mkdir -p /etc/rancher/rke2/ 

# change the ip to reflect your rancher1 ip
echo "server: https://$RANCHER1_IP:9345" > /etc/rancher/rke2/config.yaml

# change the Token to the one from rancher1 /var/lib/rancher/rke2/server/node-token 
echo "token: $TOKEN" >> /etc/rancher/rke2/config.yaml

# enable and start
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

What should this look like:

```text
root@rancher2:~# curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.21 INSTALL_RKE2_TYPE=agent sh -  
[INFO]  finding release for channel v1.21
[INFO]  using v1.21.10+rke2r2 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.21.10+rke2r2/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.21.10+rke2r2/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
root@rancher2:~# mkdir -p /etc/rancher/rke2/ 
root@rancher2:~# echo "server: https://142.93.189.52:9345" > /etc/rancher/rke2/config.yaml
root@rancher2:~# echo "token: K10f46a788f0c2298b1cb3aa9ef169d7ce9cd8965d62c003d263ddd2363892fa613::server:2213cac45dcede3e470b1a5603e889aa" >> /etc/rancher/rke2/config.yaml
root@rancher2:~# cat /etc/rancher/rke2/config.yaml 
server: https://142.93.189.52:9345
token: K10f46a788f0c2298b1cb3aa9ef169d7ce9cd8965d62c003d263ddd2363892fa613::server:2213cac45dcede3e470b1a5603e889aa
root@rancher2:~# systemctl enable rke2-agent.service
systemctl start rke2-agent.service
Created symlink /etc/systemd/system/multi-user.target.wants/rke2-agent.service → /usr/local/lib/systemd/system/rke2-agent.service.
```

Now we can go back to `rancher1` and check to see if `rancher2` is playing nice.

```text
root@rancher1:~# kubectl  get node
NAME       STATUS   ROLES                       AGE   VERSION
rancher1   Ready    control-plane,etcd,master   26m   v1.21.10+rke2r2
rancher2   Ready    <none>                      86s   v1.21.10+rke2r2
```

Rinse and repeat. Run the same install commands on `rancher3`.

Checking `rancher` again.

```text
root@rancher1:~# kubectl  get node
NAME       STATUS   ROLES                       AGE     VERSION
rancher3   Ready    <none>                      38s     v1.21.10+rke2r2
rancher1   Ready    control-plane,etcd,master   29m     v1.21.10+rke2r2
rancher2   Ready    <none>                      4m42s   v1.21.10+rke2r2
```

Huzzah! RKE2 is fully installed. From here on out we will only need to talk to the kubernetes api. Meaning we will only need to remain ssh'ed into `rancher1`.

## Rancher

### Rancher Install

For Rancher we will need [Helm](https://helm.sh/). We are going to live on the edge! Here are the [Rancher docs](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/) for reference.

```bash
# on the server rancher1
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# add needed helm charts
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
```

Quick note about Rancher. Rancher needs jetstack/cert-manager to create the self signed TLS certificates. We need to install it with the Custom Resource Definition (CRD). Please pay attention to the `helm` install for Rancher. The URL will need to be changed to fit your FQDN. Also notice I am setting the `bootstrapPassword` and replicas. This allows us to skip a step later. :D

```bash
# again from rancher1
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml

# helm install jetstack
helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace

# helm install rancher
helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=rancher.dockr.life --set bootstrapPassword=bootStrapAllTheThings --set replicas=1
```

Here is what it should look like.

```text
root@rancher1:~# helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
"rancher-latest" has been added to your repositories

root@rancher1:~# helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

root@rancher1:~# kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created

root@rancher1:~# helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
Release "cert-manager" does not exist. Installing it now.
NAME: cert-manager
LAST DEPLOYED: Mon Mar 21 14:14:47 2022
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.7.1 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

root@rancher1:~# helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=rancher.dockr.life --set bootstrapPassword=bootStrapAllTheThings --set replicas=1
Release "rancher" does not exist. Installing it now.
NAME: rancher
LAST DEPLOYED: Mon Mar 21 14:15:08 2022
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://rancher.dockr.life to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

\```
echo https://rancher.dockr.life/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
\```

To get just the bootstrap password on its own, run:

\```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
\```

Happy Containering!
root@rancher1:~#
```

We can also run a `kubectl get pod -A` to see if everything it running. Keep in mind it may take a minute or so for all the pods to come up. GUI time...

### Rancher GUI

Assuming DNS is pointing to first server, `rancher1` for me, we should be able to get to the GUI. The good news is that be default rke2 installs with the `nginx` ingress controller. Keep in mind that the browser may show an error for the self signed certificate. In the case of Chrome you can type "thisisunsafe" and it will let you proceed.

![error](img/tls_error.jpg)

Once past that you should see the following screen asking about the password. Remember the helm install? `bootStrapAllTheThings` is the password.

![welcome](img/welcome.jpg)

We need to validate the Server URL and accept the terms and conditions. And we are in!

![dashboard](img/dashboard.jpg)

### Rancher Design

Let's take a second and talk about Ranchers Multi-cluster design. Bottom line, Rancher can operate in a Spoke and Hub model. Meaning one k8s cluster for Rancher and then "downstream" clusters for all the workloads. Personally I prefer the decoupled model where there is only one cluster.

NEEDS MOAR

## Longhorn

### Lognhorn Install

There are two methods for installing. Rancher has Chart built in.

![charts](img/charts.jpg)

Now for the good news, [Longhorn docs](https://longhorn.io/docs/1.2.4/deploy/install/) show two easy install methods. Helm and `kubectl`. Let's stick with Helm for this guide.

```bash
# get charts
helm repo add longhorn https://charts.longhorn.io

# update
helm repo update

# install
helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```

Fairly easy right?

### Longhorn GUI

One of the benefits of Rancher is it's ability to adjust to whats installed. Meaning the Rancher GUI will see Longhorn is installed and provide a link. Click it.

![longhorn installed](img/longhorn_installed.jpg)

This brings up the Longhorn GUI.

![longhorn](img/longhorn.jpg)

One of the other benefits of this integration is that rke2 also knows it is installed. Run `kubectl get sc` to show the storage classes.

```text
root@rancher1:~# kubectl  get sc
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   3m58s
```

Now we have a default storage class for the cluster. This allows for the automatic creation of Physical Volumes (PVs) based on a Physical Volume Claim (PVC). The best part is that "it just works" using the existing, unused storage, on the three nodes. Take a look around in the gui. Notice the Volumes on the Nodes. For fun, here is a demo flask app that uses a PVC for Redis. `kubectl apply -f https://raw.githubusercontent.com/clemenko/k8s_yaml/master/flask_simple_nginx.yml`

## Automation

Yes we can automate all the things. Here is the repo I use automating the complete stack https://github.com/clemenko/rke2. This repo is for entertainment purposes only. There I use tools like pdsh to run parallel ssh into the nodes to complete a few tasks. Ansible would be a good choice for this. But I am old and like bash. Sorry the script is a beast. I need to clean it up.

## Conclusion

As we can see setting up RKE2, Rancher and Longhorn is not that complicated. One of the added benefits of using the Suse / Rancher stack is that all the pieces are modular. Use only what you need, when you need it.

![success](img/success.jpg)
