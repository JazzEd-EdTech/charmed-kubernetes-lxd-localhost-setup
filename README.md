# charmed-kubernetes-lxd-localhost-setup
## Pre-requisites
### Create Symbolic Link to RAID storage (only if required)
This step is necessary only if the LXC storage pool needs to be mounted on the path of another Filesystem. 

For example, to change the lxc `default` storage pool location from the root directory on a physical storage mounted at `/..` to the home directory on a RAID storage mounted at `/home/..`, create a symbolic link of the root directory mount path pointing to home directory mount path as below:
```
$ sudo ln -s /home/new/storage/path /default/storage/path
```
Example:
```
$ sudo ln -s /home/k8s-storage-pools /var/snap/lxd/common/lxd/storage-pools 
```
> Ensure to keep a backup of the default storage pool before creating the symbolic link and also to assign necessary permissions to the new storage location similar to that of default location to keep the data secure. Use `chmod --reference=dir1 dir2`.
 
### Install Conjure-up

```
$ sudo apt update
$ sudo apt upgrade
$ sudo snap install conjure-up --classic
```
### Install and initialise LXD
> Canonical recommends snap installation of lxd. Run `dpkg -s lxd |  grep Status` to check for any apt lxd installation. If it is present, then remove it by running `sudo apt purge liblxc1 lxcfs lxd lxd-client`.
```
$ sudo snap install lxd
```
Create `default` lxc storage pool:
```
$ lxc storage create default dir
```
Add the new lxc storage pool to `default` lxc profile:
```
$ lxc profile device add default root disk path=/ pool=default
```
Initialise lxd by running:
```
$ lxd init
```
Use the following setting:
```
Would you like to use LXD clustering? (yes/no) [default=no]:

Do you want to configure a new storage pool? (yes/no) [default=yes]: no

Would you like to connect to a MAAS server? (yes/no) [default=no]:

Would you like to create a new local network bridge? (yes/no) [default=yes]:

What should the new bridge be called? [default=lxdbr0]:

What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:

What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: none

Would you like LXD to be available over the network? (yes/no) [default=no]:

Would you like stale cached images to be updated automatically? (yes/no) [default=yes]

Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```
In order to access the LXD service, the `$USER` should be a part of the `lxd` group. Verify it by running:
```
$ id
```
Example output:
```
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),129(lxd)
```

If the user does not belong to the `lxd` group then add it by running the following:
```
$ sudo usermod -a -G lxd $USER
$ newgrp lxd
```
### Verify LXD storage
Ensure that at least one storage pool is created for the `default` profile by running:
```
$ lxc storage list
```
Example output:
```
+---------+-------------+--------+------------------------------------------------+---------+
|  NAME   | DESCRIPTION | DRIVER |                     SOURCE                     | USED BY |
+---------+-------------+--------+------------------------------------------------+---------+
| default |             | dir    | /var/snap/lxd/common/lxd/storage-pools/default | 1       |
+---------+-------------+--------+------------------------------------------------+---------+
```
```
$ lxc storage show default
```
Example output:
```
config:
  source: /var/snap/lxd/common/lxd/storage-pools/default
description: ""
name: default
driver: dir
used_by:
- /1.0/profiles/default
status: Created
locations:
- none
```
### Verify LXD Network

