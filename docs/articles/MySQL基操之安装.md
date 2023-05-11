# MySQL基操之安装

确实基操。这应该是每个后端开发必会技能，不过现大多是集成环境或套件之类的安装，实际很少这样。毕竟，在某种程度上来说，也属于“浪费时间”操作。我这里，也主要是为了熟悉和巩固，无甚新意。

事先说明，我这里均采用通用二进制包安装。

> 目标版本: MySQL Community Server 5.7.42

### OS-Windows

当前环境：`Windows 10 专业版-21H2-19044.2846`

#### 下载

打开[下载网页](https://dev.mysql.com/downloads/mysql/)，然后点击`Looking for previous GA versions?`会跳转到`MySQL Community Server 5.7.42`（如果没有的话，就在`Archives`中找一下）。操作系统选择`Microsoft Windows`，系统版本选择`Windows(x86，64bit)`，下拉下载第一个压缩包（不带调试和测试套件），也就是[`mysql-5.7.42-winx64.zip`](https://dev.mysql.com/downloads/file/?id=518720)。

直接解压到安装目录，我这里是`D:\env\mysql\5.7`。就以下这些目录：

```shell
├─bin       # 可执行文档目录
├─docs      # 文档目录
├─include
├─lib
└─share
```

#### 初始化和安装

1. 初始化

直接执行`bin\mysqld --initialize --console`，在回显中，注意，`[Note] A temporary password is generated for root@localhost: h9ex<fy2fq4W`，`root`账户的临时密码。

2. 注册服务

这里需要注意要用`以管理员身份运行`打开一个终端窗口，然后输入`bin\mysqld install mysql5.7`，指定服务名称为`mysql5.7`。

3. 运行服务

通过指令`sc|net start mysql5.7`就可以启动服务，`mysql5.7`是这里注册的服务名。

4. 卸载服务

首先，需要关闭服务 `sc|net stop mysql5.7`，然后通过执行`sc delete mysql5.7`或`bin\mysqld --remove`就可以卸载服务了。

### 其他配置

#### 配置环境变量-Windows

`Win`，打开`设置`，找到`系统`，最下面的`关于`，然后`高级系统设置`中`环境变量`，在`用户变量`中新建一项`变量名: MySQL5.7 变量值: D:\env\mysql\5.7`，然后双击`Path`变量，新建`%MySQL5.7%\bin`。到这里就算搞定。注意：必须新开终端才会生效。

#### 用户管理
    
* 重置密码：`alter user user() identified by "passwd"`；

* 添加用户：`create user "test"@"127.0.0.1" identified by "test"`；
    * 可以通过指令`mysql -utest -ptest -h 127.0.0.1`连接，必须要指定主机（主机非`*`的情况下），否则验证失败；

* 更新用户：`update mysql.user set authentication_string=password("123456") where User="test" and Host="localhost"`；

* 删除用户：常规操作；

* 授权操作：`grant all privileges on dbname.table to "username"@"ip addr" identified by "passwd"`；
    * 除了用户名称和密码均支持`*`，简单来说，就是无限制；

* 刷新权限：`flush privileges`，在进行授权操作之后，必执行的一个操作；