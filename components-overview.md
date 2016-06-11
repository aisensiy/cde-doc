## 核心组件

### Controller
Controller 是CDE中的控制REST API服务。用来控制整个build流程的一个rest服务。

### Builder
Builder 是作为用户的一个中心的git repo的角色存在，当用户提交代码的时候，触发相应的githook来启动相应的build流程。通过调用controller来管理build流程

### Launcher
Launcher 是paas内部的启动模块。当出现以下两个场景的时候回调用launcher

1. 当有application发生release的时候，触发部署流程。
2. 当verify需要一个lambda测试环境的时候，触发lambda环境搭建

### Deployer
Deployer 当有release 产生的时候，调用launcher进行相应的部署

### Discovery
Discovery 是CDE中的服务发现模块。当有新的application被运行在paas上的时候，discover将这些运行的状态信息保存在consul里面用来做相应的服务发现

### Monitor
Monitor 是CDE中的监控模块，主要用来整个paas集群中机器的使用情况，以及整个paas中日志的监控

### Receptor
Receptor作为整个paas中的接入模块，负责整个paas中的流量的路由，已经负载均衡的功能

### Registry
Registry用来存储在build和verfiy中产生的image，以及存储相应的release image。当发生部署的时候, launcher从这里拉取相应的image进行相应的部署

### Registry-mirror
Registry-mirror 是一个docker官方registry的一个镜像服务。集群内部的机器会由于任务的不同，会去拉取官方网站上的一些docker image，通过镜像服务，所有的镜像只需要拉取一次，减少网络的使用。加速启动流程

### Releaser
Releaser 负责当有application push成功的时候，或者当application的环境变量发生变化的时候，需要对application进行一次的release