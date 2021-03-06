# 创建和管理 python flask stack

这里给出一个以 `python` `flask` 为例的栈构建流程，说明如何在本地环境创建和测试一个栈以及如何将栈所需要的 `stackfile` `build` `verify` `template` 等，并说明如何利用 `cde` 命令行工具修改已经创建的栈。构建栈的过程中涉及到了 cde paas 构建和部署流程中的概念，如果还没有对 cde paas 构建部署流程没有了解，请到 [cde paas 介绍](http://introduction.cde.paas)。

## 创建栈的目录结构

首先准备一个构建栈的目录结构

```
.
├── stackfile.yml
├── build
│   ├── Dockerfile
│   └── build.sh
├── template
│   ├── main.py
│   └── requirements.txt
└── verify
    ├── Dockerfile
    └── rootfs
        └── bin
            └── verify.sh
```

如上所示为构建栈时的目录结构，其中 `build` 和 `verify` 分别对应着 `stackfile` 中需要构建 `build image` 和 `verify image` 的 `Dockerfile`。`template` 为用于构建栈的测试项目。

## 准备模板项目

为创建一个新的栈，首先我们需要准备一个模板项目用于在本地和 Cde PaaS 中测试栈是否可以成功的构建和测试应用。这里我们给出一个非常简单的的例子。

`template/main.py`:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

`template/requirements.txt`:

```
flask == 0.10.1
```

所要创建的 `stack` 采用 `python` 语言中的 `flask` 框架提供 web 服务，模板项目的代码见 <http://flask.pocoo.org/>。`requirements.txt` 用于描述项目的依赖。

## 创建 build

### 准备 Dockerfile

`build` 用于准备一个构建采用该技术栈的应用的 `docker image`。由于需要在这个 `docker image` 运行时创建新的 `docker image` 因此所有的 `build` 都需要在 `Dockerfile` 中安装 `docker`。由于 `python` 为解释型语言，并且我们目前的项目不需要额外的编译构建工作，因此不需要安装其他的依赖。

`Dockerfile`:

```bash
FROM alpine:3.3

ENTRYPOINT ["./build.sh"]

RUN apk update && apk upgrade && apk --update add \
    libstdc++ tzdata bash curl

ENV DOCKER_BUCKET get.docker.com
ENV DOCKER_VERSION 1.9.1
RUN curl -sjkSL "https://${DOCKER_BUCKET}/builds/Linux/x86_64/docker-$DOCKER_VERSION" -o /usr/bin/docker \
	&& chmod +x /usr/bin/docker

ADD build.sh build.sh
RUN chmod a+x build.sh
```

在 `build` 中需要构建一个可以将 code 变成 `runnable app` 的 `docker image`，那么在 `docker run flask-build` 时会执行 `build.sh` 生成一个名为 `$IMAGE` 的 `docker image`。`cde PaaS` 在执行 `build` 时会提供一些环境变量和 `docker volume` 用于提供项目代码和所有构建的镜像名称。`build.sh` 如下所示：

```bash
#!/bin/bash

CODEBASE_DIR=$CODEBASE

cd $CODEBASE_DIR

echo 'write Dockerfile'

(cat << EOF
FROM python:2.7

ENV APP_HOME /myapp
RUN mkdir \$APP_HOME
WORKDIR \$APP_HOME

ADD requirements.txt \$APP_HOME/requirements.txt
RUN pip install -r \$APP_HOME/requirements.txt
ADD . \$APP_HOME
CMD ["python", "main.py"]
EOF
) > Dockerfile

echo "Building image $IMAGE ..."
docker build -t $IMAGE . &>process.log
echo "Building image $IMAGE complete"
```

可是看到 `build.sh` 首先进入项目目录，然后生成一个 `Dockerfile` 并使用这个 `Dockerfile` 构建一个新的 `$IMAGE`。所生成的 `Dockerfile` 的内容包含了如下工作:


1. 采用 `python:2.7` 作为 `base image`

   ```
   FROM python:2.7
   ```
2. 安装项目依赖

   ```
   ADD requirements.txt \$APP_HOME/requirements.txt
   RUN pip install -r \$APP_HOME/requirements.txt
   ```
3. 拷贝项目代码

   ```
   ADD . \$APP_HOME
   ```

4. 启动应用

   ```
   CMD ["python", "main.py"]
   ```

### 在本地环境测试 build

为了测试 build 是否可以运行，我们首先需要 build 这个 `flask-build`:

    cd build; docker build -t flask-build .

为了测试 build 是否可以生成 `runnable app` 我们需要提供在 `cde PaaS` 中 `builder` 所提供的环境变量和 `volume`:

    docker run \
    	-v $PWD/template:/codebase \
     	-e CODEBASE=/codebase \
     	-e IMAGE=test-flask \
     	-v /var/run/docker.sock:/var/run/docker.sock \
     	flask-build

最后我们测试所生成的 `test-flask`

    $ docker run test-flask
    * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)

