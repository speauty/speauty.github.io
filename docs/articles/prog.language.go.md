# Golang {docsify-ignore}

### 环境构建
?> [点击打开下载页面](https://golang.google.cn/dl/)，这里按OS区分
#### Windows系统
直接下载相应安装，双击安装即可。我这里使用的是[go1.18.1.windows-amd64.msi](https://golang.google.cn/dl/go1.18.1.windows-amd64.msi)，
由于之前装有，这里就不单独安装。需要注意的是，我主机安装的版本是 `go1.16.2`。其实也没什么需要注意的，相对比较简单。

可在终端输入 `go version` 查看当前版本。当然，有可能提示错误

!> go: The term 'go' is not recognized as a name of a cmdlet, function, script file, or executable program.
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.

这表示 `go.exe` 没有在环境变量(这里指的是Path)中，需要在 `Win => 设置 => 系统 => 关于 => 高级系统设置 => 环境变量 => 系统变量(Path) => 新增`，
填入 `go.exe` 所在的bin目录
#### Linux系统
以 `Ubuntu-22.04` 系统为例，这里采用[go1.18.1.linux-amd64.tar.gz](https://golang.google.cn/dl/go1.18.1.linux-amd64.tar.gz)
1. 下载对应压缩文件 `wget https://golang.google.cn/dl/go1.18.1.linux-amd64.tar.gz`
2. 解压到指定安装目录 `tar -wzxf go1.18.1.linux-amd64.tar.gz -C /usr/local`
3. 配置环境变量，然后通过指令 `source /etc/profile` 重载配置即可。不过我当时好像是重启之后才生效的。
   ```shell
   sudo vim /etc/profile
   # 在文件末尾增加下列配置
   export GOPATH=$HOME/go
   export GOROOT=/usr/local/go
   export GOOS=linux
   export GOTOOLS=$GOROOT/pkg/tool/
   export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
   ```
4. 校验环境，在终端输入指令 `go version`，即可输出当前版本。

实际上，在 `Linux` 环境，各方面实现相对比较全面，建议采用 `Linux` 环境开发。若在 `Windows` 上，推荐使用 `WSL` 或 `Docker` 相关环境。

### gRPC服务

#### 配置环境
* 安装编译器(protoc)

    点击[protoc](https://github.com/protocolbuffers/protobuf/releases/)进入下载页面，我这里采用的是[protoc-3.20.1-rc-1-win64.zip](https://github.com/protocolbuffers/protobuf/releases/download/v3.20.1-rc1/protoc-3.20.1-rc-1-win64.zip)
    解压之后，将其中 `bin\protoc.exe` 拷贝到 `GOPATH\bin` 下，我本地是 `E:\MonkeyCode\space.go\bin`，可在终端输入`protoc --version`查看版本 `libprotoc 3.20.1-rc1`

* 安装插件(protoc-gen-go和protoc-gen-go-grpc)
    
    * 终端运行 `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`，这个需要Golang的版本不低于1.16，运行完之后，就可以在 `GOPATH\bin` 下看到 `protoc-gen-go.exe`

    * 终端运行 `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest`，这个需要Golang的版本不低于1.16，运行完之后，就可以在 `GOPATH\bin` 下看到 `protoc-gen-go-grpc.exe`

至此，环境算是配置好了，接下来，看一个简单例子吧

#### 示例Hello
  
* 定义`protoc\hello.proto`
  ```protobuf
  syntax = "proto3";

  // .代表在当前目录生成 hello表示生成的包名
  option go_package = ".;hello";
  
  message HelloResponse {
    string responseSomething = 1;
  }
  
  message HelloRequest {
    string saySomething = 1;
  }
  
  service HelloService {
    rpc Say(HelloRequest) returns (HelloResponse) {}
  }
  ```
  
  然后分别执行 `protoc --go_out=. hello.proto` 和 `protoc --go-grpc_out=. hello.proto`，生成 `hello.pb.go` 和 `hello_grpc.pb.go` 两个文件。
  如果出现有依赖未找到的，可执行 `go mod tidy`。

* 服务端`server\server.go`
  ```go
  package main

  import (
    "context"
    world "go.test/hello/proto"
    "net"
    "google.golang.org/grpc"
    "log"
  )
  
  type HelloWorld struct {
    world.UnimplementedWorldServer
    h *world.WorldServer
  }
  
  func (h *HelloWorld) MustEmbedUnimplementedWorldServer() {
    panic("implement me")
  }
  
  func (h *HelloWorld) Say(ctx context.Context, req *world.WorldRequest) (*world.WorldResponse, error) {
    log.Println("receive message:", req.GetSaySomething())
    resp := &world.WorldResponse{}
    resp.responseSomething = "roger that!"
    return resp, nil
  }
  
  func main() {
    srv := grpc.NewServer()
    world.RegisterWorldServer(srv, &HelloWorld{})
  
    listener, err := net.Listen("tcp", ":12345")
    if err != nil {
      log.Fatalf("failed to listen: %v", err)
    }
  
    err = srv.Serve(listener)
    if err != nil {
      log.Fatalf("failed to serve: %v", err)
    }
  }
  ```
  
* 客户端`client\client.go`
  ```go
  package main

  import (
    "context"
	world "go.test/hello/proto"
    "google.golang.org/grpc"
    "log"
  )
  
  func main() {
    conn, err := grpc.Dial("127.0.0.1:12345", grpc.WithInsecure(), grpc.WithBlock())
    if err != nil {
      log.Fatalf("did not connect: %v", err)
    }
    defer func(conn *grpc.ClientConn) {
      _ = conn.Close()
    }(conn)
  
    client := world.NewWorldClient(conn)
	_, err = client.Say(context.Background(), &world.WorldRequest{saySomething: "world"})
	if err != nil {
      log.Fatalf("连接失败: %v", err)
	}
  }
  ```
  
* 参考链接
  * [Protocol Buffer Basics: Go](https://developers.google.cn/protocol-buffers/docs/gotutorial) 
  * [gRPC-go 入门（1）：Hello World](https://blog.csdn.net/inet_ygssoftware/article/details/117608527)
  * [golang Grpc 中出现 it has a non-exported method and is defined in a different package](https://www.jianshu.com/p/d2c8fdd24b0f)

### 其他操作

#### 增加ico
1. 在项目根目录添加同名rc文件(exe.rc);
   ```
   IDI_ICON1 ICON "exe.ico"
   ```
2. 在项目根目录执行 `windres -o exe.syso exe.rc`;
3. 执行 `go build` 就可以生成带图标的exe文件

### 其他问题
#### fyne-cross设置网络代理
比较烦，如果没有梯子的话，直接打包的话，就会网络失败，这时候想到可能要设置代理，由于我这里采用的是镜像打包，直接设置环境参数即可，比如 `-env=GOPROXY="https://goproxy.cn"`

#### gorm零值更新
如果不特殊处理，在gorm中，零值是直接被忽略。如果需要更新零值的字段，可以通过 `Select` 指定更新字段，或者更新数据采用 `map[string]interface{}`，文档中有相关说明。

### 相关站点
* 暂无

### 附录
  * [在Windows上管理go项目脚本](/resources/win.go.pro.manage.bat ':ignore')
