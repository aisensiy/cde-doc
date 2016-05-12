# Cde Paas

Cde 是一个全功能的 `PaaS`，它管理一个 web 应用的完整的生命周期。在 Cde 创建应用只需要简单的设置应用所需的栈(stack)并通过 `git` push 代码.

Cde 可以构建任何语言、任何框架的应用。整个构建的过程都过 `dockerize` 的方式进行，一个应用通过 `git push` 提交给 Cde 后经过如下的步骤将应用部署在 Cde 中。

> Codebase ---build-->　Runnable App ---verify---> Release ---deploy---> Running App

## 栈管理

## 主要功能

1. **应用构建自动化**
2. **自动负载均衡**
3. **服务发现**
4. **应用编排**
5. **内置 CI**
6. **分布式存储**