## 创建 verify

### 准备 Dockerfile

`verify` 用于执行 `End to End` 的功能测试，通常是调用在项目中的测试。这里我们通过简单的 `curl` 测试应用的路由是否工作。

`verify` 的 Dockerfile 中包含如下工作

1. 安装 verify 执行所需要的一些以来，例如 `curl`
2. 安装在 cde 中创建临时环境的工具 `lambda`
3. 添加用于执行功能测试的脚本

`Dockerfile`:

```bash
FROM alpine:3.3

ENTRYPOINT ["verify.sh"]

RUN apk update && apk upgrade && apk --update add \
    libstdc++ tzdata bash curl jq

RUN apk add --no-cache python && \
    python -m ensurepip && \
    rm -r /usr/lib/python*/ensurepip && \
    pip install --upgrade pip setuptools && \
    rm -r /root/.cache

RUN curl -sjkSL "https://github.com/tw-cde/cde-client-binary/releases/download/0.1.5/lambda" -o /bin/lambda

COPY rootfs /

RUN chmod -R +x /bin/
```

用于执行功能测试的脚本如下所示

`verify.sh`:

```bash
#!/bin/bash

set -eo pipefail

on_exit() {
    last_status=$?
    trap '' HUP INT TERM QUIT ABRT EXIT
    local exit_status=0
    if [ "$last_status" != "0" ]; then
        if [ -f "process.log" ]; then
          cat process.log
        fi
        exit_status=1
    fi

    if [ "$LAMBDA_URI" != "" ]; then
        echo "Clean lambda env..."
        lambda deprovision --lambda-uri "$LAMBDA_URI"
        clean_status=$?
        if [ "$clean_status" != "0" ]; then
            echo "Clean lambda env fail"
        else
            echo "Clean lambda env success"
        fi
    fi

    trap - HUP INT TERM QUIT ABRT EXIT
    exit ${exit_status}
}

trap on_exit HUP INT TERM QUIT ABRT EXIT

if [ -n "$BUILD_URI" ]; then
  echo "Launch lambda env..."
  LAMBDA_URI=$(lambda provision --build-uri "$BUILD_URI")
  echo "Launch lambda success"
  cd $CODEBASE
  LAMBDA_INFO=$(lambda info --lambda-uri $LAMBDA_URI)
  ENDPOINT_HOST=$(echo $LAMBDA_INFO|jq --raw-output '.services.main.endpoint.internal.host')
  ENDPOINT_PORT=$(echo $LAMBDA_INFO|jq --raw-output '.services.main.endpoint.internal.port')
  ENDPOINT=$ENDPOINT_HOST:$ENDPOINT_PORT
fi


if [ -z "$ENDPOINT" ]; then
  echo 'environment ENDPOINT is not given'
  exit 1
fi

echo
echo "Run verify ..."
curl http://$ENDPOINT
echo "Run verify complete"
echo
```

