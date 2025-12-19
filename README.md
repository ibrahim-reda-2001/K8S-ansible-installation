## 1-install ansible ##

``` 

$ sudo apt update
$ sudo apt install software-properties-common
$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```
## 2-Generate the private key of machine ##

put this private key  path in ansible.cfg 
```
private_key_file = ./key.pem
# if you use AWS change permission first to 400
chmod 400 key.pem 
```
## 3- Edit hosts file in you machine ##
edit it with your IPs
```
vim /etc/hosts
127.0.0.1 localhost
18.205.246.4  master1
54.167.62.72  worker1
54.152.204.6 worker2
3.88.104.5  worker3
```
## 4- To verify you working right ##

```
ubuntu@ip-172-31-31-231:~/K8S-ansible-installation$ ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@master:
  |  |--master1
  |--@workers:
  |  |--worker1
  |  |--worker2
  |  |--worker3
```

And this to check reachability 
```
ubuntu@ip-172-31-31-231:~/K8S-ansible-installation$ ansible all -m ping
worker3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
master1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
ubuntu@ip-172-31-31-231:~/K8S-ansible-installation$ a
```

## 5- To run a playbook ##
```
ansible-playbook playbook.yml

```