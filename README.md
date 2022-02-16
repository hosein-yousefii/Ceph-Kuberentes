# Ceph-Kuberentes

[![GitHub license](https://img.shields.io/github/license/hosein-yousefii/Ceph-Kuberentes)](https://github.com/hosein-yousefii/Ceph-Kuberentes/blob/master/LICENSE)
![LinkedIn](https://shields.io/badge/style-hoseinyousefi-black?logo=linkedin&label=LinkedIn&link=https://www.linkedin.com/in/hoseinyousefi)

Implement Ceph cluster and use it as a StorageClass on kubernetes to claim persistent volume.

## What is Ceph?
Ceph is an open source software-defined storage solution designed to address the block, file and object storage needs of modern enterprises. Its highly scalable architecture sees it being adopted as the new norm for high-growth block storage, object stores, and data lakes. Ceph provides reliable and scalable storage while keeping CAPEX and OPEX costs in line with underlying commodity hardware prices. 

### What is Ceph RBD?
A RADOS Block Device (RBD) is software that facilitates the storage of block-based data in the open source Ceph distributed storage system.

The RBD software breaks up block-based application data into small chunks. The data chunks are subsequently stored as objects within the Reliable Autonomic Distributed Object Store (RADOS). RBD orchestrates the storage of the objects in virtual block devices throughout the Ceph storage cluster. These block devices are the virtual equivalent of physical disk drives.


## What is StorageClass?
A StorageClass provides a way for administrators to describe the "classes" of storage they offer. 


### What is PVC?
A PVC is the request to provision persistent storage with a specific type and configuration. To specify the persistent storage flavor that you want, you use Kubernetes storage classes.



# Get started:
Implementing Ceph and integrate it with kubernetes have different steps:
It's better to install 3 Ubuntu20-04 nodes.

1) we need to deploy Ceph cluster, to do this we need at least 3 nodes:

192.168.100.201  ceph1

192.168.100.202  ceph2

192.168.100.203  ceph3


Also 1 raw disk on each node should be available.


2) add these 3 nodes to each /etc/hosts file:
```
echo """
127.0.0.1 localhost

# Ceph nodes
192.168.100.201  ceph1
192.168.100.202  ceph2
192.168.100.203  ceph3
""" > /etc/hosts
```

3) Set each node's Hostname:
```
hostnamectl set-hostname ceph1
exec bash
```

4) Install some dependencies (on ceph1):
```
apt -y install software-properties-common git curl vim bash-completion ansible
```

5) Install Cephadm (on ceph1):
```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
mv cephadm  /usr/local/bin/
```

6) Generate key and copy to your nodes (on ceph1):
```
ssh-keygen
ssh-copy-id root@ceph1
ssh-copy-id root@ceph2
ssh-copy-id root@ceph3
```

7) Clone this repository (on ceph1):
```
mkdir ceph
cd ceph
git clone https://github.com/hosein-yousefii/Ceph-Kuberentes.git
cd Ceph-Kuberentes
```

8) Change ssh configuration (on ceph1):
```
tee -a ~/.ssh/config<<EOF
Host *
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 300
EOF
```

9) Create ansible inventory (on ceph1):
```
echo """
[ceph_nodes]
ceph1
ceph2
ceph3
""" > hosts-inventory.yaml
```

10) Run the playbook to install packages (on ceph1):
```
ansible-playbook -i hosts-inventory.yaml prepare-ceph-nodes.yml --user root
```

11) Reboot all nodes.

12) On ceph1 configure Ceph cluster by adding nodes and monitors:
```
cephadm bootstrap --mon-ip 192.168.100.201 --initial-dashboard-user admin --initial-dashboard-password qazwsx

ceph orch host add ceph2 192.168.100.202
ceph orch host add ceph3 192.168.100.203
```

  Add label for monitors (on ceph1):
```
ceph orch host label add ceph1 mon
ceph orch host label add ceph2 mon
ceph orch host label add ceph3 mon
```

 Apply configs (on ceph1):
 ```
ceph orch apply mon ceph2
ceph orch apply mon ceph3
```

List hosts (on ceph1):
```
ceph orch host ls
```

Add labels for osd (on ceph1):
```
ceph orch host label add  ceph1 osd
ceph orch host label add  ceph2 osd
ceph orch host label add  ceph3 osd
```

After some minute you should see your raw disks by executing this command (on ceph1):
```
ceph orch device ls
```
Time to add disks to osd (on ceph1):
```
ceph orch daemon add osd ceph1:/dev/sdb
ceph orch daemon add osd ceph2:/dev/sdb
ceph orch daemon add osd ceph3:/dev/sdb
```

13) Wait until all docker container up & running on all nodes:
```
docker ps
```

14) Then you would be able to find your disks on Ceph dashboard. https://ceph1:8443

15) Create a pool for kubernetes (on ceph1):
```
ceph osd pool create k8s
rbd pool init k8s
```

16) Create a user to access Ceph cluster from kubernetes (on ceph1):
```
ceph auth get-or-create client.kube mon 'profile rbd' osd 'profile rbd pool=k8s' mgr 'profile rbd pool=k8s'
```

17) Find your monitors IP addresses (on ceph1):
```
ceph mon dump
```

18) Find your Ceph cluster ID (on ceph1):
```
ceph -s
```

19) Login to your kubernetes cluster or where you have access to kubectl then Copy or clone *.yaml files to it.

20) Change csi-config-map.yaml based on your Ceph cluster monitor and ID.

21) Change csi-rbd-secret.yaml based on your user and key which created later on ceph1 for kubernetes cluster

22) Change csi-rbd-sc.yaml based on your ceph cluster ID and Pool name(k8s)

23) Deploy manifests:
```
kubectl apply -f csi-config-map.yaml
kubectl apply -f csi-kms-config-map.yaml
kubectl apply -f ceph-config-map.yaml
kubectl apply -f csi-rbd-secret.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
kubectl apply -f csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin.yaml
kubectl apply -f csi-rbd-sc.yaml
```

24) Check if csi pods are running:
```
kubectl get po
```

25) Create a PVC to check if the storageclass is running correctly:
```
kubectl apply -f raw-block-pvc.yaml
kubectl apply -f fs-pvc.yaml
```


That's all you had to do with. enjoy!



# How to contribute:

You are able to automate all these proccess.

Copyright 2022 Hosein Yousefi <yousefi.hosein.o@gmail.com>
