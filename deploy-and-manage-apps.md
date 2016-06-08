# 使用技术栈

## 安装客户端

```
cd $CDE/client
make deploy
cde -h
```

## 注册用户

首次使用需要注册用户，老用户可以直接登录，注册和登录命令如下：

```
cde register http://controller.tw.com --email=guest@tw.com --password=111111
cde login http://controller.tw.com --email=guest@tw.com --password=111111
```

## 上传 SSH-KEY

如果terminal中没有ssh key则，先生成ssh key

```
ssh-keygen
cde keys:add ~/.ssh/id_rsa.pub
```

## 创建 app

首先进入工程根目录 `cd dir`，然后通过cde创建 app

```
cde stacks:list
cde apps:create test javajersey
```

## 编写代码

在编辑器中完成代码编写之后，可以通过下面命令及时将代码push到PAAS上，过程的日志会打印在控制台，如果最终build success则可以跳转到下一步；如果build失败，则需要开发人员根据日志报错修复代码错误。

```
git push cde master
```

## 绑定访问路径

```
cde domains:create test.tw.com
cde routes:create test.tw.com demo
cde routes:bind test.tw.com/demo <appName>
```
**通过网址 http://test.tw.com/demo 访问应用**

## 查看部署的应用状态

```
cde apps:info
```