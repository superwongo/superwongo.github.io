---
title: Docker部署Mongodb副本集集群
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

# Docker部署Mongodb副本集集群

## 1. Mongodb简介

	目前常用的数据库主要分为两种：关系型数据库、非关系型数据库。其中关系型数据库常见的包括：Oracle、Mysql、PostgreSQL、SqlSever等，其主要特点就是大多数都遵循SQL（结构化查询语言，Structured Query Language）标准，针对结构化的数据支持比较好，十分注重数据操作的事务性、一致性。而非关系型数据库（NoSQL）特点则是非关系型数据支持比较好，多用于追求速度、高扩展性、多应用场景的项目。
	
	对于非关系型数据库主要包括以下四类：

- 键值对存储（key-value）方式：以Redis为代表；

- 列存储：以Hbase为代表；

- 文档数据库存储：以Mongodb为代表；

- 图形数据库存储：以InfoGrid为代表。

  

  Mongod是一个面向文档存储的数据库，其存储文档格式类似于JSON对象，其也可以实现高可用的分布式数据存储系统。同时，Mongodb支持多种编程语言：RUBY，PYTHON，JAVA，C++，PHP，C#等。

## 2. 使用背景

	对于日志等大数据量且非关系型的数据，使用Mongodb进行存储较为合适。为了保证存储数据的安全性、高可用性，目前越来越多的系统开始采用分布式的方式部署。因此，此篇文章对Mongodb分布式方式进行简单的探讨。
	
	Mongodb针对于分布式部署主要有三种方式：

- 主从复制`Master-Slaver`：数据库集群、服务器中均需明确指定主节点，从节点可以对主节点的数据进行同步，如果主节点宕机，可以人工的使用从节点代替主节点，目前已不推荐使用。
- 副本集`Replica Set`：存在一组存在相同数据的Mongodb实例，主mongodb接受所有的写操作，其他实例接受主实例的操作以保持数据的同步，同时主实例宕机后，其他实例可以自主选举新的主实例，这样就实现了数据库的安全性、高可用性。但是相对来说缺点就是数据的冗余。
- 分片`Sharding`：为了更好的处理大数据，分片方式是将数据进行分开存储，不同的数据库保存不同的数据，所有的数据总和即为整个数据集。

	对于存储数据量较大的大型项目中，使用分片的方式可以很好的减少服务器的压力，将数据进行水平拆分，但是由于配置较为复杂，需要`分片服务器（Shard Server）`、`配置服务器（Config Server）`、`路由服务器（Route Server）`等多类服务器进行配合使用。所以针对较小型的项目，使用副本集方式即可实现数据的安全性、高可用性，此处主要对副本集进行使用。

## 3. 副本集搭建准备

	副本集主要包括三种节点：主节点、从节点、仲裁节点。其中主节点负责数据的读、写；从节点默认情况下不负责数据的读、写，但是可以通过设置的方式，设置从节点执行外部数据的读取；仲裁节点不负责数据的复制，只负责投票，所以不容易出现故障，这样可以增加有效投票，使得整个副本集投数票尽可能多的保证参与选举的节点数据大于副本集总节点数一半的数量，以保证主节点的正常选举。
	
	官方推荐MongoDB副本节点最少为3台， 建议副本集成员为奇数，最多12个副本节点，最多7个节点参与选举。限制副本节点的数量，主要是因为一个集群中过多的副本节点，增加了复制的成本,反而拖累了集群的整体性能。 太多的副本节点参与选举，也会增加选举的时间。而官方建议奇数的节点,是为了避免脑裂 的发生。

### 3.1. 创建测试虚拟机

	通过vagrant的方式创建三台虚拟机进行副本集的搭建演示。

| 虚拟机名称 | IP            | mongodb类型 |
| ---------- | ------------- | ----------- |
| mongodb1   | 192.168.33.10 | `Primary`   |
| mongodb2   | 192.168.33.11 | `Secondary` |
| mongodb3   | 192.168.33.12 | `Secondary` |

	通过`vagrant init`命令初始化虚拟机配置文件，以`mongodb1`虚拟机为例，其他虚拟机类似

