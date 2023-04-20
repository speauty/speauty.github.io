# MQ: 消息队列 {docsify-ignore}

## RabbitMQ

?> 当前环境
```
版本	Windows 10 专业版
版本号	21H2
安装日期	‎2021-‎05-‎12
操作系统内部版本	19044.1889
体验	Windows Feature Experience Pack 120.2212.4180.0

处理器	Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz   2.59 GHz
机带 RAM	16.0 GB (15.8 GB 可用)
系统类型	64 位操作系统, 基于 x64 的处理器
笔和触控	没有可用于此显示器的笔或触控输入
```

Windows二进制手动安装, 我之前还想用choco安装来着, 因为我记得我电脑上用过那玩意, 然后一试, 寄了. 只好单独下载对应二进制文件, 过程相对简单.

### 安装Erlang环境
[点击跳转到下载页面](https://www.erlang.org/downloads), 选择适合你的版本安装, 我这里下载的是[otp_win64_25.0.4.exe](https://github.com/erlang/otp/releases/download/OTP-25.0.4/otp_win64_25.0.4.exe),
稍微有点大, 下完之后, 直接双击二进制文件, 一路默认, 对了, 安装路径, 可能需要调整一下, 它默认是在C盘, 那肯定不行.
然后就是设置环境变量ERLANG_HOME, 指到Erlang的安装路径, 然后在环境变量Path中添加%ERLANG_HOME%\bin, 最后就是打开终端, 输入erl, 看下版本号.

### 安装RabbitMQ
所以就麻烦, Windows二进制安装需要依赖Erlang环境, 其他安装方式倒是不用, 没法. 还是一样, 点击下载[rabbitmq-server-3.10.7.exe](https://download.fastgit.org/rabbitmq/rabbitmq-server/releases/download/v3.10.7/rabbitmq-server-3.10.7.exe),
安装的时候, 改一下安装路径就成, 其他也不需要动, 最后还是添加环境变量RABBITMQ_SERVER, 直到RabbitMQ的安装路径, 我这里是E:\service\RabbitMQ\rabbitmq_server-3.10.7, 然后在环境变量Path中添加%RABBITMQ_SERVER%\sbin, 就完了.

### 配置用户和授权
蒙蔽, 我不知道为什么guest(guest)这个默认账号一直登录失败, 也没其他提示, 就只好新建用户了.
```shell
# 添加账号 add_user 账号 密码
rabbitmqctl add_user test test
# 授权 set_permissions -p / 账号
rabbitmqctl set_permissions -p / test "." "." ".\*"
# 添加角色
rabbitmqctl set_user_tags test administrator
# 如果出现"no access to this vhost" 错误
rabbitmqctl add_vhost new_vhost_name
# 然后再赋权
rabbitmqctl set_permissions -p 用户名 new_vhost_name "." "." ".*"
# 因为在代码中, 连接mq需要具体的vhost
# conn, err := amqp.Dial("amqp://sky:password@ip:5672/admin”)
```

### 启动服务
安装管理界面 `rabbitmq-plugins enable rabbitmq_management`, 似乎也启动了服务, 如果没启动的话, 再运行一下 `rabbitmq-server.bat` 即可. 如果出现权限问题, 就用以管理员身份运行终端.


