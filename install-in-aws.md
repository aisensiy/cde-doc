# 在 aws 部署集群

目录 `contrib/ansible/iaas/aws` 为对 aws 环境做部署的脚本，本地环境的安装以及 PaaS 的部署参见[安装 cde](./install-cde-raw.md)，这里只包含基础设施的搭建。

1. 安装 aws 客户端及其依赖
	
	```
    brew install awscli
    brew install jq
	```
2. 配置 aws 的 key 以及机器默认的区域

	```
	aws configure
	```
	
3. 配置aws机器以及类型

	```
	$CDE/contrib/ansible/iaas/aws/
	vim customization.json
	```
	更改相应的变量配置
	
4. 初始化aws机器

	```
    ./provision.sh cde
	```
	
5. 准备aws hosts 文件
   在运行过 `./provision.sh cde` 之后会在当前目录下生成文件名为 `infrastructure.json` 的机器清单以及 `hosts` 文件
   
6. 准备相应的extravars.json 文件
   根据生成的infrastructure.json 来准备相应的extravars.json 文件,具体参照[安装 cde](./install-cde-raw.md)

7. 获取依赖的ansible role
	
	```
	ansible-galaxy install -r playbooks/roles.yml -p playbooks/roles --force
	```
	
8. 基础环境部署

	```
	ansible-playbook --extra-vars="@extravars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts playbooks/master.yml
	ansible-playbook --extra-vars="@extravars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts playbooks/slave.yml
	ansible-playbook --extra-vars="@extravars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts playbooks/elasticsearch.yml
	ansible-playbook --extra-vars="@extravars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts playbooks/ceph.yml
	ansible-playbook --extra-vars="@extravars.json" --connection=ssh --timeout=30 --limit='all' --inventory-file=hosts playbooks/rbd-driver.yml
	```
	
9. 准备一个带有桌面环境的机器以访问集群中的 `mesos`。

	在aws开一台带有桌面操作系统的机器
	
	```
	sudo vim /etc/ssh/sshd_config # edit line "PasswordAuthentication" to yes
	sudo /etc/init.d/ssh restart
	sudo apt-get update
	sudo apt-get install ubuntu-desktop
	sudo apt-get install vnc4server
	vncserver

	vncserver -kill :1
	
	vim awsgui/.vnc/xstartup
   ``` 
   
   Then hit the Insert key, scroll around the text file with the keyboard arrows, and delete the pound (#) sign from the beginning of the two lines under the line that says "Uncomment the following two lines for normal desktop." And on the second line add "sh" so the line reads
   
	```
	exec sh /etc/X11/xinit/xinitrc.
	vncserver
	```

	config the security group make the desktop can access the masters and slaves.