```bash
Administrator@SUPERWONG D:\Vagrant\mongodb1
# vagrant init
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

	修改虚拟机配置文件，此处已将与虚拟机配置文件所在目录同级的`data`目录映射到虚拟机中的`/vagrant_data`目录。

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"

  # 端口映射
  # config.vm.network "forwarded_port", guest: 80, host: 8080
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # 设置私有网络固定IP
  config.vm.network "private_network", ip: "192.168.33.10"
  # 设置公有网络
  config.vm.network "public_network"

  # 共享目录
  config.vm.synced_folder "../data", "/vagrant_data"

  # 虚拟相关配置
  config.vm.provider "virtualbox" do |vb|
    # 不显示虚拟机图形界面
    vb.gui = false
    # 虚拟机内存
    vb.memory = "1024"
    # 虚拟机名称
    vb.name = "mongodb1"
  end

  # 虚拟机启动时执行命令
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
```

	通过`vagrant up`命令启动虚拟机。通过`vagrant ssh`命令登录虚拟机内部修改`/etc/ssh/sshd_config`配置文件，设置为允许密码登录，并通过`service sshd restart`重启ssh服务。设置完成之后既可以通过SSH类工具进行密码登录。

```latex
PasswordAuthentication yes
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901224912706.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

	统一安装`docker`、`docker-compose`、`ansible`、`sshpass`。通过`ansible`进行多服务器的环境部署，通过`docker-compose`进行docker容器的创建，`sshpass`是`ansible`进行ssh连接时的依赖包。其中`ansible`、`sshpass`只在`mongodb1`主机安装即可

```bash
[root@localhost ~]# yum install -y docker
[root@localhost ~]# service docker start
```

```bash
[root@localhost ~]# yum install -y epel-release
[root@localhost ~]# yum install -y python-pip
[root@localhost ~]# pip install --upgrade pip -i https://pypi.doubanio.com/simple
[root@localhost ~]# pip install docker-compose -i https://pypi.doubanio.com/simple
[root@localhost ~]# pip install ansible -i https://pypi.doubanio.com/simple
[root@localhost ~]# yum install -y sshpass
```

	拉取官方mongodb的docker镜像。

```bash
[root@localhost ~]# docker pull mongo
Using default tag: latest
Trying to pull repository docker.io/library/mongo ... 
latest: Pulling from docker.io/library/mongo
35c102085707: Pull complete 
251f5509d51d: Pull complete 
8e829fe70a46: Pull complete 
6001e1789921: Pull complete 
62fb80b8f88c: Pull complete 
76be8dc9ea13: Pull complete 
c73353d62de1: Pull complete 
9dfe7c37b46c: Pull complete 
1fdf813927b6: Pull complete 
87b9bd03dc66: Pull complete 
24c524d289d7: Pull complete 
306b575ddfff: Pull complete 
ee1475733b36: Pull complete 
Digest: sha256:93c98ffc714faa1fa501297d35670a62835dbb7e62243cee0c491433ea523f30
Status: Downloaded newer image for docker.io/mongo:latest
[root@localhost ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/mongo     latest              cdc6740b66a7        2 weeks ago         361 MB
```

### 3.2. 编写ansible配置文件

	`Ansible`的主要功能在与批量主机操作，为了便于便捷的使用其中的部分主机，可以在inventory file中将其分组命名，默认的inventory file为`/etc/ansibel/hosts`。此处只需要在`mongodb1`主机设置即可。
	
	通过`[mongodb]`设置分组信息，即其下方所有配置IP均为`mongodb`组中主机。`[mongodb:vars]`用于设置`mongodb`组中的公共参数（SSH登录时的用户名、密码）。

```ini
[mongodb]
192.168.33.10
192.168.33.11
192.168.33.12

[mongodb:vars]
ansible_user=root
ansible_ssh_pass=vagrant
```

	通过在`mongodb1`主机`/vagrant_data/ansible`目录下执行`ansible mongodb -m ping`命令，测试配置是否正确。

```bash
[root@localhost ansible]# ansible mongodb -m ping
192.168.33.10 | SUCCESS => &#123;
    "ansible_facts": &#123;
        "discovered_interpreter_python": "/usr/bin/python"
    &#125;, 
    "changed": false, 
    "ping": "pong"
&#125;
192.168.33.12 | SUCCESS => &#123;
    "ansible_facts": &#123;
        "discovered_interpreter_python": "/usr/bin/python"
    &#125;, 
    "changed": false, 
    "ping": "pong"
&#125;
192.168.33.11 | SUCCESS => &#123;
    "ansible_facts": &#123;
        "discovered_interpreter_python": "/usr/bin/python"
    &#125;, 
    "changed": false, 
    "ping": "pong"
&#125;
```

### 3.3. 编写ansible部署脚本

	通过`ansible`可以进行多服务器环境搭建。首先在与虚拟机配置文件所在目录同级的`data`目录下创建`mongodb_install.yml`文件来设置ansible安装步骤，新建的文件会同时映射到虚拟机中的`/vagrant_data`目录下的`mongodb_install.yml`。
	
	其中，需要说明的内容如下：

1. 通过`hosts: all`设置作用于所有已配置的主机，配置文件为`/etc/ansible/hosts`；
2. 通过`vars`设置ansible部署过程中的环境变量，ansible脚本中通过双大括号`&#123;&#123;&#125;&#125;`方式引用；
3. 通过`tasks`设置ansible部署的步骤；
4. 通过`file`模块创建部署服务器上的BASE目录，上传文件会默认在与此配置文件同级的`files`目录中查找；
5. 通过`template`模块上传模板文件到部署服务器，上传的模板文件会默认在与此配置文件同级的`templates`目录中查找；
6. 通过`copy`模块上传文件到部署服务器，与`template`的区别在于`template`上传文件时，会对文件中的双大括号`&#123;&#123;&#125;&#125;`方式引用的参数进行自动填充；
7. 通过`command`模块在部署服务器上执行`shell`命令，执行的命令中`docker-compose -f docker-compose.yml down`用于根据`docker-compose.yml`的配置清除所有已安装的docker容器，`docker-compose -f docker-compose.yml up -d`则是根据`docker-compose.yml`的配置创建docker容器，`docker exec -it mongodb /bin/bash /opt/init-rs.sh`用于执行docker容器的`shell`脚本；
8. 通过`groups.mongodb`可以获取配置文件中`mongodb`组中的所有主机host信息，`inventory_hostname`用于获取`ansible`当前部署的服务器的host信息，而`groups.mongodb.index(inventory_hostname)`则获取当前部署主机在配置的`mongodb`组中的排列顺序，以0开始。即只有第一个配置的服务器才进行副本集配置，配置后其将会作为`primary`主机。
9. 通过`wait_for`设置等到10秒后检测mongdb是否启动完成，因为在设置副本集后，mongodb进行初始化，初始化完成前创建用户信息，会提示`not master`非master主机错误。

```yaml
---
- hosts: all
  user: root
  vars:
    # mongdb初始化管理员用户
    MONGO_INITDB_ROOT_USERNAME: "root"
    # mongdb初始化管理员用户密码
    MONGO_INITDB_ROOT_PASSWORD: "123456"
    # mongdb初始化数据库名称
    MONGO_INITDB_DATABASE: "admin"
    # mongdb普通用户
    MONGO_USER: test
    # mongdb普通用户密码
    MONGO_PWD: 123456
    # mongdb数据库名称
    MONGO_DATABASE: "replicaSetTest"
    # mongdb服务端口
    MONGO_PORT: "27017"
    # mongdb基础目录
    MONGO_BASE_DIR: "/opt/mongodb"
    # mongodb部署方式：single.单机、replication.主从、replicaSet.副本集、sharding.分片
    MONGO_DEPLOY_TYPE: "replicaSet"
    # mongodb 副本集名称
    MONGO_REPL_SET: replica-set-test
    # mongodb 密钥文件
    MONGO_KEY_FILE: /opt/keyfile/mongodb-keyfile
  tasks:
    - name: 创建BASE目录
      file:
        path: "&#123;&#123; item &#125;&#125;"
        state: directory
      with_items:
        - "&#123;&#123; MONGO_BASE_DIR &#125;&#125;/db"
        - "&#123;&#123; MONGO_BASE_DIR &#125;&#125;/config"

    - name: 上传mongodb初始化脚本
      template:
        src: "&#123;&#123; item &#125;&#125;"
        dest: "&#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/&#123;&#123; item &#125;&#125;"
        mode: 0755
      with_items:
        - mongodb-init-rs.sh
        - mongodb-init-db.sh

    - name: 上传mongodb秘钥文件
      copy:
        src: mongodb-keyfile
        dest: "&#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/mongodb-keyfile"
        mode: 0600
        owner: "999"

    - name: 删除旧docker容器
      command: docker-compose -f docker-compose.yml down
      args:
        chdir: /tmp

    - name: 上传docker-compose配置文件
      template:
        src: docker-compose.yml
        dest: /tmp/docker-compose.yml

    - name: 创建docker容器
      command: docker-compose -f docker-compose.yml up -d
      args:
        chdir: /tmp

    - name: 等待5秒，检测mongo服务是否启动
      wait_for:
        port: 27017
        delay: 5
      when: "groups.mongodb.index(inventory_hostname) == 0"

    - name: 设置副本集配置
      command: docker exec -it mongodb /bin/bash /opt/init-rs.sh
      when: "groups.mongodb.index(inventory_hostname) == 0"

    - name: 等待10秒，等待副本集设置完成
      wait_for:
        port: 27017
        delay: 10
      when: "groups.mongodb.index(inventory_hostname) == 0"

    - name: 创建mongo用户信息
      command: docker exec -it mongodb /bin/bash /opt/init-db.sh
      when: "groups.mongodb.index(inventory_hostname) == 0"
```

### 3.4. 生成keyFile密钥文件

	在`mongodb1`主机`/vagrant_data/ansible/files`目录下通过`openssl rand -base64 24 > mongodb-keyfile`命令生成密钥文件`mongodb-keyfile`。

### 3.5. 编写mongodb副本集设置脚本

	在`mongodb1`主机`/vagrant_data/ansible/templates`目录下创建`mongodb-init-rs.sh`用于docker容器创建后，通过`rs.initiate`命令设置副本集信息。其中用到了`ansible`中的`jinja2`模板语言，说明如下：

1. `"&#123;&#123; MONGO_REPL_SET &#125;&#125;"`是用于获取ansible全局参数`MONGO_REPL_SET `的值；
2. `&#123;% for host in groups.mongodb -%&#125;`和`&#123;% endfor %&#125;`为`for`循环语句，即循环`mongodb `组中所有的主机host信息；
3. `&#123;% if groups.mongodb.index(host)|int + 1 < groups.mongodb|length -%&#125;`、`&#123;% else -%&#125;`和`&#123;% endif -%&#125;`用于判断当前循环的主机是否为最后一个主机，如果为最后一个主机，则最后不加`，`。

```bash
#!/bin/bash

mongo <<-EOJS
    rs.initiate(&#123;
        _id:"&#123;&#123; MONGO_REPL_SET &#125;&#125;", 
        members:[
            &#123;% for host in groups.mongodb -%&#125;
            &#123;% if groups.mongodb.index(host)|int + 1 < groups.mongodb|length -%&#125;
            &#123; _id: &#123;&#123; groups.mongodb.index(host) &#125;&#125;, host: "&#123;&#123; host &#125;&#125;:&#123;&#123; MONGO_PORT &#125;&#125;" &#125;, 
            &#123;% else -%&#125;
            &#123; _id: &#123;&#123; groups.mongodb.index(host) &#125;&#125;, host: "&#123;&#123; host &#125;&#125;:&#123;&#123; MONGO_PORT &#125;&#125;" &#125;
            &#123;% endif -%&#125;
            &#123;% endfor %&#125;

        ]
    &#125;);
EOJS
```

	模块转义（模板上传后自动转义）后的文件内容如下：

```bash
#!/bin/bash

mongo <<-EOJS
    rs.initiate(&#123;
        _id:"replica-set-test", 
        members:[
            &#123; _id: 0, host: "192.168.33.10:27017" &#125;, 
            &#123; _id: 1, host: "192.168.33.11:27017" &#125;, 
            &#123; _id: 2, host: "192.168.33.12:27017" &#125;
            
        ]
    &#125;);
EOJS
```

### 3.6. 编写mongodb用户创建脚本

	在`mongodb1`主机`/vagrant_data/ansible/templates`目录下创建`mongodb-init-db.sh`用于docker容器创建后通过`db.createUser`命令创建用户信息，其中也使用到了`ansible`中的`jinja2`模板语言。

```bash
#!/bin/bash

<code>
mongo &#123;&#123; MONGO_INITDB_DATABASE &#125;&#125; <<-EOJS
    db.createUser(&#123;
        user: '&#123;&#123; MONGO_INITDB_ROOT_USERNAME &#125;&#125;', 
        pwd: '&#123;&#123; MONGO_INITDB_ROOT_PASSWORD &#125;&#125;', 
        roles: [&#123; role: 'root', db: '&#123;&#123; MONGO_INITDB_DATABASE &#125;&#125;' &#125;] 
    &#125;);
EOJS
</code>

<code>
mongo &#123;&#123; MONGO_INITDB_DATABASE &#125;&#125; -u &#123;&#123; MONGO_INITDB_ROOT_USERNAME &#125;&#125; -p &#123;&#123; MONGO_INITDB_ROOT_PASSWORD &#125;&#125; <<-EOJS
use &#123;&#123; MONGO_DATABASE &#125;&#125;;
db.createUser(&#123; 
    user: '&#123;&#123; MONGO_USER &#125;&#125;', 
    pwd: '&#123;&#123; MONGO_PWD &#125;&#125;', 
    roles: [&#123; role: 'readWrite', db: '&#123;&#123; MONGO_DATABASE &#125;&#125;' &#125;] 
&#125;);
EOJS
</code>
```

	模板转义（模板上传后自动转义）后文件内容如下：

```bash
#!/bin/bash

<code>
mongo admin <<-EOJS
    rs.status();
    db.createUser(&#123;
        user: 'root', 
        pwd: '123456', 
        roles: [&#123; role: 'root', db: 'admin' &#125;] 
    &#125;);
EOJS
</code>

<code>
mongo admin -u root -p 123456 <<-EOJS
use replicaSetTest;
db.createUser(&#123; 
    user: 'test', 
    pwd: '123456', 
    roles: [&#123; role: 'readWrite', db: 'replicaSetTest' &#125;] 
&#125;);
EOJS
</code>
```

### 3.7. 编写docker安装编排脚本

	在`mongodb1`主机`/vagrant_data/ansible/templates`目录下创建`docker-compose.yaml`文件，用于设置docker容器的安装编排。其中也使用到了`ansible`中的`jinja2`模板语言，只有`mongodb`组中的第一台主机映射了`mongodb-init-rs.sh`、`mongodb-init-db.sh`等副本集初始换脚本，因为从节点`SECONDARY`主机会自动同步主节点`PRIMARY`主机的操作。
	
	同时启动`mongodb`服务的命令中，通过`--dbpath`设置数据存放目录，`--smallfiles`设置以最小的方式初始化`mongodb`，`--replSet`设置副本集名称（所有主从节点均为同一名称），`--keyFile`用于设置主从节点通讯时验证用的密钥文件，`--auth`设置需要密码验证。

```yaml
version: '3.1'

services:
  &#123;% if groups.mongodb.index(inventory_hostname) == 0 -%&#125;
  mongo:
    image: docker.io/mongo
    container_name: mongodb
    restart: always
    ports:
      - &#123;&#123; MONGO_PORT &#125;&#125;:27017
    volumes:
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/db:/data/db
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/mongodb-init-rs.sh:/opt/init-rs.sh
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/mongodb-init-db.sh:/opt/init-db.sh
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/mongodb-keyfile:&#123;&#123; MONGO_KEY_FILE &#125;&#125;
      - /etc/localtime:/etc/localtime
    command:
      mongod --dbpath /data/db --smallfiles --replSet &#123;&#123; MONGO_REPL_SET &#125;&#125; --keyFile &#123;&#123; MONGO_KEY_FILE &#125;&#125; --auth
  &#123;% else -%&#125;
  mongodb:
    image: docker.io/mongo
    container_name: mongodb
    restart: always
    ports:
      - &#123;&#123; MONGO_PORT &#125;&#125;:27017
    volumes:
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/db:/data/db
      - &#123;&#123; MONGO_BASE_DIR &#125;&#125;/config/mongodb-keyfile:&#123;&#123; MONGO_KEY_FILE &#125;&#125;
      - /etc/localtime:/etc/localtime
    command:
      mongod --dbpath /data/db --smallfiles --replSet &#123;&#123; MONGO_REPL_SET &#125;&#125; --keyFile &#123;&#123; MONGO_KEY_FILE &#125;&#125; --auth
  &#123;% endif %&#125;
```

	主节点模板转义（模板上传后自动转义）后文件内容如下：

```yaml
version: '3.1'

services:
  mongo:
    image: docker.io/mongo
    container_name: mongodb
    restart: always
    ports:
      - 27017:27017
    volumes:
      - /opt/mongodb/db:/data/db
      - /opt/mongodb/config/mongodb-init-rs.sh:/opt/init-rs.sh
      - /opt/mongodb/config/mongodb-init-db.sh:/opt/init-db.sh
      - /opt/mongodb/config/mongodb-keyfile:/opt/keyfile/mongodb-keyfile
      - /etc/localtime:/etc/localtime
    command:
      mongod --dbpath /data/db --smallfiles --replSet replica-set-test --keyFile /opt/keyfile/mongodb-keyfile --auth
```

	从节点模板转义（模板上传后自动转义）后文件内容如下：

```yaml
version: '3.1'

services:
  mongodb:
    image: docker.io/mongo
    container_name: mongodb
    restart: always
    ports:
      - 27017:27017
    volumes:
      - /opt/mongodb/db:/data/db
      - /opt/mongodb/config/mongodb-keyfile:/opt/keyfile/mongodb-keyfile
      - /etc/localtime:/etc/localtime
    command:
      mongod --dbpath /data/db --smallfiles --replSet replica-set-test --keyFile /opt/keyfile/mongodb-keyfile --auth
```

## 4. 通过ansible部署mongodb副本集

### 4.1. 关闭所有服务器的selinux

	若`selinux`服务处于开启状态，则docker容器中映射的共享目录会存在权限问题。
	
	临时关闭方法如下：

```bash
[root@localhost ansible]# getenforce
Enforcing
[root@localhost ansible]# setenforce 0
[root@localhost ansible]# getenforce
Permissive
```

	永久关闭方式如下，修改`/etc/sysconfig/selinux`配置文件中的`SELINUX=enforcing`参数修改为`SELINUX=disabled`。

### 4.2. 部署mongodb副本集

	在`mongodb1`主机`/vagrant_data/ansible`目录下执行`ansible-playbook mongodb_install.yml`命令，进行多服务器同时部署。

```bash
[root@localhost ansible]# ansible-playbook mongodb_install.yml 

PLAY [all] ******************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************************************************************************************************
ok: [192.168.33.10]
ok: [192.168.33.11]
ok: [192.168.33.12]

TASK [创建BASE目录] *************************************************************************************************************************************************************************************************************************
ok: [192.168.33.10] => (item=/opt/mongodb/db)
ok: [192.168.33.11] => (item=/opt/mongodb/db)
ok: [192.168.33.12] => (item=/opt/mongodb/db)
ok: [192.168.33.10] => (item=/opt/mongodb/config)
ok: [192.168.33.11] => (item=/opt/mongodb/config)
ok: [192.168.33.12] => (item=/opt/mongodb/config)

TASK [上传mongodb初始化脚本] *******************************************************************************************************************************************************************************************************************
ok: [192.168.33.10] => (item=mongodb-init-rs.sh)
ok: [192.168.33.11] => (item=mongodb-init-rs.sh)
ok: [192.168.33.12] => (item=mongodb-init-rs.sh)
ok: [192.168.33.10] => (item=mongodb-init-db.sh)
ok: [192.168.33.11] => (item=mongodb-init-db.sh)
ok: [192.168.33.12] => (item=mongodb-init-db.sh)

TASK [上传mongodb秘钥文件] ********************************************************************************************************************************************************************************************************************
ok: [192.168.33.10]
ok: [192.168.33.11]
ok: [192.168.33.12]

TASK [删除旧docker容器] **********************************************************************************************************************************************************************************************************************
changed: [192.168.33.10]
changed: [192.168.33.12]
changed: [192.168.33.11]

TASK [上传docker-compose配置文件] *************************************************************************************************************************************************************************************************************
ok: [192.168.33.10]
ok: [192.168.33.11]
ok: [192.168.33.12]

TASK [创建docker容器] ***********************************************************************************************************************************************************************************************************************
changed: [192.168.33.10]
changed: [192.168.33.12]
changed: [192.168.33.11]

TASK [等待5秒，检测mongo服务是否启动] ***************************************************************************************************************************************************************************************************************
skipping: [192.168.33.11]
skipping: [192.168.33.12]
ok: [192.168.33.10]

TASK [设置副本集配置] **************************************************************************************************************************************************************************************************************************
skipping: [192.168.33.11]
skipping: [192.168.33.12]
changed: [192.168.33.10]

TASK [等待10秒，，等待副本集设置完成] *****************************************************************************************************************************************************************************************************************
skipping: [192.168.33.11]
skipping: [192.168.33.12]
ok: [192.168.33.10]

TASK [创建mongo用户信息] **********************************************************************************************************************************************************************************************************************
skipping: [192.168.33.11]
skipping: [192.168.33.12]
changed: [192.168.33.10]

PLAY RECAP ******************************************************************************************************************************************************************************************************************************
192.168.33.10              : ok=11   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.33.11              : ok=7    changed=2    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0   
192.168.33.12              : ok=7    changed=2    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0 
```

### 4.3. 安装验证

	通过`Robo 3T`管理工具连接`mongodb`进行测试。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901224948739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

	在`192.168.33.10`主节点创建一个集合，并插入一条数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901225000892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

	其他两个从节点自动同步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901225016124.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

	在主节点test集合中创建一个文档。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901225025553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901225037796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

	从节点已自动同步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190901225046860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019090122505716.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Nkd2FuZzE5ODkxMg==,size_16,color_FFFFFF,t_70)
	通过`docker stop mongodb`命令停止主节点mongdb服务。

```bash
[root@localhost ansible]# docker stop mongodb
mongodb
[root@localhost ansible]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
4673db8d361e        docker.io/mongo     "docker-entrypoint..."   22 minutes ago      Exited (0) 5 seconds ago                       mongodb
```

	从节点`192.168.33.11`已自动被选举为主节点。

```bash
[root@localhost ~]# docker exec -it mongodb mongo
MongoDB shell version v4.2.0
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session &#123; "id" : UUID("e0037f48-d93d-4796-b96e-19fbcd1125da") &#125;
MongoDB server version: 4.2.0
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
replica-set-test:PRIMARY> 
```

	原主节点重启后，变成了从节点。

```bash
[root@localhost ansible]# docker start mongodb
mongodb
[root@localhost ansible]# docker exec -it mongodb mongo
MongoDB shell version v4.2.0
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session &#123; "id" : UUID("0c7af1af-94e2-43e3-b101-5d5eb966b567") &#125;
MongoDB server version: 4.2.0
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
replica-set-test:SECONDARY> exit
bye
```