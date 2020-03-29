---
title: Go 实现 gRPC 服务
date: 2020-03-29 16:50:13
tags:
- Go
- gRPC
categories:
- Go
---

# Go 实现 gRPC 服务
## 介绍

最近学习了一下 gRPC 在 Go 中的实现，做一些记录。

### gRPC 是什么

讲 gRPC 之前，可以先讲一下 RPC(Remote Procedure Call) 远程过程调用，它能让客户端直接类似于调用本地方法一样调用服务端的方法。gRPC 是 Google 开源的一款高性能 RPC 框架。

gRPC 中主要有四种请求/响应模式：

* 普通 RPC
* 服务端流式 RPC
* 客户端流式 RPC
* 服务端/客户端双向流式 RPC

### gRPC 好处

RPC 一般是建立在 TCP 或者 HTTP 网络连接之上的，是一个框架，所以对于开发者而言，如果使用 RPC 去实现服务端与客户端的通信，基本可以不考虑网络协议、连接等内容，专注与处理逻辑。

使用 gRPC 的话，可以将我们的服务定义在 .proto 文件中，然后在任何一种支持 gRPC 的语言中实现客户端和服务端，这可以让服务端和客户端运行在不同的环境中，另外使用 gRPC 还有其他的好处：高效序列化与反序列化、简单的 IDL 语言、方便接口更新

## 基础使用

