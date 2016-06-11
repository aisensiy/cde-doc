

## 本地环境准备

### 安装自动化部署工具 ansible

```
brew install ansible
```

其他操作系统的安装详见[官方文档](http://docs.ansible.com/ansible/intro_installation.html)


### 安装 docker

```
brew install docker-machine
brew install virtualbox
docker-machine create --driver virtualbox --virtualbox-disk-size=100000 --virtualbox-memory=400 --virtualbox-host-dns-resolver --virtualbox-dns-proxy  --engine-insecure-registry=10.0.0.0/8 --engine-insecure-registry=172.16.0.0/12 --engine-insecure-registry=192.168.0.0/16 --engine-insecure-registry=registry.{{you domain here}}:80 registry
```

### 安装 go

```
brew install go --with-cc-common
echo 'export GOPATH=/workplace/golang' >> ~/.zshrc
echo 'PATH=$PATH:$GOPATH/bin' >> ~/.zshrc
source ~/.zshrc
mkdir -p $GOPATH/src/github.com
```

## PaaS 集群环境准备

### 准备 `ansible inventory`

inventory 是指部署时的详单，修改 `playbooks/hosts` 在不同的角色下添加相应的机器 ip，其中 

* `[all]` 下为集群中所有的节点
* `[zookeeper]` `[marathon]` `[consul-server]` `[mesos-master]` `[mons]` 都设置为 3 台 mater 的 ip
* `[mesos-slave]` `[osds]` `[rbd]` 都设置为 3 台 slave 的 ip

### 准备集群相关配置

依照实际的集群 ip 与所在的网络修改 `playbooks/extra_vars.json` 

* `ansible_ssh_user` 为所有机器的登录账号，需要使用 root
* `ansible_ssh_pass` 为所有机器的密码，这里所有的密码都是统一的，如果各个机器的密码不统一，需要在 `inventory` 中单独设置（参见 inventory  中的 [elastic-master]），并在 `playbooks/extra_vars.json` 删除 `ansible_ssh_pass` 字段
* `username` 与 `password` 为内部代理需要的账号密码
* `no_proxy` 为不需要使用代理的 ip 与域名，需要包含集群中的所有节点的 ip
* `registry_mirror` 为中心 registry 的 ip
* `zookeeper_hosts` 为 zookeeper 的三个机器，将三个 ip 替换为 inventory 中的 `[zookeeper]` 的 ip
* `consul_servers` 为 inventory 中 `[consul-server]` 的 ip
* `devices` 为集群机器中未使用的硬盘
* `public_network` 为集群所在的网络
   
其他参数保持不变即可
   

### 执行自动化脚本

`cd playbooks` 依次执行如下自动化脚本

```
ansible-playbook --extra-vars="@extra_vars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts -s -vvvv networks.yml
ansible-playbook --extra-vars="@extra_vars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts -s -vvvv master.yml
ansible-playbook --extra-vars="@extra_vars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts -s -vvvv slave.yml
ansible-playbook --extra-vars="@extra_vars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts -s -vvvv ceph.yml
ansible-playbook --extra-vars="@extra_vars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts -s -vvvv rbd-driver.yml
```

在浏览器打开 `http://{ONE_MARATHON_NODE_IP}:8080` 查看 `marathon` 是否安装成，在 `http://{ONE_MARATHON_NODE_IP}:5050` 查看 mesos 是否安装成功。 

## PaaS 部署

```
cd $GOPATH/src/github.com/sjkyspa/stacks
```

设置 `CDECTL_TUNNEL` 到任意一台 `[marathon]` 的机器的 IP

```
export CDECTL_TUNNEL=ONE_MASTER_MACHINE_IP
```

设置远程机器的用户

```
export SSH_USERNAME={username}
```

添加 `public key`

```
eval `ssh-agent -s`
ssh-add ~/.ssh/id_rsa.pub
```

安装 PaaS 的组件

```
make deploy
```

在浏览器打开 `http://ONE_MARATHON_NODE_IP:8080` 查看组件的部署情况