# 如何创建 stack

## 技术栈的选择

技术栈作为整个应用实现的框架，作为整个栈的构建人员首先应该了解当前实现某种类型应用最好的一套语言，框架以及相应的应用。这里以 `java` 作为我们的开发语言为例提供一套以 [`jersey`](https://jersey.java.net/) 作为 REST 服务的框架，`mysql` 作为后端存储的栈.

## 技术栈的构建

在 `cde paas` 中一个应用的部署流程如下所示。在这个流程的多个环节都依赖于所提供的 `stackfile`。

> Codebase ---build-->　Runnable App ---verify---> Release ---deploy---> Running App

其中，

* `codebase` 以栈构建人员所提供的 `template` 作为基础代码
* `build` 将 `codebase` 编译成为一个 `runnable app` 依赖于 stack 中所提供的 `build image`
* `verify` 对 应用进行集成测试，其测试的流程由 `verify image` 所提供
* `deploy` 依据 `stackfile` 中所描述的 `services` 构建应用所以来的其他服务

那么我们的技术栈构建人员需要做的事情主要有如下几个步骤:

### 定义 template 作为 codebase 的基础结构

template 做为 codebase 所要遵循的代码结构对一个项目的发展有重要的指导作用，template 需要包含一个项目基本的文件夹结构（src, test, integration test），所使用的代码的包管理描述（build.gradle）以及数据库 `migration` 脚本的目录等。[javajersey_api](https://github.com/tw-cde/javajersey_api) 就是我们所提供的一个模板项目。

### 准备将 codebase 变为 runnable app 的 build image

定义好 codebase 的结构后，我们就可以根据结构去定义相应的 build 流程,此时我们需要一个 build image 来将我们的 codebase 编译成一个可执行的 docker image。codebase 可以访问的环境变量如下:

|变量名|含义|
|----|----|
|CODEBAE|codebase被挂载daocontainer里面的目录|
|CACHE_DIR|build image 可以使用这个目录来缓存build过程中的文件,比如maven的jar包,用来加速整个build流程|
|HOST|build image启动的宿主机IP|
|APP_NAME|build image 在build时,操作的app的名称|
|IMAGE|build 成功之后image的名称|

在 build image 里面可以使用这几个变量获取代码，进行相应的编译操作。在将 codebase 编译为可以运行的的 app (runnable app) 后，需要 push 到 docker registry。其中 `build` 的主要内容如下所示

```
CODEBASE_DIR=$CODEBASE
HOST_IP=$(ip route|awk '/default/ { print $3 }')

export DB_USERNAME=mysql
export DB_PASSWORD=mysql
export DB_NAME=testdb

echo
puts_step "Launching baking services ..."
MYSQL_CONTAINER=$(docker run -d -P -e MYSQL_USER=$DB_USERNAME -e MYSQL_PASSWORD=$DB_PASSWORD -e MYSQL_DATABASE=$DB_NAME -e MYSQL_ROOT_PASSWORD=$DB_PASSWORD mysql:5.6)
MYSQL_PORT=$(docker inspect -f '{{(index (index .NetworkSettings.Ports "3306/tcp") 0).HostPort}}' ${MYSQL_CONTAINER})
until docker exec $MYSQL_CONTAINER mysql -h127.0.0.1 -P3306 -umysql -pmysql -e "select 1" &>/dev/null ; do
    echo "...."
    sleep 1
done

export DB_HOST=$HOST_IP
export DB_PORT=$MYSQL_PORT
export DATABASE="jdbc:mysql://$DB_HOST:$DB_PORT/$DB_NAME?user=$DB_USERNAME&password=$DB_PASSWORD&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&createDatabaseIfNotExist=true"

puts_step "Complete Launching baking services"
echo

cd $CODEBASE_DIR

echo
puts_step "Start migratioin ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle fC fM &> process.log
puts_step "Migration complete"
echo

echo
puts_step "Start test ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle clean test -i &> process.log
puts_step "Test complete"
echo

puts_step "Start generate standalone ..."
GRADLE_USER_HOME="$CACHE_DIR" gradle standaloneJar &>process.log
puts_step "Generate standalone Complete"

(cat  <<'EOF'
#!/bin/sh

export DATABASE="jdbc:mysql://127.0.0.1:$DB_PORT/$DB_NAME?user=mysql&password=mysql&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&createDatabaseIfNotExist=true"
flyway migrate -url="$DATABASE" -locations=filesystem:`pwd`/dbmigration
[ -d `pwd`/initmigration  ] && flyway migrate -url="$DATABASE" -locations=filesystem:`pwd`/initmigration -table="init_version" -baselineOnMigrate=true -baselineVersion=0
java -jar app-standalone.jar
EOF
) > wrapper.sh

(cat << EOF
FROM hub.deepi.cn/jre-8.66

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
puts_step "Building image $IMAGE ..."
docker build -q -t $IMAGE . &>process.log
puts_step "Building image $IMAGE complete "
echo
```

具体内容见 [https://github.com/tw-cde/stackrepo/blob/master/javajersey-build/build.sh](https://github.com/tw-cde/stackrepo/blob/master/javajersey-build/build.sh)

### 准备验证 Runnable App 变为 Release 的 verify image

在 verfiy 阶段，cde paas 采用 `build image` 所构建的 `runnable app` 启动一个 `lambda env` 即临时的环境用于应用的集成测试, verify 需要获取 `lambda env` 所提供的 `entrypoint` 作为参数执行调用项目中的集成测试。例如在实例中我们将集成测试编译为 `verify.jar` 并执行它以完成测试。其中 `verify` 的主要内容如下所示

```
puts_step "Building verify jar..."
GRADLE_USER_HOME="$CACHE_DIR" gradle itestJar &>process.log
puts_step "Build verify finished"
puts_step "Start verify"
ENTRYPOINT=http://$ENDPOINT java -jar build/libs/verify-standalone.jar
puts_step "Verify finished"
```

具体内容见 [https://github.com/tw-cde/stackrepo/tree/master/javajersey-verify](https://github.com/tw-cde/stackrepo/tree/master/javajersey-verify)

### 将上面的准备好的东西用 stackfile 描述出来

`java-jersey` 栈的 `stackfile` 如下所示

```
name: "javajersey"
description: "A sample java jersey stack"
template:
  type: "git"
  uri: 'https://github.com/aisensiy/javajersey_api.git'
tags:
    - "java"
languages:
  - name: "java"
    version: "1.8"
  - name: "xml"
    version: "4.0"
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
      image: '192.168.99.100:5000/javajersey-build'
      mem: 512
      cpus: 0.5
    verify:
      image: '192.168.99.100:5000/javajersey-verify'
      mem: 512
      cpus: 0.5
    main: yes
    mem: 512
    cpus: 0.4
    instances: 1
    links:
      - db
    expose: 8088
    health:
      - protocol: "COMMAND"
        command: "exit 0"
        interval: "3"
        timeout: "2"
  db:
    mem: 256
    instances: 1
    cpus: 0.2
    image: tutum/mysql
    environment:
      MYSQL_PASS: "mysql"
      MYSQL_USER: "mysql"
      MYSQL_PASSWORD: "mysql"
      ON_CREATE_DB: "stacks"
      EXTRA_OPTS: "--lower_case_table_names=1"
    expose: 3306
    volumes:
      - data:/var/lib/mysql
    health:
      - protocol: "COMMAND"
        interval: "3"
        timeout: "2"
        command: "exit 0"

```

其中 `name` `description` `tags` `languages` `frameworks` `tools` 描述了整个栈的一些基本信息

`template` 描述了栈所指定的项目模板，其中包含了一些基础代码。

`services` 描述了使用该栈所构建的应用所包含的服务。其中 `services` 有两类服务，一种是 `main service` 其包含了字段 `main: yes` 为使用项目的 `codebase` 所构建的应用，实例中的 `web` 为这样的服务，它包含了其所使用的 `build` `verify` 的 `image`。其他的服务为 `backing service` 是该项目所以来的其他应用。这些 `backing service` 都是已经都建好的 `docker image`，例如实例中的 `db` 为 `mysql` 的 `docker image`。

## 更新栈

如果在使用的过程中发现所使用的栈存在一些问题可以修改栈。

首先需要标记栈为 `UNPUBLISHED`

    cde stacks:unpublish <stack-id>

之后除去栈的创建者之外所有使用这个栈的应用都将不再能 `git push cde master`，并且也不能利用这个栈创建新的应用了。然后通过 `cde stacks:update <new-stackfile>` 更新栈。

**注意** 对于正在被其他应用使用的技术栈，对其进行修改应该非常的谨慎。只有如下的情况是建议采用 `stacks:update` 去修改栈：

1. bug typo 的修复
2. 服务的小幅升级，例如 abc:v1.1 -> abc:v1.2

如下的情况是不建议修改栈的:

1. 更换服务，例如 `mysql` to `postgresql`
2. 架构升级，`redis` to `redis-sentinel`
3. 添加服务
4. 修改栈中所使用的语言，框架，构建工具

如上的修改已经彻底的改变了原有的技术栈，对栈的修改会导致使用该栈的应用构建或部署失败。当有如上的需求时应当创建新的栈并将相应的应用通过 `apps:stack-update` 更换栈。

## 删除栈

对于废弃不再使用的栈可以对其记性删除，但是删除栈的前提是 cde paas 中没有任何应用使用该技术栈。将使用该栈的应用通过

    cde apps:stack-update --app=<app-name> --stack=<stack-name>

切换栈之后，通过

    cde stacks:remove <stack-name>

删除栈。
