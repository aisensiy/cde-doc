# 创建和管理 jersey and mysql stack

## 目录

* Workflow of cde
* 准备工作
	* 安装 docker-machine
	* 安装 cde 客户端
* Build stack
	* What do we need in a stack
	* Stackfile
	* A boilerplate project
	* 创建 Build
	* 创建 Verify
	* 在 cde 中测试 stack

## Workflow of cde

![](workflow-of-cde.png)

## 准备工作

### 安装 docker-machine

1. install [cask](https://caskroom.github.io/)

   ```
   brew tap caskroom/cask
   brew install cask
   ```
2. install [docker toolbox](https://www.docker.com/products/docker-toolbox)

   ```
   brew cask install dockertoolbox
   ```
3. install a docker machine

	```
	docker-machine create --driver=virtualbox \
						  --virtualbox-disk-size=100000 \
						  --virtualbox-memory=400 \
						  --virtualbox-host-dns-resolver \
						  --virtualbox-dns-proxy  \
						  --engine-insecure-registry=10.0.0.0/8 \
						  --engine-insecure-registry=172.16.0.0/12 \
						  --engine-insecure-registry=192.168.0.0/16 \
						  registry
	```

### 安装 cde 客户端

1. 在浏览器打开链接 `https://drive.google.com/open?id=0B2uUP4BIHpNDRWRvV0Q2Z3F5YTQ` 下载 cde 客户端
2. 将 cde 客户端拷贝到 `$PATH` 下
   
   ```
   cd ~/Downloads
   chmod 755 cde
   cp cde /usr/local/bin
   ```
3. 检查 cde 客户端是否可以使用
   
   ```
   cde -h
   ```

## Build stack

### What do we need in a stack

在创建一个栈的时候，需要准备如下的内容：

1. stackfile，一个 cde paas 定义的栈的描述文件，其中定义了栈的所有依赖
2. 一个适用于该技术栈的项目模板，所有使用该技术栈的项目都可以以此作为项目的初始代码
3. build image，用于将项目进行编译构建的 docker image
4. verify image，用于对完成了 build 步骤的项目进行功能测试的 docker image

### Stackfile

```
name: "..."
description: "..."
template:
  type: "git"
  uri: "..."
tags:
    - "java"
languages:
  - name: "java"
    version: "1.8"
frameworks:
  - name: "jersey"
    version: "2.17"
  - name: "mybatis"
    version: "3.3"
tools:
  - name: "gradle"
    version: "2.8"
services:
  web:
    build:
      image: '...'
      mem: 512
      cpus: 0.5
    verify:
      image: '...'
      mem: 512
      cpus: 0.5
    main: yes
    mem: 512
    cpus: 0.4
    instances: 1
    links:
      - db
    expose: 8088
  db:
    mem: 256
    instances: 1
    cpus: 0.2
    image: tutum/mysql
    expose: 3306
    volumes:
      - data:/var/lib/mysql
```

cde 所定义的技术栈文件如上所示，其中

* `name` `description` `tags` `languages` `frameworks` `tools` 描述了整个栈的一些基本信息
* `template` 描述了构建栈时所提供的项目模板
* `services` 描述了使用该栈所构建的应用所包含的服务。`services` 分为两种类型，一种是 `main service` 其包含了字段 `main: yes` 为所构建的应用。实例中的 `web` 为这样的服务，它包含了其所使用的 `build` `verify` 的 `image`。另一种为 `backing service` 是该项目所以来的其他应用。这些 `backing service` 都是已经都建好的 `docker image`，例如实例中的 `db` 为 `mysql` 的 `docker image`。一个 stack 只能包含一个 `main service`并可以包含多个 `backing service`。

### A boilerplate project

在 cde 所构建的应用与采用其他方式构建的应用相比，需要遵循如下一些原则：

1. 将应用对其他服务的依赖以环境变量的形式引入
2. 项目包含三个部分，源代码、单元测试以及集成测试

[https://github.com/aisensiy/cde-jersey-mysql-init-project/](https://github.com/aisensiy/cde-jersey-mysql-init-project/) 

### 创建 build

#### 准备 Dockerfile

build image 的工作主要包含如下内容：

1. 为应用的单元测试准备环境（例如准备数据库）并执行单元测试
2. 将应用编译打包
3. 将应用容器化

`Dockerfile` 如下:

```bash
FROM hub.deepi.cn/gradle:0.1

ENTRYPOINT ["./build.sh"]

ENV DOCKER_BUCKET get.docker.com
ENV DOCKER_VERSION 1.9.1
RUN curl -sjkSL "https://${DOCKER_BUCKET}/builds/Linux/x86_64/docker-$DOCKER_VERSION" -o /usr/bin/docker \
	&& chmod +x /usr/bin/docker

ADD build.sh build.sh
RUN chmod a+x build.sh
```

在 `build` 中需要构建一个可以将 code 变成 `runnable app` 的 `docker image`，因此所有的 `build` 都需要在 `Dockerfile` 中安装 `docker`。在 `docker run jersey-mysql-build` 时会执行 `build.sh` 生成一个名为 `$IMAGE` 的 `docker image`。`cde PaaS` 在执行 `build` 时会提供一些环境变量和 `docker volume` 用于提供项目代码和所有构建的 docker image 的名称。

`build.sh` 如下所示：

```bash
#!/bin/bash

# builder 在调用 stack 的 build image 时会传入如下一些环境变量
# APP_NAME:  应用的名称
# CODEBASE:  应用代码的目录
# CACHE_DIR: build image 可以使用这个目录来缓存build过程中的文件,比如maven的jar包,用来加速整个build流程
# IMAGE:     build 成功之后image的名称

set -eo pipefail

on_exit() {
    last_status=$?
    if [ "$last_status" != "0" ]; then
        if [ -f "process.log" ]; then
          cat process.log
        fi

        if [ -n "$MYSQL_CONTAINER" ]; then
            echo
            echo "Cleaning ..."
            docker stop $MYSQL_CONTAINER &>process.log && docker rm $MYSQL_CONTAINER &>process.log
            echo "Cleaning complete"
            echo
        fi
        exit 1;
    else
        if [ -n "$MYSQL_CONTAINER" ]; then
            echo
            echo "Cleaning ..."
            docker stop $MYSQL_CONTAINER &>process.log && docker rm $MYSQL_CONTAINER &>process.log
            echo "Cleaning complete"
            echo
        fi
        exit 0;
    fi
}

trap on_exit HUP INT TERM QUIT ABRT EXIT

HOST_IP=$(ip route|awk '/default/ { print $3 }')

export DB_USERNAME=mysql
export DB_PASSWORD=mysql
export DB_NAME=testdb

echo
echo "Launching baking services ..."
MYSQL_CONTAINER=$(docker run -d -P -e MYSQL_USER=$DB_USERNAME -e MYSQL_PASS=$DB_PASSWORD -e ON_CREATE_DB=$DB_NAME -e MYSQL_ROOT_PASSWORD=$DB_PASSWORD tutum/mysql)
MYSQL_PORT=$(docker inspect -f '{{(index (index .NetworkSettings.Ports "3306/tcp") 0).HostPort}}' ${MYSQL_CONTAINER})
until docker exec $MYSQL_CONTAINER mysql -h127.0.0.1 -P3306 -umysql -pmysql -e "select 1" &>/dev/null ; do
    echo "...."
    sleep 1
done

export DB_HOST=$HOST_IP
export DB_PORT=$MYSQL_PORT

echo "Complete Launching baking services"
echo

cd $CODEBASE

echo
echo "Start migratioin ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle fC fM &> process.log
echo "Migration complete"
echo

echo
echo "Start test ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle clean test -i &> process.log
echo "Test complete"
echo

echo "Start generate standalone ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle standaloneJar &>process.log
echo "Generate standalone Complete"

(cat  <<'EOF'
#!/bin/sh

export DATABASE="jdbc:mysql://127.0.0.1:$DB_PORT/$DB_NAME?user=$DB_USERNAME&password=$DB_PASSWORD&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&createDatabaseIfNotExist=true"
flyway migrate -url="$DATABASE" -locations=filesystem:`pwd`/dbmigration -baselineOnMigrate=true -baselineVersion=0
[ -d `pwd`/initmigration  ] && flyway migrate -url="$DATABASE" -locations=filesystem:`pwd`/initmigration -table="init_version" -baselineOnMigrate=true -baselineVersion=0
java -jar app-standalone.jar
EOF
) > wrapper.sh

(cat << EOF
FROM hub.deepi.cn/jre-8.66:0.1

CMD ["./wrapper.sh"]

RUN apk --update add tar
RUN mkdir /usr/local/bin/flyway && \
    curl -jksSL https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/4.0/flyway-commandline-4.0.tar.gz \
    | tar -xzf - -C /usr/local/bin/flyway --strip-components=1
ENV PATH /usr/local/bin/flyway/:\$PATH

ADD build/libs/app-standalone.jar app-standalone.jar

ADD wrapper.sh wrapper.sh
RUN chmod +x wrapper.sh
ENV APP_NAME \$APP_NAME

ADD src/main/resources/db/migration dbmigration
COPY src/main/resources/db/init initmigration

EOF
) > Dockerfile

echo
echo "Building image $IMAGE ..."
docker build -q -t $IMAGE . &>process.log
echo "Building image $IMAGE complete "
echo
```

可以看到 `build.sh` 首先执行了应用的单元测试，然后将应用打包，最后生成一个 `Dockerfile` 并使用这个 `Dockerfile` 构建一个新的 `$IMAGE`。所生成的 `Dockerfile` 的内容包含了如下工作:


1. 采用 `hub.deepi.cn/jre-8.66` 作为 `base image`

   ```
   FROM hub.deepi.cn/jre-8.66:0.1
   ```

   `jre-8.66` 是 cde paas 做过定制的 jre 环境，包含了服务发现的机制
   
2. 安装 `flyway`：

	```
	RUN apk --update add tar
	RUN mkdir /usr/local/bin/flyway && \
	    curl -jksSL https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/4.0/flyway-commandline-4.0.tar.gz \
	    | tar -xzf - -C /usr/local/bin/flyway --strip-components=1
	ENV PATH /usr/local/bin/flyway/:\$PATH
	```
	
3. 添加准备好的 jar 包

   ```
   ADD build/libs/app-standalone.jar app-standalone.jar
   ```
   
4. 拷贝 migration 文件

   ```
   ADD src/main/resources/db/migration dbmigration
   COPY src/main/resources/db/init initmigration
   ```

5. 提供应用的启动脚本 `wrapper.sh`

   ```
   ADD wrapper.sh wrapper.sh
   RUN chmod +x wrapper.sh
   CMD ["./wrapper.sh"]
   ```

	`wrapper.sh`:
	
   ```
	#!/bin/sh

	export DATABASE_URL="jdbc:mysql://127.0.0.1:$DB_PORT/$DB_NAME?user=$DB_USERNAME&password=$DB_PASSWORD&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&createDatabaseIfNotExist=true"
	flyway migrate -url="$DATABASE_URL" -locations=filesystem:`pwd`/dbmigration -baselineOnMigrate=true -baselineVersion=0
	[ -d `pwd`/initmigration  ] && flyway migrate -url="$DATABASE_URL" -locations=filesystem:`pwd`/initmigration -table="init_version" -baselineOnMigrate=true -baselineVersion=0
	java -jar app-standalone.jar
   ```

#### 在本地环境测试 build

为了测试 build 是否可以运行，我们首先需要 build 这个 `java-jersey-build`:

    $ cd build; docker build -t java-jersey-build .

为了测试 build 是否可以生成 `runnable app` 我们需要提供在 `cde PaaS` 中 `builder` 所提供的环境变量和 `volume`:

    $ docker run \
    	-v $PWD/template:/codebase \
     	-e CODEBASE=/codebase \
     	-e IMAGE=test-jersey-mysql \
     	-v /tmp:/cache \
     	-e CACHE_DIR=/cache \
     	-v /var/run/docker.sock:/var/run/docker.sock \
     	java-jersey-build

最后我们测试所生成的 `test-jersey-mysql`

    $ docker run test-jersey-mysql

### 创建 verify

#### 准备 Dockerfile

`verify` 用于执行 `End to End` 的功能测试。在 build 完成之后，`builder` 会根据 `stackfile` 构建一个 `lambda` 环境，并为 `verify` 提供 `lambda` 环境的 `endpoint` 用于测试。

```bash
FROM hub.deepi.cn/gradle:0.1

ENTRYPOINT ["./verify.sh"]

WORKDIR /home
ADD verify.sh verify.sh
RUN chmod a+x verify.sh
```

`verify.sh`:

```
#!/bin/bash

# builder 在调用 stack 的 verify image 时会传入如下一些环境变量
# APP_NAME:  应用的名称
# CODEBASE:  应用代码的目录
# CACHE_DIR: build image 可以使用这个目录来缓存build过程中的文件,比如maven的jar包,用来加速整个build流程
# ENDPOINT:  在执行 verify 之前，builder 会采用已经在 build 过程中构建的 docker image 创建一个临时的
#            lambda 环境，用于测试，ENDPOINT 就是这个 lambda 环境的入口，包含了 IP 和端口

set -eo pipefail

on_exit() {
    last_status=$?
    if [ "$last_status" != "0" ]; then
        if [ -f "process.log" ]; then
          cat process.log
        fi

        exit 1;
    else
        exit 0;
    fi
}

trap on_exit HUP INT TERM QUIT ABRT EXIT

cd $CODEBASE

echo
echo "Building verify jar..."
GRADLE_USER_HOME="$CACHE_DIR" gradle itestJar &>process.log
echo "Build verify finished"
echo

echo
echo "Start verify"
ENTRYPOINT=http://$ENDPOINT java -jar build/libs/verify-standalone.jar
echo "Verify finished"
echo

```

在 template 中有一个文件件 `src/itest` 包含了用于功能测试的 `Concordian` 代码。通过 

```
GRADLE_USER_HOME="$CACHE_DIR" gradle itestJar &>process.log
```

将测试代码打包。

通过 

```
ENTRYPOINT=http://$ENDPOINT java -jar build/libs/verify-standalone.jar
```

执行功能测试。其中 `ENDPOINT` 在 cde PaaS 在创建 `lambda` 环境之后提供给 `verify`。

#### 在本地环境测试 verify

构建 `jersey-mysql-verify`

    $ docker build -t jersey-mysql-verify .

测试 `jersey-mysql-verify`

    $ docker run -e ENDPOINT=www.baidu.com \
    	-v $PWD/template:/codebase \
     	-e CODEBASE=/codebase \
     	-v /tmp:/cache \
     	-e CACHE_DIR=/cache \
     	jersey-mysql-verify

### 在 cde 中测试 stack

1. Prepare stackfile
	
	```
	name: "jersey-mysql"
	description: "A sample java jersey stack"
	template:
	  type: "git"
	  uri: 'https://github.com/aisensiy/cde-jersey-mysql-init-project.git'
	tags:
	    - "java"
	languages:
	  - name: "java"
	    version: "1.8"
	frameworks:
	  - name: "jersey"
	    version: "2.17"
	  - name: "mybatis"
	    version: "3.3"
	tools:
	  - name: "gradle"
	    version: "2.8"
	services:
	  web:
	    build:
	      image: 'hub.deepi.cn/jersey-mysql-build'
	      mem: 512
	      cpus: 0.5
	    verify:
	      image: 'hub.deepi.cn/jersey-mysql-verify'
	      mem: 512
	      cpus: 0.5
	    main: yes
	    mem: 512
	    cpus: 0.2
	    instances: 1
	    links:
	      - db
	    expose: 8088
	    environment:
	      DB_NAME: 'data_store'
	      DB_USERNAME: 'mysql'
	      DB_PASSWORD: 'mysql'
	  db:
	    mem: 512
	    instances: 1
	    cpus: 0.2
	    image: "tutum/mysql"
	    environment:
	      MYSQL_ROOT_PASSWORD: "mysql"
	      MYSQL_USER: "mysql"
	      MYSQL_PASS: "mysql"
	      ON_CREATE_DB: "data_store"
	      MYSQL_PASSWORD: "mysql"
	      MYSQL_DATABASE: "data_store"
	      EXTRA_OPTS: "--lower_case_table_names=1"
	    expose: 3306
	    volumes:
	      - db:/var/lib/mysql
	```
	
2. Create stack
    
   ```
   cde register <entrypoint-of-cde>
   cde keys:add <key>
   cde stacks:create stackfile.yml
   ```   
   
3. Create app

	```
	cde apps:create jersey-mysql-test-app jersey-mysql
	```
	
4. Push
   
   ```
   git push cde master
   ```