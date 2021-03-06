---
Title: "golang如何远程调试代码？"
Keywords: "远程调试代码,delve,dlv,goland"
Description: "golang一般都是运行在linux机器上的，如果用不到linux特有服务的话都好说，本地直接调试开发就ok了，万一用到epoll了，只能远程调试了"
Label: "delve"
---

最近在研究websocket的路上愈走愈远了，从开始只是搭建聊天室到单机服务如何支持百万连接。现在是在弄epoll的道路上，经历一路坎坷。这里记录一下心路历程。

问题： **epoll这只能运行在linux，有问题没发调试，蛋疼吗？**

首先想到装个docker，运行个centos，然后对着docker又是一顿撸。还是没发调试，

[gdb](/linux/gdb.md) 也是可以调试go程序的，问题是，这tm一个项目几百个文件都有了，我gdb怎么断点啊，大概的思路就是从函数调用的地方开始入手，这啥时候是个头啊，简单的还行，果断放弃了

[delve](https://github.com/go-delve/delve) dlv其实很不错的，也能像gdb一样支持，对goroutine支持也很完美，问题来了，docker运行的centos，我就是装个dlv, `go get -u github.com/go-delve/delve/cmd/dlv` 装不动，只能心里默默诅咒那个下决定对github限速的人。google我能理解，但是github这个真理解不了。只能走代理了。关于[怎么在centos上使用代理](/linux/centos使用ss代理.md)

**goland配置**

`Run/Edit configurations` 

![](./assert/delve-goland.png)

**host配置**

docker运行的centos，创建container的时候，指定 `-p 2345:2345` , 把容器的端口映射到宿主机器上就ok了

**编译调试**

goland里面介绍的很明白，有一个坑 `exec ./main`的时候后面怎么传参数给调试进程，正确的姿势: 后面跟 `--`，告诉dlv，这是给调试进程用的。

```shell
dlv --listen=:9004 --headless=true --api-version=2 --accept-multiclient exec ./main -- start --env debug
```

**dlv listen以后怎么退出来？**

不好意思，我也没有找到正确的姿势，万能的方式，新启动一个终端kill掉他，然后会问，docker怎么attach以后还是当前窗口，这个....

```shell
docker exec -it centos7 bash
```

**看一下我调试解决的bug**

```go
func start() {
	for {
		clients, err := hub.Wait()
		if err != nil {
			logger.Debugf("epoll wait %v", err)
			continue
		}
		for _, client := range clients {
      // 这里是68行，下面这一行会报 空指针 的错误
			bt, opCode, err := wsutil.ReadClientData(client.conn)
			if err != nil {
				log.Printf("read message error: %v", err)
				continue
			}
			// 处理ping/pong/close
			if opCode.IsControl() {
				err := wsutil.HandleClientControlMessage(client.conn, wsutil.Message{
					OpCode:  opCode,
					Payload: bt,
				})
				if err != nil {
					if _, ok := err.(wsutil.ClosedError); ok {
						hub.unregister <- *client
					}
					continue
				}
				continue
			}
			cmsg := ClientMessage{}
			if err := json.Unmarshal(bt, cmsg); err != nil {
				logger.Errorf("json unmarshal error: %v", err)
				continue
			}
			hub.broadcast <- NewDefaultMsg(client, cmsg.Content, cmsg.ChannelId)
		}
	}
}
```

报错信息:

```go
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x8 pc=0xdf08d4]

goroutine 20 [running]:
dyc/internal/module/chat.start()
	/root/github/api.douyacun.com/internal/module/chat/client.go:68 +0x64
created by dyc/internal/module/chat.init.0
	/root/github/api.douyacun.com/internal/module/chat/client.go:26 +0x82
```

可以看出来是client为nil了，那就出现`hub.Wait()`,  调用epoll wait的时候返回的client有nil，debug一下

![](./assert/delve-debug.png)

wait函数的代码:

```go
func (e *epoll) Wait() ([]*Client, error) {
	events := make([]syscall.EpollEvent, 100)
	n, err := syscall.EpollWait(e.Fd, events, 100)
	if err != nil {
		return nil, err
	}
	connections := make([]*Client, n)
	for i := 0; i < n; i++ {
		conn := e.connections[int(events[i].Fd)]
		connections = append(connections, conn)
	}
	return connections, nil
}
```

next一直运行，看一下有连接进来的情况:

![](./assert/delve-debug-wait.png)

这里应该是对`syscall.EpollWait` 第二个参数events， 第一个返回值n的理解有问题，但这不是关键，关键是远程调试姿势有了。

