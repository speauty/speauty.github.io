# Jenkins配置 {docsify-ignore}

### 构建服务
#### Docker构建
1. 拉取镜像 `docker pull jenkins/jenkins`
2. 创建容器 `docker run -d -u root -p 10240:8080 -p 10241:50000 --name jenkins --network dev --network-alias jenkins jenkins/jenkins:latest`
3. 查看密码 `docker logs jenkins`
    ```text
    Jenkins initial setup is required. An admin user has been created and a password generated.
    Please use the following password to proceed to installation:
    
    82f3c048d704402c86f90f384e9e3d9a
    
    This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
    ```
4. 默认推荐配置安装即可，不过由于网络原因，会导致大部分插件下载失败。这里需要改个配置，在 `系统管理 => 插件管理 => 高级 => 升级站点` 中，输入 `http://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json` 即可，重新选择相应插件更新重启即可。


### 插件-角色权限
内置的权限认证，好像都不是想要的。我当前的一个实际场景就是，根据不同用户所在组，显示不同的视图或任务之类的。
经过一番搜索，发现 `Role-based Authorization Strategy` 这款插件好像是我想要，折腾出来了，效果也不错，所以这里整理一下。
#### 安装插件
在 `系统管理` => `插件管理` => `可选插件` 中输入 `Role-based Authorization Strategy`，搜索安装即可。
#### 设置插件
安装完成后，可在 `系统管理` => `Manage and Assign Roles` 打开这个插件，主要就是管理和分配角色。角色主要分为三大类：全局(global)，项目(item)以及节点(node)，
其中节点角色是针对Jenkins集群使用的，所以这里暂且不讨论。全局角色的话，这里只需要一个 `全部 Read` 的角色即可。至于项目角色的话，则根据具体项目匹配划分，可采用正则匹配模式，所以一般项目角色的权限会全部勾上。

其中，全局角色勾选的权限会覆盖项目角色对应的权限。接下来，就是分配角色，将不同用户或组分配到指定的角色下，一般是需要一个全局可读的角色，以及随即项目角色。不过我这里的项目角色，只配置了任务的 `Build Cancel Read Workspace`这几项权限。

### 插件-DingTalk
一款构建通知钉钉群的插件，主要通过配置钉钉机器人实现，在 `系统管理 => 插件管理 => 可选插件` 中搜索 `DingTalk` 即可下载安装。重启之后，可在 `系统管理 => 系统配置` 中，看到钉钉相关配置。点击新增可添加机器人配置。

至于机器人配置中，id只要不与其他机器人重复，名称随意，webhook填钉钉群机器人那边的地址，并填写对应的关键字。

打开钉钉，点击群设置，然后是智能群助手，添加自定义机器人即可，设置关键字。

### 相关问题
#### 相关命令未找到
主要是由于未加载`/etc/profile`，导致无法使用环境变量，需要在任务构建脚本顶部添加如下内容：
```shell
#!/bin/bash -ilex
source /etc/profile
```
#### 容器时区以及构建时间问题
容器默认是UTC，所以需要调整下时区，在容器内部执行 `ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime` 即可。而构建时间，需要在 `系统管理` => `脚本命令行` 中执行 `System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')
` 即可。