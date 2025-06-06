---
author: Maulana malik jabbar budianto
title: Setup cluster kubernetes 3 node with kubeadm using ansible
showToc: true
tocopen: false
---
# Setup cluster kubernetes 3 node with kubeadm using ansible

## Setup ssh

Setup ssh for all node for ssh passwordless to user

execute from master-1

```
ssh-keygen -t rsa
```

copy /.ssh/id.rsa.pub to all node

```
cat ~/.ssh/id_rsa.pub
```

![](/images/Screenshot_16.png)

insert id\_rsa.pub master to all worker node

```
nano ~/.ssh/authorized_keys
```

![](/images/Screenshot_17.png)  
after insert [idrsa.pub](http://idrsa.pub) , test ssh to all worker node to user root , make sure the ssh passwordless

## Setup ansible

Install ansible on master node

```
sudo apt install software-properties-common -y
```

```
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

```
sudo apt install ansible -y
```

makesure ansible has been installed

```
ansible --version
```

![](/images/Screenshot_18.png)