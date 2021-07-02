---
title: GitLab + Drone
layout: article
tags: CICD
mode: immersive
lang: zh-Hans
outhor: Boom Young
pageview: true
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---

> 写在前面：
>
> 项目了使用公司的私有镜像仓库和GitLab以及已经搭建好的Drone 0.8，所以部署安装这部分跳过，本文着重讲解drone的配置部分。
>
> 另外Drone 0.8版本和1.0及以上在语法上有较大的差异，使用新版本的请参考官方文档



# 一、基本概念

### 什么是DRONE？



Drone是一个基于Docker容器技术的可扩展的持续集成引擎，用于自动化测试与构建，甚至发布。每个构建都在一个临时的Docker容器中执行，使开发人员能够完全控制其构建环境并保证隔离。 开发者只需在项目中包含 .drone.yml 文件，将代码推送到 git 仓库，Drone 就能够自动化的进行编译、测试、发布。

### DRONE基本原理解析



Drone的部署分为Server（drone-server）和Agent（drone-agent），Server端负责后台管理界面以及调度，Agent则负责具体的任务执行。比如配置文件 .drone.yml 里的 parsing 发生在 drone-server端，而具体的步骤则在 drone-agent端里执行。也就意味着environment/build_args关键字是不影响 配置文件.drone.yml 本身环境变量的解析替换。

目前Drone 支持多种代码托管服务，几乎涵盖市面上主流的代码仓库，如Github, GitLab, Gogs, Gitea、Bitbucket Server等，且在drone-server 里预设了对应托管服务的 API，Drone 的很多功能比如拉取 git repo list/add webhook to repo 都是通过这些 API 完成的。另外 Drone的账户体系依赖于托管服务的账户系统， 并不存在维护账户这个概念。例如我们使用GitLab作为代码托管服务，我们在登录 Drone 时，实际上是 Drone 把用户名密码传给了 GitLab. 因此，激活某个 Repository （仓库） 的构建(为Repository 添加webhook) 能否成功取决于该账号在 GitLab 里是不是该 Repository 的管理员。

### 什么是Webhooks



Drone调用代码仓库的 API 给 Repository 增加一个 webhook ，当 Repository 触发相应事件(push, tag, pull request)时，代码仓库发起 http 请求回调 drone 触发构建。

Drone 通过 OAuth 认证或账号密码登录代码仓库后，获得完整的控制权。在 Drone 的 web后台管理页面激活 Repository 后，Drone调用代码仓库的 API 给 Repository 增加一个 webhook ，当 Repository 触发相应事件(push, tag, pull request)时，代码仓库发起 http 请求回调 drone 触发构建。

Webhooks 由代码仓库发送，用于触发 pipeline。代码仓库会在下面 3 种情况下，自动发送 Webhook 请求到 Drone：

- 代码被 push 到Repository
- 新建一个合并请求（pull request）
- 新建一个tag



# 二、开始DRONE



下面以一个最小的Flask项目为例，讲解drone的流程及配置

http://drone.iflytek.com/boyang6/drone_test

https://hub.iflytek.com/harbor/projects/621/repositories/drone-test%2Fdrone-test-app
  
https://git.iflytek.com/boyang6/drone_test



### 打通GitLab和Drone

#### Drone

1. 在drone的前端页面Repositories配置里勾选你的项目，若不存在则点击Synchronize刷新；
2. 设置Secret，`docker_username`、`docker_password`分别保存hub仓库的账号密码，`ssh_password`保存远程主机的登录密码；
3. 设置settings，Repository Hooks 只勾选tag，表示只有当代码提交tag的时候才会触发钩子；
4. 复制Token里的Personal Token，备用。

![image.png](\assets\gitlab1.png)

#### GitLab

1. 在项目设置里选择集成，使用现有集成Drone CI，配置如下：

![image.png](\assets\drone1.png)

Token使用上面Drone复制的Token，填写Drone url，测试无误后保存修改

#### HUB

1. 创建镜像仓库，用于存储构建成功后的镜像



### 编写 .drone.yml

.drone.yml 文件放在项目的根目录下，与Dockerfile同级

```yaml
# 定义矩阵变量，通过 ${VARIABLE} 方式引用变量
matrix: 
  IMAGE_REPO:
    - hub.iflytek.com/drone-test/drone-test-app # 定义了一个镜像存储路径的全局变量
  RESGISTRY:
    - hub.iflytek.com # 定义镜像仓库地址
  REPORT_EMAIL:
    - boyang6@iflytek.com # CI/CD报告发送的邮件地址

# 手动配置克隆步骤，如果没有定义具体的克隆步骤，Drone 会自动配置
clone:
  git:
    image: plugins/git
    depth: 50 # 当depth为1时，只克隆当前版本，不克隆当前版本以外的提交记录，如果该参数未指定，默认所有版本提交都克隆
    tags: true

# drone流水线描述
pipeline:

  # 构建和发布镜像
  build:
    image: plugins/docker # 使用 Docker 插件来构建和发布镜像
    registry: ${RESGISTRY}
    repo: ${IMAGE_REPO}
    secrets: [docker_username, docker_password] # 引用Drone 的 web 页面添加的secrets进行认证
    tags: # 同时构建两个镜像，标签为latest方便部署
      - ${DRONE_TAG=latest}
      - latest
    dockerfile: Dockerfile
    insecure: true # 启用对此 registry 的不安全通信
    when: # 触发条件
      event: [ tag ]
      
  # 部署
  deploy_staging:
    image: appleboy/drone-ssh # 使用 drone-ssh 插件对远程服务器操作
    host:
      - 172.16.59.204
    username: root
    port: 22
    secrets: [ ssh_password ] # 引用Drone 的 web 页面添加的secrets进行认证
    command_timeout: 3m # 超时时间，必须带单位
    script:
      - docker pull ${IMAGE_REPO}:latest
      - docker rm -f drone-test-app || true # 这里这样是因为如果不存在docker-demo，rm会报错
      - docker run -itd -p 9080:9080 --name drone-test-app ${IMAGE_REPO}:latest
    when:
      event: [ tag ]
```

下面有更详细的注解：

- image：使用插件，指定当前步骤将在该容器内执行
- registry：向这个 registry 进行验证
- username：使用此用户名进行身份验证
- password：使用此密码进行身份验证
- repo：用于存储镜像的仓库名
- tags：用于镜像的仓库的 tag
- dockerfile：要使用的 dockerfile，默认是 `Dockerfile`
- auth：registry 的身份验证 token
- context：要使用的上下文路径，默认为 git 仓库的根目录
- target：要使用的构建目标，必须在 dockerfile 中定义
- force_tag=false：替换现有的匹配到的镜像的 tag
- insecure=false：启用对此 registry 的不安全通信
- mirror：使用 registry 镜像，而不是直接从 Docker 默认的 Hub 中获取镜像
- bip=false：用于传递 bridge IP
- custom_dns：为容器设置自定义 DNS 服务器
- storage_driver：支持 aufs，overlay 或 vfs 驱动程序
- build_args：自定义参数传递给 docker build
- auto_tag=false：根据 git 分支和 git 标签自动生成标签名称
- auto_tag_suffix：用这个后缀生成标签名称
- debug, launch_debug：以详细调试模式启动 docker 守护进程