---
layout:     post
title:      "C# 使用 grpc 问题汇总"
subtitle:   ""
date:       2016-10-24
author:     "Tim zhong"
header-img: "/img/post/10-24-grpcstart/bk.png"
catalog: true
tags:
    - c#
    - grpc
---

> 使用.net架构的系统，不用 wcf，选用 grpc 是一个最新的选择。

grpc 是个跨平台的远程调用框架，第一次在windows平台使用，有很多问题需要面对，下面做个汇总，以备检查

![跨平台](/img/post/10-24-grpcstart/cross.png)

# 消息长度设置

## 出现的问题 1

客户端出现异常

``` console
未经处理的异常:  Grpc.Core.RpcException: Status(StatusCode=Internal, Detail="Max message size exceeded")
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 Grpc.Core.Internal.AsyncCall`2.UnaryCall(TRequest msg)
   在 Grpc.Core.Calls.BlockingUnaryCall[TRequest,TResponse](CallInvocationDetails`2 call, TRequest req)
   在 Grpc.Core.DefaultCallInvoker.BlockingUnaryCall[TRequest,TResponse](Method`2 method, String host, CallOptions options, TRequest request)
   在 Grpc.Core.Internal.InterceptingCallInvoker.BlockingUnaryCall[TRequest,TResponse](Method`2 method, String host, CallOptions options, TRequest request)
   在 Helloworld.Greeter.GreeterClient.GetUsers(SearchUserRequest request, CallOptions options) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\Greeter\HelloworldGrpc.cs:行号 148
   在 Helloworld.Greeter.GreeterClient.GetUsers(SearchUserRequest request, Metadata headers, Nullable`1 deadline, CancellationToken cancellationToken) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\Greeter\HelloworldGrpc.cs:行号 144
   在 GreeterClient.Program.Main(String[] args) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\GreeterClient\Program.cs:行号 61
```

在服务端出现异常

``` console
W1024 09:50:02.749888 Grpc.Core.Server Exception while handling RPC. System.InvalidOperationException: Error sending status from server.
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.ValidateEnd(Task task)
   在 Grpc.Core.Internal.UnaryServerCallHandler`2.<HandleCall>d__0.MoveNext()
--- 引发异常的上一位置中堆栈跟踪的末尾 ---
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 Grpc.Core.Server.<HandleCallAsync>d__11.MoveNext()
```
## 出现问题 2

在客户端出现异常

``` console
未经处理的异常:  Grpc.Core.RpcException: Status(StatusCode=Internal, Detail="{"created":"@1477274209.883000000","description":"RST_STREAM","file":"c:\jenkins\workspace\gRPC_build_artifacts\architecture\x86\language\csharp\platform\windows\vsprojects\..\src\core\ext\transport\chttp2\transport\frame_rst_stream.c","file_line":107,"http2_error":2}")
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 Grpc.Core.Internal.AsyncCall`2.UnaryCall(TRequest msg)
   在 Grpc.Core.Calls.BlockingUnaryCall[TRequest,TResponse](CallInvocationDetails`2 call, TRequest req)
   在 Grpc.Core.DefaultCallInvoker.BlockingUnaryCall[TRequest,TResponse](Method`2 method, String host, CallOptions options, TRequest request)
   在 Grpc.Core.Internal.InterceptingCallInvoker.BlockingUnaryCall[TRequest,TResponse](Method`2 method, String host, CallOptions options, TRequest request)
   在 Helloworld.Greeter.GreeterClient.GetUsers(SearchUserRequest request, CallOptions options) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\Greeter\HelloworldGrpc.cs:行号 148
   在 Helloworld.Greeter.GreeterClient.GetUsers(SearchUserRequest request, Metadata headers, Nullable`1 deadline, CancellationToken cancellationToken) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\Greeter\HelloworldGrpc.cs:行号 144
   在 GreeterClient.Program.Main(String[] args) 位置 E:\MyTest\grpc-1.0.x\examples\csharp\helloworld\GreeterClient\Program.cs:行号 71
```
在服务端出现异常 

``` console
W1024 09:56:49.886636 Grpc.Core.Internal.UnaryServerCallHandler`2 Exception occured in handler. System.ArgumentException: 值不在预期的范围内。
   在 Grpc.Core.Internal.UnaryServerCallHandler`2.<HandleCall>d__0.MoveNext()
W1024 09:56:49.889643 Grpc.Core.Server Exception while handling RPC. System.InvalidOperationException: Error sending status from server.
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.ValidateEnd(Task task)
   在 Grpc.Core.Internal.UnaryServerCallHandler`2.<HandleCall>d__0.MoveNext()
--- 引发异常的上一位置中堆栈跟踪的末尾 ---
   在 System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
   在 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   在 Grpc.Core.Server.<HandleCallAsync>d__11.MoveNext()
```

## 解决方法

出现这个问题是grpc对默认的消息大小(massage size)做了限制，限制的值在各个版本中还不太一样，为了让grpc的效率更高，开发者认为应该不要太大，4M比较合适。

 但实际应用中，虽然可以规定编写contract要合理，但是还会很多大的对象。

 解决方法是手动设置messagesize的值。

 问题1 是没有在客户端设置messagesize，接收到大的对象后客户端无法解析。问题2 是在服务端没有设置messagesize，服务端无法解析发送来的消息。

在服务端需要的代码

{%highlight cSharp %}
 var options = new List<ChannelOption> {
                new ChannelOption(ChannelOptions.MaxMessageLength,int.MaxValue)
            };
 Channel channel = new Channel("127.0.0.1:50051", ChannelCredentials.Insecure, options);
{%endhighlight%}

在客户端需要的代码

{%highlight cSharp %}
Server server = new Server(new List<ChannelOption> { new ChannelOption(ChannelOptions.MaxMessageLength, int.MaxValue) })
{
        Services = { Greeter.BindService(new GreeterImpl()) },
        Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
 };
 {%endhighlight%}

 到底这个值设置为多少合理，如果设置到int.MaxValue，对性能有什么影响，在后面使用中逐步体会。