已官方的 helloworld 作为例子进行讲解，[示例](https://github.com/grpc/grpc-go/tree/master/examples/helloworld)

### 准备工作

* Mac 安装 protoc 编译器

```
$ brew install protobuf

```

* 安装 protoc 编译器插件

```
$ go get -u github.com/golang/protobuf/protoc-gen-go

```

### 定义服务

gRPC 是通过 Protocol Buffers 去定义 gRPC 相关的 service 服务以及请求和响应相关的类型，如果对 Protocol Buffer 不熟悉的，其实也没什么关系，直接看 protoc 文件也是比较容易看懂的。如果对 Protocol Buffer 想深入了解的 ，可以参考这个链接:  [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview)


创建 helloworld.proto 文件，并填写以下内容

```protoc
syntax = "proto3";
package protos;

// The greeting service definition.
service Greeter {
	// Sends a greeting
	rpc SayHello (HelloRequest) returns (HelloReply) {}
}


// The request message containing the user's name.
message HelloRequest {
	string name = 1;
}

// The response message containing the greetings
message HelloReply {
	string message = 1;
}

```

来解释一下上面的代码：

* 第一行 `syntax = "proto3"` 表示目前使用的是 proto3 的语法
* 通过 service 关键字定义了一个 Greeter 服务，且这个服务目前只有 SayHello 一个方法，SayHello 这个方法会接收一个 HelloRequest 类型的消息，并返回一个 HelloReply 类型的消息
* 通过 message 关键字定义了 HelloRequest 类型消息的结构是什么样的，上述代码中，HelloRequest 消息结构只有一个字段，是 string 类型
* 通过 message 关键字定义了 HelloReply 类型消息的结构是什么样的，上述代码中，HelloReply 消息结构只有一个字段，是 string 类型
* 定义消息结构字段的时候，需要指定字段的字段编号，比如 `string name = 1` 中，1 就是字段编号，这个字段编号是一个唯一的数字，作用是在二进制消息体中表示字段用的
* 定义消息结构字段时，需要制定更多数据类型的话，可以参考链接：[Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)

在 proto 文件中完成定义 gRPC 的服务方法和消息之后，我们就可以来生成 gRPC 通信中服务端与客户端的接口了，这个时候就需要我们之前安装的 protoc 以及 protoc-gen-go 工具了

```
$ protoc --go_out=plugins=grpc:. helloworld.proto

--go_out 表示指定最终生成文件的输出路径，. 表示生成在当前路径下

## 注意这边如果使用 protoc --go_out=grpc:. hellororld.proto 命令生成的结果和指定了 plugins 生成的结果有不同，需要加上 plugins 去生成
```

命令执行完成之后，会生成 helloworld.pd.go，具体的内容因为篇幅原因，不完全展示了，可以参考：[helloworld.pd.go](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/helloworld/helloworld.pb.go)

会根据 proto 文件中定义的服务方法和消息生成对应的代码，比如：

* 是消息的话会对应生成
 * 对应的 struct 结构体
 * Reset()/String()/ProtoMessage()/Descriptor()/XXX_Unmarshal()/XXX_Marshal()/XXX_Merge()/XXX_Merge()/XXX_DiscardUnknown() 方法
 * 消息中定义字段的 get 方法
* 是服务的话会对应生成
 * 生成服务对应的 server 以及 client 接口
 * 接口对应的方法

### 编写服务端和客户端代码

gRPC 服务方法和消息定义完成，对应的服务端和客户端接口也定义完成之后，就需要来写服务端和客户端的具体实现代码了

#### 服务端代码

这部分代码也是 grpc-go 官方 helloworld example 例子中的代码，借这个简单的代码理解一下服务端代码的编写

```go
package main

import (
	"context"
	"log"
	"net"
	pb "github.com/Win-Man/sg-server/protos"
	"google.golang.org/grpc"
)

const (
	port = ":50051"
)

// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
	// 创建一个 gRPC 服务端实例
	s := grpc.NewServer()
	// 注册
	pb.RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

解释一下上面的代码

* 首先定义了一个 server 的结构体，这个没有指定字段，直接继承了生成的 helloworld.pb.go 中定义的 pb.UnimplementedGreeterServer 结构体
* 实现了 server 结构体的 SayHello 方法，SayHello 方法接受两个参数（ctx context.Context,in *pb.HelloRequest）返回两个参数(*pb.HelloReply,error),HelloRequest 和 HelloReply 其实就是 proto 中定义的消息对象，SayHello 方法的具体实现就看具体需求了，在例子中的话，就是获取了发送过来的请求中的 name 字段，将获取到的 name 值拼接上 Hello 以 Message 返回给客户端
* 在 main 函数中定义了服务端的启动和监听过程

#### 客户端代码

这部分代码也是 grpc-go 官方 helloworld example 例子中的代码，借这个简单的代码理解一下客户端代码的编写

```go
package main

import (
	"context"
	"log"
	"os"
	"time"
	pb "github.com/Win-Man/sg-server/protos"
	"google.golang.org/grpc"
)

const (
	address = "localhost:50051"
	defaultName = "sg"
)



func main() {
	// 建立连接
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
 	}

	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	// 获取命令行中传入的第一个参数作为 name 值，否则name就使用默认值
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	// 设置超时时间
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}

```

解释一下上面的代码：

* 所有的逻辑都是在 main 函数中
* ` conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())` 这个是在建立客户端与服务端之间的 grpc 连接
*  `c := pb.NewGreeterClient(conn)` 通过连接初始化客户端
* ` r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})` 客户端通过调用 gRPC 方法与服务端通信，就收服务端返回的信息

### 运行演示

* 运行服务端代码

```
$ go run server.go
2020/03/29 00:19:48 Received: sg
2020/03/29 00:22:11 Received: hhhh
```

* 运行客户端代码

```
$ go run client.go
2020/03/29 00:19:48 Greeting: Hello sg

$ go run client.go hhhh
2020/03/29 00:22:11 Greeting: Hello hhhh
```

客户端成功通过 gRPC 调用了服务端。

## 进阶使用

简单使用中已经介绍了简单 RPC 的使用方式，在这一节中，会补充讲解 gRPC 其他几种请求/响应模式，标准步骤都是：

1. 在 .proto 文件中定义 RPC 服务方法和消息
2. 根据 .proto 文件生成 pb.go 文件
3. 实现服务端代码
4. 实现客户端代码

### 服务端流式 RPC

服务端流式 RPC 顾名思义就是服务端会按照多次返回响应，直到服务端认为响应发送完毕，会告诉客户端响应应发送完毕

* 首先在 .proto 文件中定义服务端流式 RPC 的服务方法

```go
service Greeter {
......
	rpc TellMeSomething(HelloRequest) returns (stream Something){}
......
}