For localhost deployments, LXD must have a network bridge defined. This is already setup during the lxd initialization step. Verify by running:
```
$ lxc network show lxdbr0
```
Example output:
```
config:
  ipv4.address: 10.99.16.1/24
  ipv4.nat: "true"
  ipv6.address: none
  ipv6.nat: "false"
description: ""
name: lxdbr0
type: bridge
used_by: []
managed: true
status: Created
locations:
- none
```
If any of the configs are set differently except for `ipv4.address`, then update it by running:
```
lxc network set lxdbr0 <config> <value>
```
Example:
```
$ lxc network set lxdbr0 ipv6.nat false
```
Ensure that the lxd `default` profile is set to use `lxdbr0` as its bridged interface.
```
$ lxc profile show default
```
Example output:
```
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
used_by: []
```
To update any config, run `lxc profile edit default`.
### Test LXD setup
Create an ubuntu container and execute `ping` command inside it to test network connectivity:
```
$ lxc launch ubuntu:16.04 u1
$ lxc exec u1 ping deakin.edu.au
```
If everything works, remove the container:
```
$ lxc stop u1
$ lxc delete u1
```
## Deploy Kubernetes cluster
### Conjure charmed kubernetes
Summon the spell to setup Charmed Kubernetes by running:
```
$ conjure-up
```
From the command Line UI, select the `The Canonical Distribution of Kubernetes` spell and continue installation on `localhost` with default settings.
> The spell will deploy charmed kubernetes using Juju application modelling tool. It allows you to deploy, configure, scale and operate your software on public and private clouds. https://juju.is/docs
## Post-deployment
### Forward traffic to kubeapi load balancer
When the cluster is running behind a load balancer or a server, request traffic has to be forwarded to lxd container running the Kubernetes API server. To do that, add an LXD proxy device which can forward traffic to the desired container.
```
lxc config device add <container_name> <device_name> proxy listen=tcp:<server_ip>:<port> connect=tcp:<container_ip>:<port>
```
> Identify the kubeapi container name by running `juju status` and the IP of server by running `juju status | grep kubeapi-load-balancer`

Example:
```
$ lxc config device add juju-8a5ef8-7 kubeapi-port proxy listen=tcp:192.168.122.168:6443 connect=tcp:10.99.16.56:443
```
Verify the setting by running:
```
lxc config device show <container_name>
```
### Regenerate Subject Alternate Name (SAN) certificate
To provide the users with access to the cluster, the Kubernetes API server needs to authenticate the certificate presented by the user request. If the cluster is running behind a load balancer or server with an external IP, the certificate needs to hold information about the external domain as well.

Use juju's `extra_sans` configuration to regenerate certificate by including the additional domain. https://github.com/charmed-kubernetes/bundle/wiki/Certificate-regeneration-via-extra_sans-options
```
juju config kubeapi-load-balancer extra_sans="master.mydomain.com lb.mydomain.com"
```
Example:
```
$ juju config kubeapi-load-balancer extra_sans="192.168.122.32"
```
### Setup kubectl authorization for clients
To provide kubectl access to the users, add `RBAC` and `Node` as the authorization mode.
```
$ juju config kubernetes-master authorization-mode="RBAC,Node"
```
#### Manage users
To add a user, edit the `/root/cdk/basic_auth.csv` file in master. Note that the format for this file is `password,user,uid,"group1,group2,group3"`.
```
$ juju ssh kubernetes-master/0
$ sudo nano /root/cdk/basic_auth.csv
```
Restart the master after updating the file:
```
$ juju run-action kubernetes-master/0 restart
```
#### Create user roles
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#kubectl-create-role
```
kubectl create role <role-name> [options]
```
Example:
```
$ kubectl create role pod-deployment-reader --verb=get --verb=list --verb=watch --resource=pods --resource=deployment
```
#### Create user rolebinding
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#kubectl-create-rolebinding
```
kubectl create rolebinding <role-binding-name> [options]
```
Example:
```
$ kubectl create rolebinding pod-reader-deployment-binding --role=pod-deployment-reader --user=bob --namespace=development
```
## Scale deployment
Information on cluster scaling is available in the official documentation: https://ubuntu.com/kubernetes/docs/scaling
## Tear down deployment
Run the below juju command to list the controller and the model associated with it:
```
$ juju controllers
``` 
Run below juju commands to detroy the model and controller:
```
juju destroy-model <controller>:<model>
juju kill-controller <controller>
```
Once the controller and model are safely removed, detach the storage pool from the lxc profile and delete it by running the following:
```
$ printf 'config: {}\ndevices: {}' | lxc profile edit default
$ lxc storage delete default
```
Remove redundant lxc profiles:
```
juju profile delete <profile>
```