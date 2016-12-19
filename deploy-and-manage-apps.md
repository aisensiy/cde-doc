# 使用技术栈

## 安装客户端

在 [https://github.com/tw-cde/cde-client-binary/releases](https://github.com/tw-cde/cde-client-binary/releases) 下载并安装最新的客户端

## 注册用户

首次使用需要注册用户，老用户可以直接登录，注册和登录命令如下：

```
cde register http://controller.xxx.com --email=guest@tw.com --password=111111
cde login http://controller.xxx.com --email=guest@tw.com --password=111111
```

## 上传 SSH-KEY

如果 terminal 中没有 ssh key 先生成ssh key

```
ssh-keygen
cde keys:add ~/.ssh/id_rsa.pub
```

采用 sss-agent

```
eval `ssh-agent -s`
ssh-add ~/.ssh/id_rsa
```

## 创建 app

首先进入项目根目录，然后通过 cde 创建 app

```
cde stacks:list
cde apps:create test-flask flask
```

其中 `test-flask` 为所要创建的项目名称，`flask` 是所选用的技术栈

## 使用本地开发环境

cde 提供 `dev` 命令用于创建本地环境，通过如下命令在本地构建开发环境

```
cde dev:up
```

`cde dev` 采用 `docker-compose` 构建一个本地开发环境，在当前项目目录下 `.local` 目录中包含所使用的 `dockercompose.yml`

## 编写代码

在编辑器中完成代码编写之后，可以通过下面命令及时将代码 push 到 PaaS上，过程的日志会打印在控制台，如果最终 build success 则可以跳转到下一步；如果 build 失败，则需要开发人员根据日志报错修复代码错误。

```
git push cde master
```

当项目部署成功后会提供一个默认的域名用于应用的访问。并且通过客户端可以为应该应用绑定其他的域名。

## 绑定访问路径

```
cde domains:create test.xxx.com
cde routes:create test.xxx.com /
cde routes:bind test.xxx.com/ test-flask
```

首先创建一个域名 `test.xxx.com` 然后创建一个可以访问的路径（通常会采用根路径），最后将我们的应用与新创建的路径绑定即可。

**通过网址 http://test.tw.com/demo 访问应用**

## 查看部署的应用状态

```
cde apps:info
```

## 采用技术栈构建 docker image

`cde` 的 `build` 过程本质上是采用一个 `build image` 将代码构建为了一个可以运行的 `service image`，这个 `service image` 本质上是可以运行在任何支持 `docker` 的地方的。如需将项目部署到 `cde` 意外的地方可以通过在本地构建的方式获取 `service image`：

每个 stack 都会对应一个 `build image`，例如 `jersey-mysql` stack 的 `build image` 为 `hub.deepi.cn/jersey-mysql-build`。每个 stack 的 `build image` 可以在 [https://github.com/tw-cde/cde-stacks](https://github.com/tw-cde/cde-stacks) 中每个技术栈目录下的 `stackfile.yml` 中查看。

参考 [创建 python flask stack](./create-stack-for-python) 以及 [创建 jersey mysql stack](./create-stack-for-jersey-mysql.md) 在本地构建 `service image`。通过命令

```
docker run \
  -v $PWD/template:/codebase \
  -e CODEBASE=/codebase \
  -e IMAGE=jersey-mysql-service \
  -v /var/run/docker.sock:/var/run/docker.sock \
  hub.deepi.cn/jersey-mysql-build
```

构建 `service image`: jersey-mysql-service
