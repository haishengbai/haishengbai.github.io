---
title: java_spring_cloud_stream_kafka
date: 2020-05-04 16:38:47
tags: stream,kafka
category: spring cloud
---

# Spring Cloud Stream 之 kafka 初窥

> 版本：Fishtown
>
> 原文链接: https://cloud.spring.io/spring-cloud-static/spring-cloud-stream/2.1.3.RELEASE/single/spring-cloud-stream.html#spring-cloud-stream-preface-adding-message-handler
>
> os: 由于公司在用F版，有时间再研究下最新版本吧





### Spring Cloud Stream 简介

Spring Cloud Stream是用于构建消息驱动的微服务应用程序的框架。并使用Spring Integration提供与消息代理的连接。它提供了来自多家供应商的中间件的合理配置，并介绍了持久性发布-订阅语义，使用者组和分区的概念。

### 简单使用

1. 导入包

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-stream</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-stream-binder-kafka</artifactId>
   </dependency>
   ```

2. 将`@EnableBinding`注释添加到应用程序中，以立即连接到消息代理，添加`@StreamListener`以使该方法接收事件以进行流处理。以下示例显示了接收外部消息的接收器应用程序：

   ```java
   @SpringBootApplication
   @EnableBinding(Sink.class)
   public class VoteRecordingSinkApplication {
   
     public static void main(String[] args) {
       SpringApplication.run(VoteRecordingSinkApplication.class, args);
     }
   
     @StreamListener(Sink.INPUT)
     public void processVote(Vote vote) {
         votingService.recordVote(vote);
     }
   }
   ```

   `@EnableBinding` 注解可以使用一个或多个参数(本例中，采用一个`Sink`接口)。参数接口声明输入和输出通道。Spring Cloud Stream 提供`Source`，`Sink`和`Processor`接口。也可以自定义接口。

   `Sink`接口的定义：

   ```java
   public interface Sink {
     String INPUT = "input";
   
     @Input(Sink.INPUT)
     SubscribableChannel input();
   }
   ```

   `@Input`注解标记一个输入的通道，通过该方法接收消息输入到应用程序。

   `@Output`注解标记一个输出通道，通过该方法发布消息从程序输出。

   `@Input`和`@Output`注解可以采取频道名称作为参数。如果未提供名称，则使用所注释方法的方法名。

   

### 主要概念

   ###### Spring Cloud Stream 的应用模型

   Spring Cloud Stream应用程序由与中间件无关的核心组成。该应用程序通过Spring Cloud Stream注入到其中的输入和输出通道与外界进行通信。通道通过特定于中间件的Binder实现连接到外部代理。

   ![Spring Cloud Stream应用程序](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/SCSt-with-binder.png)

   ###### Binder

   Spring Cloud Stream 为Kafka和Rabbit MQ提供了默认的binder实现。

   Spring Cloud Stream 提供一个TestSupportBinder用于测试。也可以用API编写自己的Binder。

   具体可以配置在application.yml文件中。

   ###### 订阅-发布支持

   应用程序之间的通信遵循发布-订阅模型，其中数据通过共享主题进行广播。

   **Spring Cloud Stream发布-订阅**

   ![SCSt传感器](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/SCSt-sensors.png)

   

   传感器报告给HTTP端点的数据将发送到名为的公共目标`raw-sensor-data`。从公共目标开始，它由计算时间窗平均值的微服务和另一个将原始数据提取到HDFS（Hadoop分布式文件系统）的微服务独立处理。为了处理数据，两个应用程序都在运行时将声明的主题作为其输入。

   发布-订阅通信模型降低了生产者和使用者的复杂性，并允许在不中断现有流程的情况下将新应用添加到拓扑中。例如，在平均计算应用程序的下游，您可以添加一个应用程序，该应用程序计算用于显示和监视的最高热度值。然后，您可以添加另一个解释相同平均值流以进行故障检测的应用程序。通过共享主题而不是点对点队列进行所有通信可以减少微服务之间的耦合。

   尽管发布-订阅消息传递的概念并不新鲜，但Spring Cloud Stream采取了额外的步骤，使其成为其应用程序模型的明智选择。通过使用本机中间件支持，Spring Cloud Stream还简化了跨不同平台的发布-订阅模型的使用。

   ###### Group(消费者)

   Spring Cloud Stream通过消费者群体的概念对这种行为进行建模。每个消费者都可以使用`spring.cloud.stream.bindings..group`属性指定组名。

   订阅给定目标的所有组都将收到已发布数据的副本，但是每个组中只有一个成员从该目标接收给定消息。默认情况下，当未指定组时，Spring Cloud Stream会将应用程序分配给与所有其他使用者组具有发布-订阅关系的匿名且独立的单成员使用者组。

   ###### 消费者类型

   支持两种类型的消费者：

   - 消息驱动（有时称为异步）
   - 轮询（有时称为同步）

   在2.0版之前，仅支持异步使用者。消息一旦可用，就会被传递，并且有线程可以处理它。

   当您希望控制消息的处理速率时，可能需要使用同步消费者。

   ###### 耐久性

   与Spring Cloud Stream公认的应用程序模型一致，消费者组订阅是持久的。也就是说，binder的实现可确保组订阅是持久的，并且一旦创建了至少一个组订阅，该组将接收消息，即使在组中所有应用程序停止，依然会发送消息。

   > 匿名订阅本质上是非持久的。对于某些binder实现（例如RabbitMQ），可能具有非持久的组订阅。

   通常，在将应用程序绑定到给定目标时，最好始终指定使用者组。扩展Spring Cloud Stream应用程序时，必须为其每个输入绑定指定使用者组。这样做可以防止应用程序的实例接收重复的消息（除非需要这种行为）

   ###### 分区支持

   Spring Cloud Stream支持在给定应用程序的多个实例之间对数据进行分区。在分区方案中，物理通信介质（例如代理主题）被视为结构化为多个分区。一个或多个生产者应用程序实例将数据发送到多个消费者应用程序实例，并确保由共同特征标识的数据由同一消费者实例处理。

   Spring Cloud Stream提供了用于以统一方式实现分区处理用例的通用抽象。因此，无论代理本身是自然分区（例如Kafka）还是非自然分区（例如RabbitMQ），都可以使用分区。

   **Spring Cloud Stream分区**

   ![SCSt分区](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/SCSt-partitioning.png)

   

   分区是有状态处理中的关键概念，对于确保所有相关数据都一起处理，分区是至关重要的（出于性能或一致性方面的考虑）。例如，在带时间窗的平均计算示例中，重要的是，来自任何给定传感器的所有测量都应由同一应用实例处理。

   > 要设置分区处理方案，必须同时配置数据生产者端和数据消费者端。

### 编程模型

- **目标绑定器（Destination Binders）：**负责与消息队列集成的组件。

- **目标绑定（Destination Bindings）：**消息队列和提供*生产者*和*消费者*消息的应用程序之间的桥梁组件（由Destination Bindings创建）。

- **消息（Message）：**生产者和消费者使用的规范数据结构，用于与目标绑定程序（并因此通过外部消息传递系统与其他应用程序）进行通信。

###### Destination Binders

  Destination Binders是Spring Cloud Stream的扩展组件，负责提供必要的配置和实现以促进与外部消息传递系统的集成。这种集成负责连接，委派和与生产者和消费者之间的消息路由，数据类型转换，用户代码调用等等。

Binders要承担很多样板工作，否则这些工作就落在了您的肩膀上。但是，要实现这一点，binder仍然需要用户提供的一些简单但需要的指令，通常以某种类型的配置形式出现。

所有可用的binder和配置选项（本手册的其余部分都涉及它们）下一节将详细讨论。

###### Destination Bindings

 test!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!----------------------------------