// The request message containing the user's name.
message HelloRequest {
	string name = 1;
}

......
message Something{
	int64 lineCode = 1;
	string line = 2;
}
```

在原先定义的 Greeter 服务的基础上添加了一个 TellMeSomething 的 RPC 方法，这个 RPC 方法接收的是请求结构是 HelloRequest 类型，返回的结构是 Something 类型，且用了 stream 定义，表示返回结构是一个流，Something 类型包含一个 int64 类型的 lineCode 字段和以一个 string 类型的 line 字段。

* 根据 .proto 文件生成 pb.go 文件，因为是自动生成的，这边就不展示完全的 pb.go 文件内容
* 编写服务端代码，在服务端实现 TellMeSomething 方法

```go
func (s *server)TellMeSomething(in *pb.HelloRequest, stream pb.Greeter_TellMeSomethingServer) error{
	log.Printf("Received ServerStream request from: %v", in.GetName())
	// 获取客户端发送的请求内容
	hellostr := fmt.Sprintf("Hello,%s",in.GetName())
	something := []string{hellostr,"ServerLine1","ServerLine2","ServerLine3"}
	for i,v := range(something){
		if err := stream.Send(&pb.Something{LineCode:int64(i),Line:v});err != nil{
			return err
		}
	}
	return nil
}
```

TellMeSomething 方法接受两个参数，一个是 *pb.HelloRequest 类型的参数，表示客户端发送的请求，另一个是 pb.Greeter_TellMeSomethingServer 类型的参数，这个参数其实就是服务端返回给客户端的流对象接口，生成的 .pb.go 文件中已经帮我们定义好了这个接口，包含一个 Send() 方法，用于发送响应给客户端，还有就是继承成 grpc.ServerStream 的流对象

```go
type Greeter_TellMeSomethingServer interface {
	Send(*Something) error
	grpc.ServerStream
}
```

在服务端通过  pb.Greeter_TellMeSomethingServer 对象的 Send() 方法给客户端发送响应，如果发送所有响应结束，则通过 return nil 的方式通知客户端响应发送结束，客户端会接收到 io.EOF 信号知道服务端已经发送完毕。如果发送中间过程有错误，则通过 return err 的方式将对应的错误通知给客户端

* 编写客户端代码，在客户端调用 TellMeSomethinng 方法

```go
func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	clientFunc := "Default"
	if len(os.Args) == 3 {
		name = os.Args[1]
		clientFunc = os.Args[2]
	}else{
		log.Fatal("Please input 2 arguments")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	switch clientFunc {
	......
	case "ServerStream":
		stream,err := c.TellMeSomething(ctx,&pb.HelloRequest{Name:name})
		if err != nil{
			log.Fatalf("TellMeSomething error: %v ",err)
		}
		for {
			something,err := stream.Recv()
			// 服务端信息发送完成，退出
			if err == io.EOF{
				break
			}
			if err != nil{
			log.Fatalf("TellMeSomething stream error:%v",err)
			}
		log.Printf("Recevie from server:{LineCode:%v Line:%s}\n",something.GetLineCode(),something.GetLine())
		}
......
}
```

客户端调用 TellMeSomething 方法，发送了一个 HelloRequest 类型的请求，获得一个 stream 返回对象，通过调用 Recv() 方法客户端接收服务端发送的一次次响应内容

### 客户端流式 RPC

客户端流式 RPC 就是客户端不断发送请求，直到发送完毕之后通知服务端请求发送完毕

* 首先在 .proto 文件中定义服务端流式 RPC 的服务方法

```go
service Greeter {
......
	rpc TellYouSomething(stream Something) returns(HelloReply){}
......
}
......
// The response message containing the greetings
message HelloReply {
	string message = 1;
}

message Something{
	int64 lineCode = 1;
	string line = 2;
}
```

在原先定义的 Greeter 服务的基础上添加了一个TellYouSomething 的 RPC 方法，这个 RPC 方法接收的是请求结构是 Something 类型，且用了 stream 定义，表示接受的请求是一个流，Something 类型包含一个 int64 类型的 lineCode 字段和以一个 string 类型的 line 字段，服务端返回给客户端一个 HelloReply 类型的响应

* 根据 .proto 文件生成 pb.go 文件，因为是自动生成的，这边就不展示完全的 pb.go 文件内容
* 编写服务端代码，在服务端实现 TellYouSomething 方法

```go
func (s *server)TellYouSomething(stream pb.Greeter_TellYouSomethingServer) error{
	log.Printf("Recevied ClientStream request")
	messgeCount := 0
	length := 0
	for {
		something,err := stream.Recv()
		// 接收到客户端所有请求，返回响应结果
		if err == io.EOF{
			return stream.SendAndClose(&pb.HelloReply{Message:fmt.Sprintf("It's over.\nReceived %v times. Length:%v",messgeCount,length)})
		}
		if err != nil{
			return err
		}
		messgeCount ++
		length += len(something.GetLine())
	}
	return nil
}
```

这个与服务端流式 RPC 中客户端的代码比较类似，服务端通过调用 stream 的 Recv() 方法获取客户端发送的请求，直到接受到 io.EOF 信号表示已经接收到全部客户端的请求，可以返回响应了，如果中间接收请求出错，则将错误直接返回给客户端

*  编写客户端代码，在客户端调用 TellYouSomething 方法

```go
func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	clientFunc := "Default"
	if len(os.Args) == 3 {
		name = os.Args[1]
		clientFunc = os.Args[2]
	}else{
		log.Fatal("Please input 2 arguments")
	}

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	switch clientFunc {
	......
	case "ClientStream":
		stream,err := c.TellYouSomething(ctx)
		if err != nil{
			log.Fatalf("TellYouSomething err :%v",err)
		}
		clientStr := []string{"ClientLine1","ClientLine2"}
		for i,v := range(clientStr){
			if err := stream.Send(&pb.Something{LineCode:int64(i),Line:v});err != nil{
				log.Fatalf("TellYouSomething stream error:%v",err)
			}
		}
		rep,err := stream.CloseAndRecv()
		if err != nil{
			log.Fatalf("CloseAndRecd error:%v",err)
		}
		log.Printf("Recive from server:%s",rep.GetMessage())
	......	
}
```

与服务端流式 RPC 中服务端的代码类似，通过调用 stream 的 Send() 方法给客户端发送请求，等所有请求发送完毕通过调用 CloseAndRevc() 方法通知服务端请求发送完毕，并接收服务端的响应。

### 服务端/客户端双向流式 RPC

服务端/客户端双向流式 RPC 即客户端和服务端都是流式方式发送请求的

* 首先在 .proto 文件中定义服务端流式 RPC 的服务方法

```go
service Greeter {
......
	rpc TalkWithMe(stream Something) returns(stream Something){}
}

......
message Something{
	int64 lineCode = 1;
	string line = 2;
}
```

在原先定义的 Greeter 服务的基础上添加了一个TalkWithMe的 RPC 方法，这个 RPC 方法接收的是请求结构是 Something 类型，且用了 stream 定义，表示接受的请求是一个流，Something 类型包含一个 int64 类型的 lineCode 字段和以一个 string 类型的 line 字段，服务端返回给客户端的也是 Something 类型的 stream 流。

* 根据 .proto 文件生成 pb.go 文件，因为是自动生成的，这边就不展示完全的 pb.go 文件内容
* 编写服务端代码，在服务端实现 TalkWithMe 方法

```go
func (s *server)TalkWithMe(stream pb.Greeter_TalkWithMeServer) error{
	log.Printf("Recevied Stream request")
	messageCount := 0
	for {
		something ,err := stream.Recv()
		if err == io.EOF{
			return nil
		}
		messageCount ++
		length := len(something.GetLine())
		line := fmt.Sprintf("Got %s,Length:%v",something.GetLine(),length)
		if err := stream.Send(&pb.Something{LineCode: int64(messageCount),Line:line});err != nil{
			return err
		}
	}
	return nil
}
```

服务端代码与客户端流式 RPC 中服务端代码几乎是一样的，所以就不作解释

* 编写客户端代码，调用 TalkWithMe 方法

```go
func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	clientFunc := "Default"
	if len(os.Args) == 3 {
		name = os.Args[1]
		clientFunc = os.Args[2]
	}else{
		log.Fatal("Please input 2 arguments")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	switch clientFunc {
......
	case "Stream":
		stream,err := c.TalkWithMe(ctx)
		if err != nil{
			log.Fatalf("TalkWithMe err:%v",err)
		}
		waitc := make(chan struct{})
		go func(){
			for {
				something,err := stream.Recv()
				// 服务端信息发送完成，退出
				if err == io.EOF{
					break
				}
				if err != nil{
					log.Fatalf("TalkWithMe stream error:%v",err)
				}
				log.Printf("Got %v:%s\n",something.GetLineCode(),something.GetLine())
			}
		}() 
		clientStr := []string{"one","two","three"}
		for i,v := range(clientStr){
			if err := stream.Send(&pb.Something{LineCode:int64(i),Line:v});err != nil{
				log.Fatalf("TalkWithMe Send error:%v",err)
			}
		}
		stream.CloseSend()
		<- waitc
	default:
		log.Fatal("Please input second args in Default/ServerStream/ClientStream/Stream")
	}
	
}
```

客户端发送请求还是通过调用 stream 的 Send() 方法发送请求，但是通知服务端请求发送完毕是通过调用 CloseSend() 方法，不过因为服务端发送的请求不是一次性发送的，所以这边用 goroutine 新开了一个线程用于接收服务端返回的响应。



### 演示

* 服务端流式 RPC

```

## 终端 1
$ go run server.go
2020/03/29 19:50:47 Received ServerStream request from: Aob

## 终端 2
$ go run client.go Aob ServerStream
2020/03/29 19:50:47 Recevie from server:{LineCode:0 Line:Hello,Aob}
2020/03/29 19:50:47 Recevie from server:{LineCode:1 Line:ServerLine1}
2020/03/29 19:50:47 Recevie from server:{LineCode:2 Line:ServerLine2}
2020/03/29 19:50:47 Recevie from server:{LineCode:3 Line:ServerLine3}
```

* 客户端流式 RPC

```
## 终端 1 
$ go run server.go
2020/03/29 19:51:47 Recevied ClientStream request

## 终端 2
$ go run client.go Aob ClientStream
2020/03/29 19:51:47 Recive from server:It's over.
Received 2 times. Length:22 
```

* 服务端/客户端流式 RPC

```
## 终端 1 
$ go run server.go
2020/03/29 19:52:46 Recevied Stream request

## 终端 2
$ go run client.go Aob Stream      
2020/03/29 19:52:46 Got 1:Got one,Length:3
2020/03/29 19:52:46 Got 2:Got two,Length:3
2020/03/29 19:52:46 Got 3:Got three,Length:5
```

## 总结

以上就是通过 Go 学习 gRPC 的一些记录，只是简单跑通了服务端与客户端之间的请求。[完整代码地址](https://github.com/Win-Man/NormalAccumulation/tree/master/learn-for-go/grpc-example)

一开始刚开始看 gRPC 的时候其实有点晕，因为又是 gRPC 又是 protocol 又是流式 RPC 的，但是实际学习下来发现定义简单的 gRPC 服务还是比较简单的，按照步骤走就可以：

1. 在 .proto 文件中定义 RPC 服务方法和消息
2. 根据 .proto 文件生成 pb.go 文件
3. 实现服务端代码
4. 实现客户端代码

下一步准备学习一下 gRPC 与 TLS 安全加密相关的内容