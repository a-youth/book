---
Title: "kafka 多线程单消费者 vs 单线程单消费者"
Keywords: "kafka,go,多线程消费,多分区"
Description: "1.kafka 多线程处理单个消费者消息 2. kafka 单线程处理单个消费者消息 3. kafka 增加分区数量"
Label: "kafka parition"
---

- kafka 多线程处理单个消费者消息
- kafka 单线程处理单个消费者消息
- kafka 增加分区数量

# kafka 多线程处理单个消费者消息

写php的时候经常拿redis来当做消息队列，然后启动多个进程处理队列中的消息，失败的消息记录写入失败队列中。所以在处理kafka consumer group时首先会想到这种方式来处理消息。只是如果kafka也按照这种方式来的话，保证消息不丢失的语义将会是一件麻烦事。

位移: kafka partition 只会维护一个数字，即处理的最后一条消息。

如果我们启用多线程来处理同一个消费者的消息的话，考虑一下这种情况:

4个线程处理，分别拿到了 位移 5,6,7,8 的消息

- 拿到8的线程成功处理了消息。但是5，6，7的线程处理失败了，此时我们还需要一个队列来处理失败的消息。就像上面处理redis队列一样
- 还是拿到8的线程成功处理了消息，提交了位移，5的线程最后处理完消息，最后提交commit，此时6，7，8的消息会被重复消费

所以我们多线程处理单个消费者消息是需要解决这处理失败，消息重复消费这2个问题，不符合kafka消息交付语义。实现链路长，代码复杂

**那么为什么还是需要多线程消费？**

答：考虑需要使用消息队列场景，比如：视频、图片处理

- 处理周期长，超过web请求设定的超时时间
- 异步处理，业务需求更关注处理结果，对时间有一定包容度
- 服务降级，比如文章阅读统计但是还参杂算法推荐，运营活动，但是不要影响阅读量统计的逻辑。毕竟吞吐量要求比较高

java 实现代码可以看  [胡夕老师的多线程实践](https://www.cnblogs.com/huxi2b/p/13668061.html) 

go实现代码

```go

type Consume struct {
	ctx     context.Context
	encoder encode.Encoder
	// sem 信号量，限制并发数
	sem chan struct{}
}

// n: goroutine 并发数量
func NewPdfConvertConsume(ctx context.Context, encoder encode.Encoder, n int) *Consume {
	return &Consume{
		ctx:     ctx,
		encoder: encoder,
		sem:     make(chan struct{}, n),
	}
}

func (*Consume) Setup(sarama.ConsumerGroupSession) error {
	return nil
}

func (*Consume) Cleanup(sarama.ConsumerGroupSession) error {
	return nil
}

func (p *Consume) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for m := range claim.Messages() {
		p.sem <- struct{}{}
		go func(msg sarama.ConsumerMessage) {
			defer func() {
				<-p.sem
			}()
			data := &action.Context{}
			logger.Debugf("consumer msg: %s", string(msg.Value))
			if err := p.encoder.Decode(data, msg.Value); err != nil {
				logger.Wrapf(err, "msg decoder failed")
			} else {
        // todo 业务逻辑
			}
		}(*m)
		session.MarkMessage(m, "")
	}
	return nil
}

```

# kafka 单线程处理单个消费者的消息

- 可以保证kafka的消息交付语义的，实现方式简单易维护
- 需要更多的TCP连接

- 线程自己处理消息可能会造成超时导致rebalance

如果消息处理时间较长，可以调整参数 或者 细化处理步骤保证消息处理速度 来避免频繁的rebalance

**参数调优**:

`heartbeat.interval.ms` : 心跳间隔，用来保持consumer的会话，并且在有consumer加入或者离开group时帮助进行rebalance，这个值必须要小于 `session.timeout.ms`, 在超过`session.timeout.ms` 时间内如果没有收到hearbeat消息，就会将该consumer移出 consumer group, 一般设置 `session.timeout.ms `的 1/3

`session.timeout.ms` ： session会话过期时间

`max.poll.interval.ms` : 最大poll数据间隔，默认值是3s, 如果超过这个时间还是没有发起poll请求，即使heartbeat依旧在发，还是会把consumer 移出 consumer group



# kafka 增加分区数

```
./kafka-topics.sh --zookeeper localhost:2181/kafka --alter --topic demo --partitions 6
```