**注意** `verify.sh` 中一般都会包含两部分模板代码

1. 启动并获取临时环境。在 `cde` 中运行 `verify` 的时候会为其提供一个 `$BUILD_URI`，它包含了在 `build` 阶段创建的一个 `artifact`。然后利用 `lambda` 工具创建一个运行该应用的临时环境并获取改临时环境的入口
   ```bash
   if [ -n "$BUILD_URI" ]; then
     echo "Launch lambda env..."
     LAMBDA_URI=$(lambda provision --build-uri "$BUILD_URI")
     echo "Launch lambda success"
     cd $CODEBASE
     LAMBDA_INFO=$(lambda info --lambda-uri $LAMBDA_URI)
     ENDPOINT_HOST=$(echo $LAMBDA_INFO|jq --raw-output '.services.main.endpoint.internal.host')
     ENDPOINT_PORT=$(echo $LAMBDA_INFO|jq --raw-output '.services.main.endpoint.internal.port')
     ENDPOINT=$ENDPOINT_HOST:$ENDPOINT_PORT
   fi
   ```
2. `on_exit` 函数。用于在执行 `verify` 之后的资源清理。

在 `verify.sh` 通过 `curl $ENDPOINT` 判断应用的 `/` 路由是否有效，其中 `ENDPOINT` 在 cde PaaS 在创建 `lambda` 环境之后提供给 `verify`。而在我们本地测试时测试通过环境变量直接传递给 `verify`。

### 在本地环境测试 verify

构建 `flask-verify`

    docker build -t flask-verify .

测试 `flask-verify`

    docker run -e ENDPOINT=www.baidu.com flask-verify

## 将 docker image 推送到 docker registry

为了构建栈需要将栈所依赖的 `build image` 和 `verify image` 推送到可以被 Cde PaaS 访问到的 docker registry。

```bash
docker tag flask-build hub.deepi.cn/flask-build
docker push hub.deepi.cn/flask-build
docker tag flask-verify hub.deepi.cn/flask-verify
docker push hub.deepi.cn/flask-verify
```

## 创建 stackfile

在准备好了 `build image` 和 `verify image` 之后，我们就可以创建 stackfile 了，内容如下

`stackfile.yml`:

```yaml
name: "flask"
description: "A flask stack"
template:
  type: "git"
  uri: 'https://github.com/aisensiy/cde-flask-init-project.git'
tags:
    - "python"
languages:
  - name: "python"
frameworks:
  - name: "flask"
tools:
services:
  main:
    build:
      image: 'hub.deepi.cn/flask-build'
      mem: 512
      cpus: 0.5
    verify:
      image: 'hub.deepi.cn/flask-verify'
      mem: 512
      cpus: 0.5
    main: yes
    mem: 512
    cpus: 0.4
    instances: 1
    expose: 5000
    health:
      - protocol: "COMMAND"
        command: "exit 0"
        interval: "3"
        timeout: "2"
```

所定义的栈只有一个服务 `main`，并且所暴露的端口为 `flask` 的默认端口 `5000`，`stackfile` 的详细定义见上方 yaml 文件。

然后利用 `cde` 客户端提交栈。

```bash
cde login <controller-entry-point>
cde stacks:create stackfile.yml
```

**注意** 在创建栈成功之后，栈的状态为 `UNPUBLISHED` 这意味着这个栈还不能被其他用户所使用，只有栈的构建者可以利用这个栈创建应用，并且在栈为 `UNPUBLISHED` 时是可以随时通过 `cde stacks:update` 修改栈的。当栈构建者认为这个栈已经足够稳定后可以通过 `cde stacks:publish <stack-id>` 的命令发布栈，之后其他的用户就可以使用这个栈创建应用了。

## 在 cde 测试栈

在成功创建栈之后可以利用 `template` 项目在 Cde PaaS 中创建应用测试栈。

```bash
cd template
cde apps:create test-flask flask
git push cde master
```
