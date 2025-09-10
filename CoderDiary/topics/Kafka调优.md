# Kafka调优

## 参数调优

1. `flush.ms`增加**page cache**的刷新时间，提高吞吐量。
2. `flush.messages`控制多少条消息触发一次刷新，增加可提高吞吐量。
3. `compression.type`消息压缩策略。常见的压缩策略有Gzip、Snappy、Lz4 和 Zstd，其中Zstd和Snappy是最平衡的策略，兼顾性能和压缩率。
    - 压缩适用场景：重复数据多，如果本身只是传输UUID、MD5等值，压缩无效；CPU过剩，或节省磁盘和带宽的收益大于多出的CPU的成本；系统对压缩时略微增加的延迟无要求。
    - 压缩配置适用于Broker和Topic两个级别，Topic级别配置会覆盖Broker级别配置。
4. 批量发送参数：
   - batch-size：producer端配置，批量发送消息的大小，默认16KB。
   - buffer-memory：producer端配置，缓存消息的内存大小，默认32MB。如果将要发送的消息大于这个值，则producer会阻塞，等待消息发送完毕。
   - linger.ms：producer端配置，消息发送延时，默认0。
5. `num.network.threads`网络IO操作线程数可以设置为2N或N+1，充分利用CPU资源。
6. JVM参数调优，包括堆内存、GC等参数。
7. `socket.send.buffer.bytes` 和 `socket.receive.buffer.bytes`可以调节发送和接收缓冲区大小。

## 硬件调优

1. 分配足够的带宽。
2. 使用高速SSD。