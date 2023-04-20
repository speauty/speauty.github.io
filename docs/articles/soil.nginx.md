# 从入门到入土之nginx 

> [nginx](http://nginx.org/en/) [engine x] is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server, originally written by Igor Sysoev.

?> 就以往的工作来说, 将nginx作为web服务器和反向代理较多, 至于邮件服务器等相对使用较少, 所以关于nginx, 大多在于配置的优化和原理的理解上面.

#### 1. 安装nginx
   * 源码编译
   ```shell script
wget http://nginx.org/download/nginx-1.19.6.tar.gz
tar -zxf nginx-1.19.6.tar.gz
cd nginx-1.19.6 && ./configure && sudo make && sudo make install
# 通用配置选项
# --prefix=<path> nginx安装的根路径, 所有其他的安装路径都要依赖于该选项
# --sbin-path=<path> 指定nginx二进制文件的路径, 默认值 prefix/sbin
# --conf-path=<path> 指定的配置文件, 默认值 prefix/conf
# --error-log-path=<path> 指定错误文件的路径, nginx将会🕸其中写入错误日志文件
# --pid-path=<path> 指定的文件将会写入nignx-master进程的pid, 通常在/var/run下
# --lock-path=<path> 共享存储器互斥锁文件的路径
# --user=<user> worker进程运行的用户
# --group=<group> worker进程运行的组
# --with-file-aio 为FreeBSD 4.3+和Linux 2.6.22+系统启用异步I/O
# --with-debug 用于启用调试日志. 在生产环境的系统中不推荐使用该选项
```