# 深入Kafka：什么是rebalance机制？

## 什么是Rebalance {id="rebalance_1"}

**Rebalance（再平衡）** 是Kafka消费者组的一种机制，用于在消费者组成员发生变化时，重新分配分区给各个消费者，确保：
- 每个分区只被组内的一个消费者消费
- 所有消费者尽可能均衡地分配到分区
- 组内成员变化时保持消费的连续性

## Rebalance的触发条件 {id="rebalance_2"}

- 消费者加入组
- 消费者离开组
- 消费者心跳超时
- 主题元数据变化

## Rebalance的执行过程 {id="rebalance_3"}

### 阶段1：消费者检测到需要Rebalance

### 阶段2：选举Group Coordinator

每个消费者组都有一个协调者（通常是最早创建的broker），负责管理rebalance过程。

### 阶段3：消费者加入组

### 阶段4：分区分配（由Leader消费者执行）

### 阶段5：SyncGroup阶段

Leader将分配方案发送给协调者，协调者再分发给所有消费者。

## 分区分配策略

### 1. Range Assignor（默认策略）
```java
// 按主题范围分配
// 消费者C1: [T0P0, T0P1, T1P0, T1P1]
// 消费者C2: [T0P2, T1P2]
// 消费者C3: [T0P3, T1P3]
```

### 2. RoundRobin Assignor
```java
// 轮询分配所有分区
// 消费者C1: [T0P0, T0P3, T1P2]
// 消费者C2: [T0P1, T1P0, T1P3]  
// 消费者C3: [T0P2, T1P1]
```

### 3. Sticky Assignor（推荐）

```java
// 粘性分配，尽量减少分区移动
// 第一次分配:
// 消费者C1: [T0P0, T0P1, T1P0, T1P1]
// 消费者C2: [T0P2, T1P2]

// 消费者C1离开后重新分配:
// 消费者C2: [T0P0, T0P1, T0P2, T1P0, T1P1, T1P2] 
// 保持原有分配尽可能不变
```

配置粘性分配策略：
```properties
spring.kafka.consumer.properties.partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor
```

### 4. Cooperative Sticky Assignor（2.4.0+强烈推荐）

该分区策略是Kafka 2.4.0引入的，它与Sticky Assignor类似，都是尽量维持原有的分区分配，但是更智能。
最大的区别是：

> `StickyAssignor`仍然是基于`eager`协议，分区重分配时候，都需要`consumers`先放弃当前持有的分区，重新加入`consumer group`;
> 而`CooperativeStickyAssignor`基于`cooperative`协议，该协议将原来的一次全局分区重平衡，改成多次小规模分区重平衡。渐进式的重平衡。

```java
// CooperativeStickyAssignor策略
// 第一次分配:
// 消费者C1: [T0P0, T0P2]
// 消费者C2: [T0P1]

// 消费者C3加入后重新分配:
// 消费者C1: [T0P0]
// 消费者C2: [T0P1]
// 消费者C3: [T0P2]
// 保持原有分配尽可能不变
```

#### 基于eager协议的分区重分配策略流程：

1. consumer1、 consumer2正常发送心跳信息到Group Coordinator。
2. 随着consumer3加入，Group Coordinator收到对应的Join Group请求，Group Coordinator确认有新成员需要加入消费者组。
3. Group Coordinator 通知consumer1和consumer2，需要rebalance（再平衡）了。
4. consumer1和consumer2放弃（revoke）当前各自持有的已有分区，重新发送Join Group请求到Group Coordinator。
5. Group Coordinator依据指定的分区分配策略的处理逻辑，生成新的分区分配方案，然后通过Sync Group请求，将新的分区分配方案发送给consumer1、consumer2、consumer3。
6. 所有consumers按照新的分区分配，重新开始消费数据。

#### 基于cooperative协议的分区分配策略的流程：

1. consumer1、 consumer2正常发送心跳信息到Group Coordinator。
2. 随着consumer3加入，Group Coordinator收到对应的Join Group请求，Group Coordinator确认有新成员需要加入消费者组。
3. Group Coordinator 通知consumer1和consumer2，需要rebalance了。
4. consumer1、consumer2通过Join Group请求将已经持有的分区发送给Group Coordinator。注意：并没有放弃(revoke)已有分区。
5. Group Coordinator取消consumer1对分区p2的消费，然后发送sync group请求给consumer1、consumer2。
6. consumer1、consumer2接收到分区分配方案，重新开始消费。至此，一次Rebalance完成。
7. 当前p2也没有被消费，再次触发下一轮rebalance，将p2分配给consumer3消费。

## Rebalance 协议

Kafka的消费者组Rebalance协议分为两种：
- **eager**：重新平衡协议要求消费者在参与重新平衡事件之前**始终撤销其拥有的所有分区**。因此，它允许完全改组分配。
- **cooperative**：协议允许消费者在**参与再平衡事件之前保留其当前拥有的分区**。分配者不应该立即重新分配任何拥有的分区，而是可以指示消费者需要撤销分区，以便可以在下一次重新平衡事件中将被撤销的分区重新分配给其他消费者。

## Rebalance的影响和问题 {id="rebalance_4"}

### 1. 消费暂停（Stop-the-World）

### 2. 重复消费

## 优化Rebalance的策略 {id="rebalance_5"}

### 1. 合理配置参数
```properties
# 优化rebalance配置
spring.kafka.consumer.session.timeout.ms=30000
spring.kafka.consumer.heartbeat.interval.ms=10000
spring.kafka.consumer.max.poll.interval.ms=300000
spring.kafka.consumer.max.poll.records=500
```

### 2. 使用静态成员资格（避免不必要的Rebalance）

```properties
# 启用静态成员资格
spring.kafka.consumer.properties.group.instance.id=consumer-instance-1
```

## 总结

Rebalance是Kafka保证高可用和扩展性的重要机制，但频繁的Rebalance会影响系统性能。通过合理配置参数、使用粘性分配策略、实现优雅的Rebalance处理和使用静态成员资格，可以显著减少Rebalance的影响，提高系统的稳定性和性能。