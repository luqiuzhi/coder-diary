# Kafka中的ACK机制

## 一、什么是Kafka的ACK机制？

ACK（Acknowledgment）确认机制是Kafka生产者（Producer）与Broker之间的一种可靠性协议。它定义了生产者发送一条消息后，需要等待多少个Broker副本确认接收，才认为这条消息发送成功。

这个机制直接决定了Kafka在**消息可靠性**和**吞吐量**之间的权衡。ACK级别设置越高，数据越安全，但吞吐量越低（延迟越高）；反之，吞吐量越高，但数据丢失的风险越大。

---

## 二、Kafka的三种ACKs配置

在Kafka中，通过配置 `acks` 参数来指定确认级别。这个参数在Spring Boot中对应 `spring.kafka.producer.acks`。

### 1. `acks = 0` - **"发后即忘"（Fire and Forget）**

* **机制**：生产者发送消息后，**完全不需要等待Broker的任何确认**。它假设消息总是能够成功到达Broker。
* **优点**：**延迟极低，吞吐量最高**。因为不需要等待任何网络往返或磁盘写入。
* **缺点**：**数据极易丢失**。如果网络闪断、Broker宕机，或者消息还没来得及发送到任何Broker，消息就没了，且生产者无从知晓。
* **适用场景**：适用于对可靠性要求极低的场景，例如收集一些可以容忍丢失的实时日志 metrics。

### 2. `acks = 1` - **默认Leader确认（Default）**

* **机制**：生产者发送消息后，会等待**分区的Leader副本**将消息成功写入其**本地日志（Log）** 后，返回确认响应。
* **优点**：在**吞吐量**和**可靠性**之间取得了很好的平衡。保证了Leader副本上有这条消息。
* **缺点**：仍然有数据丢失的风险。如果Leader副本刚写入成功就突然宕机，且其他Follower副本还没来得及同步这条消息，那么新的Leader被选举出来后，这条消息就永远丢失了（因为它不在最新的副本集中）。
* **适用场景**：这是Kafka的默认配置。适用于大多数业务场景，能够接受在Leader突然故障时丢失极少量消息。

### 3. `acks = all` (或 `acks = -1`) - **全副本确认（Full ISR确认）**

* **机制**：生产者发送消息后，必须等待**分区的Leader副本和所有处于ISR（In-Sync Replicas，同步副本集）中的Follower副本**
  都成功将消息写入本地日志后，才会返回确认响应。这是最严格的确认方式。
* **优点**：**最高级别的可靠性**。只要至少有一个ISR中的副本存活，消息就不会丢失。
* **缺点**：**延迟最高，吞吐量最低**
  。因为它需要等待多个副本的网络传输和磁盘写入。如果ISR中有一个副本响应缓慢，就会拖慢整个发送过程。此外，如果ISR中只剩下Leader一个副本（例如，所有Follower都因网络问题被踢出了ISR），那么
  `acks=all` 的行为会退化为 `acks=1`。
* **适用场景**：对数据可靠性要求极高的场景，如金融交易、关键业务状态变更等。

---

## 三、结合Spring Boot应用说明

在Spring Boot应用中，我们通过 `application.yml` (或 `application.properties`) 和 `KafkaTemplate` 来配置和使用ACK机制。

### 1. 配置文件示例 (`application.yml`)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      # 关键配置：设置ACK机制
      acks: all
      # 其他重要相关配置
      retries: 3 # 发送失败后的重试次数，对于高可靠性是必须的
      properties:
        delivery.timeout.ms: 120000 # 配置从发送到收到ACK的总超时时间
        request.timeout.ms: 30000 # 生产者等待请求响应的超时时间
        linger.ms: 5 # 生产者发送消息前等待少量消息加入批次的时间，可提升吞吐量
    consumer:
      group-id: my-spring-boot-group
      auto-offset-reset: earliest
      enable-auto-commit: false # 推荐设置为false，由程序手动提交偏移量以保证精确一次处理
```

### 2. 发送消息的代码 (使用 `KafkaTemplate`)

Spring Boot的 `KafkaTemplate` 封装了底层的生产者，其发送行为受上述配置控制。

```java
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;

@Service
public class MessageProducerService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public MessageProducerService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendMessage(String topic, String message) {
        // 发送消息，返回一个ListenableFuture
        ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(topic, message);

        // 添加回调，用于异步处理发送成功或失败的结果
        future.addCallback(new ListenableFutureCallback<>() {
            @Override
            public void onSuccess(SendResult<String, String> result) {
                System.out.println("Sent message=[" + message + 
                                  "] with offset=[" + result.getRecordMetadata().offset() + "]");
            }

            @Override
            public void onFailure(Throwable ex) {
                // 当ACK未返回或返回失败（如重试次数用尽后）会进入这里
                System.err.println("Unable to send message=[" + message + "] due to : " + ex.getMessage());
                // 此处应添加业务级的重试或降级处理逻辑
            }
        });
    }
}
```

### 3. 代码与ACK机制的结合解析

* **当 `acks=all` 时**：
    1. `kafkaTemplate.send()` 被调用，消息被放入生产者的缓冲区。
    2. 生产者线程异步地将消息发送到Kafka Broker的Leader。
    3. Leader会等待所有ISR中的Follower副本都同步成功。
    4. 一旦所有副本确认，Broker向生产者发送ACK响应。
    5. 生产者收到ACK后，Spring Boot的 `ListenableFuture` 会触发 `onSuccess` 回调方法。
    6. 如果在这个过程中发生网络问题、或者副本无法在指定 `request.timeout.ms` 内完成同步，生产者会根据 `retries`
       配置进行重试。如果重试多次后仍然失败，将触发 `onFailure` 回调。

* **重试（`retries`）机制的重要性**：
  仅仅设置 `acks=all` 并不完全可靠。例如，如果Leader选举正在发生，短时间内可能没有Leader，发送请求会失败。此时必须配合
  `retries`（重试）机制，生产者会自动重试发送，直到新的Leader被选举出来，消息最终得以成功存储。在Kafka 2.4+版本，与
  `delivery.timeout.ms` 配合使用可以更好地控制重试行为。

---

## 四、总结与最佳实践建议

| ACK级别      | 可靠性    | 吞吐量 | Spring Boot配置 | 适用场景             |
|:-----------|:-------|:----|:--------------|:-----------------|
| **0**      | 最低     | 最高  | `acks: 0`     | 日志收集， metrics    |
| **1**      | 中等     | 中等  | `acks: 1`     | 一般业务消息，容忍少量丢失    |
| **all/-1** | **最高** | 最低  | `acks: all`   | **支付、交易、关键状态变更** |

**在Spring Boot中构建高可靠性消息发送的最佳实践：**

1. **设置 `acks: all`**：确保消息被完整复制到ISR中的所有副本。
2. **开启重试 `retries: <一个较大的值>`** 或 `retries: MAX_INT`：与 `acks=all` 搭配，应对临时性故障。
3. **合理设置超时**：配置 `delivery.timeout.ms` 和 `request.timeout.ms` 以控制生产者的总等待时间和每次请求的等待时间。
4. **使用生产者回调**：就像示例中一样，务必为 `KafkaTemplate.send()` 添加回调函数，并实现健壮的失败处理逻辑（如记录错误日志、告警、存入死信队列等）。
5. **消费者端配合**：生产者保证了消息不丢失，消费者端需要将消费逻辑和偏移量提交（Offset Commit）放在一个事务中（例如关闭自动提交
   `enable-auto-commit: false` 并手动提交），才能实现端到端的精确一次（Exactly-Once）语义。

