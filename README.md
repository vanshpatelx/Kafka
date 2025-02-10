# Kafka


---

## **🚀 Summary**
| **Scenario** | **Partitions** | **Consumer Group Strategy** | **Message Distribution** |
|-------------|--------------|--------------------------|--------------------------|
| **Multiple Partitions, Multiple Services** | ✅ Works | **Separate consumer groups per service** | Each service gets full messages, each pod gets unique events. |
| **Multiple Partitions, Single Service** | ✅ Works | **Single consumer group per service** | Each pod gets a separate partition (parallel processing). |
| **Single Partition, All Pods Must Receive** | ❌ Doesn't work normally | **Unique consumer group per pod OR fan-out pattern** | All pods receive messages correctly. |

---

# **Designing Kafka-Based Microservices Communication**

## **1️⃣ Multiple Partitions, Multiple Services**
### **Scenario**
- `auth-service` publishes `auth.registered` events.
- Two services (`db-server`, `engine`), each with **multiple pods**, need to process the events.

### **Kafka Configuration**
```sh
kafka-topics.sh --create --topic auth.registered --partitions 2 --replication-factor 3 --bootstrap-server kafka:9092
```

### **Consumer Group Strategy**
| **Partition** | **DB Server Pod (db-server-group)** | **Engine Pod (engine-group)** |
|--------------|--------------------------------|----------------------------|
| Partition 0  | Pod 1                          | Pod 1                      |
| Partition 1  | Pod 2                          | Pod 2                      |

### **Producer Code (auth-service)**
```typescript
import { Kafka } from "kafkajs";
const kafka = new Kafka({ clientId: "auth-service", brokers: ["kafka:9092"] });
const producer = kafka.producer();

const publishEvent = async () => {
    await producer.connect();
    await producer.send({
        topic: "auth.registered",
        messages: [{ value: JSON.stringify({ userId: "12345", email: "user@example.com" }) }],
    });
};

publishEvent();
```

### **Consumer Code (db-server and engine service)**
```typescript
const consumer = kafka.consumer({ groupId: "db-server-group" });
await consumer.connect();
await consumer.subscribe({ topic: "auth.registered", fromBeginning: false });

await consumer.run({
    eachMessage: async ({ message }) => {
        console.log(`DB Server processing: ${message.value}`);
    },
});
```
✅ **Each pod within a service gets different events.**  
✅ **Each service gets all messages separately.**

---

## **2️⃣ Multiple Partitions, Single Service**
### **Scenario**
- `order-service` publishes `order.created`.
- `engine-service` (multiple pods) processes the orders in parallel.

### **Kafka Configuration**
```sh
kafka-topics.sh --create --topic order.created --partitions 4 --replication-factor 3 --bootstrap-server kafka:9092
```

### **Consumer Group Strategy**
| **Partition** | **Assigned Engine Pod** |
|--------------|-------------------------|
| Partition 0  | Pod 1                   |
| Partition 1  | Pod 2                   |
| Partition 2  | Pod 3                   |
| Partition 3  | Pod 4                   |

### **Consumer Code (engine-service)**
```typescript
const consumer = kafka.consumer({ groupId: "engine-group" });
await consumer.connect();
await consumer.subscribe({ topic: "order.created", fromBeginning: false });

await consumer.run({
    eachMessage: async ({ partition, message }) => {
        console.log(`Engine Pod processing partition ${partition}: ${message.value}`);
    },
});
```
✅ **Each pod processes different partitions, ensuring full parallelism.**  
✅ **Scaling works automatically by adding more partitions.**

---

## **3️⃣ Single Partition, All Pods Must Receive**
### **Scenario**
- `notification-service` consumes `user.notification.sent`.
- **All pods** of `notification-service` must get the same event.

### **Problem: Default Kafka Behavior**
Kafka allows **only one consumer per partition per group**, meaning:
❌ **Only one pod will process messages by default.**

### **✅ Solution 1: Unique Consumer Groups Per Pod**
Assign each pod its own unique consumer group:
```typescript
const consumer = kafka.consumer({ groupId: `notification-group-${process.pid}` });
await consumer.connect();
await consumer.subscribe({ topic: "user.notification.sent", fromBeginning: false });
```
✅ **Each pod gets all messages since they have different consumer groups.**

### **✅ Solution 2: Fan-Out Pattern with a New Topic**
- One pod reads from `user.notification.sent`, republishes to `notification.fanout`.
- All notification pods consume from `notification.fanout`.

```typescript
const producer = kafka.producer();
await producer.send({
    topic: "notification.fanout",
    messages: [{ value: JSON.stringify(notificationEvent) }],
});
```
✅ **All pods receive messages correctly!**

---

## **🚀 Improved Summary**
| **Scenario**                          | **Partitions** | **Consumer Group Strategy**                | **Message Distribution**                           |
|--------------------------------------|--------------|-----------------------------------|------------------------------------------------|
| **Multiple Partitions, Multiple Services** | ✅ Works     | Separate consumer groups per service   | Each service gets full messages, each pod gets unique events. |
| **Multiple Partitions, Single Service**   | ✅ Works     | Single consumer group per service      | Each pod gets a separate partition (parallel processing). |
| **Single Partition, All Pods Must Receive** | ❌ Doesn't work normally | Unique consumer group per pod OR fan-out pattern | All pods receive messages correctly. |


### **Extending Kafka Publisher Side - Multiple Publishers & Groups**  

Now, let's explore two advanced scenarios:  
1️⃣ **Multiple publishers sending messages to the same Kafka topic using the same producer group**  
2️⃣ **Multiple publishers, each with separate producer groups, sending messages to the same event topic**  

---

## **1️⃣ Multiple Publishers Using the Same Producer Group**  
✅ **Scenario**  
- We have multiple instances of `auth-service` publishing `auth.registered` events.  
- All instances share the same `producer-group`, ensuring **load-balanced message production**.  

✅ **How It Works?**  
- Since Kafka **doesn’t enforce a producer group concept**, producers can simply publish events to a topic without conflict.  
- Each producer (auth-service instance) **adds messages to the same topic**, and consumers process them as usual.  

### **Kafka Setup**  
```sh
kafka-topics.sh --create --topic auth.registered --partitions 3 --replication-factor 3 --bootstrap-server kafka:9092
```

### **Multiple Publishers Code (Same Group)**  
```typescript
import { Kafka } from "kafkajs";

const kafka = new Kafka({ clientId: "auth-service", brokers: ["kafka:9092"] });
const producer = kafka.producer();

const publishEvent = async (userId: string, email: string) => {
    await producer.connect();
    await producer.send({
        topic: "auth.registered",
        messages: [{ value: JSON.stringify({ userId, email }) }],
    });
    console.log(`Published event for user: ${userId}`);
};

publishEvent("12345", "user1@example.com");
publishEvent("67890", "user2@example.com");
```
✅ **Multiple instances of `auth-service` can safely produce events to the same topic.**  
✅ **Kafka ensures high availability & ordering per partition.**  

---

## **2️⃣ Multiple Publishers with Different Producer Groups (Fan-In Pattern)**  
✅ **Scenario**  
- Multiple services (`auth-service`, `web-service`, `mobile-service`) publish events to the same topic `user.activity`.  
- Each publisher has **different producer clients**, but **all messages are consumed by one consumer group**.  

✅ **How It Works?**  
- Kafka **doesn’t require a producer group concept** (unlike consumer groups).  
- Each service **publishes events independently**, and consumers **process them from the same topic**.  

### **Kafka Setup**  
```sh
kafka-topics.sh --create --topic user.activity --partitions 3 --replication-factor 3 --bootstrap-server kafka:9092
```

### **Publisher Code (`auth-service`, `web-service`, `mobile-service`)**  
#### **Auth-Service Publisher**  
```typescript
const kafka = new Kafka({ clientId: "auth-service", brokers: ["kafka:9092"] });
const producer = kafka.producer();

const publishEvent = async () => {
    await producer.connect();
    await producer.send({
        topic: "user.activity",
        messages: [{ value: JSON.stringify({ event: "auth_success", userId: "12345" }) }],
    });
};

publishEvent();
```

#### **Web-Service Publisher**  
```typescript
const kafka = new Kafka({ clientId: "web-service", brokers: ["kafka:9092"] });
const producer = kafka.producer();

const publishEvent = async () => {
    await producer.connect();
    await producer.send({
        topic: "user.activity",
        messages: [{ value: JSON.stringify({ event: "page_view", page: "/home", userId: "67890" }) }],
    });
};

publishEvent();
```

#### **Mobile-Service Publisher**  
```typescript
const kafka = new Kafka({ clientId: "mobile-service", brokers: ["kafka:9092"] });
const producer = kafka.producer();

const publishEvent = async () => {
    await producer.connect();
    await producer.send({
        topic: "user.activity",
        messages: [{ value: JSON.stringify({ event: "app_launch", userId: "78901" }) }],
    });
};

publishEvent();
```
✅ **All publishers write to the same topic.**  
✅ **Consumers process all messages from multiple services.**  

### **Consumer Code (Single Group for All Messages)**
```typescript
const kafka = new Kafka({ clientId: "activity-processor", brokers: ["kafka:9092"] });
const consumer = kafka.consumer({ groupId: "activity-group" });

const startConsumer = async () => {
    await consumer.connect();
    await consumer.subscribe({ topic: "user.activity", fromBeginning: false });

    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            console.log(`Consumed from ${topic} partition ${partition}: ${message.value}`);
        },
    });
};

startConsumer();
```
✅ **All services publish independently.**  
✅ **Single consumer group processes all activity events.**  

---

## **🚀 Summary**
| **Scenario**                                      | **Producer Strategy**                 | **Consumer Strategy**                    | **Message Flow**                         |
|--------------------------------------------------|--------------------------------|--------------------------------|--------------------------------|
| **Multiple Publishers Using the Same Producer Group** | **Same producer group, same topic** | Separate consumer groups per service | Load-balanced publishing |
| **Multiple Publishers with Different Producer Groups** | **Different producer groups, same topic** | Single consumer group | Fan-in pattern (multi-source to one consumer) |

✅ **Kafka is highly flexible! You can safely scale publishers and decide whether to use one or multiple consumer groups.**  


3️⃣ Key Features of This Implementation

Issue	Solution
Pod crashes?	Kafka assigns its partitions to another pod.
Pod restarts?	Kafka rebalances partitions across all available pods.
Prevent duplicate processing?	Offsets are committed manually after successful processing.
Ensure smooth rebalance?	heartbeat() keeps Kafka informed that the consumer is alive.


```typescript
const kafka = new Kafka({ clientId: "db-server", brokers: ["kafka:9092"] });
const consumer = kafka.consumer({ groupId: "db-server-group" });

const startConsumer = async () => {
    await consumer.connect();
    await consumer.subscribe({ topic: "auth.registered", fromBeginning: false });

    await consumer.run({
        eachMessage: async ({ topic, partition, message, heartbeat, commitOffsetsIfNecessary }) => {
            try {
                console.log(`Processing message from partition ${partition}: ${message.value}`);

                // ✅ Process message (business logic)
                await processMessage(JSON.parse(message.value?.toString() || "{}"));

                // ✅ Commit offsets so Kafka knows this message was processed
                await commitOffsetsIfNecessary([{ topic, partition, offset: (Number(message.offset) + 1).toString() }]);

                // ✅ Keep the connection alive (prevents unnecessary rebalancing)
                await heartbeat();
            } catch (error) {
                console.error(`Error processing message: ${error.message}`);
                await handleFailedMessage(message);
            }
        },
    });

    consumer.on("consumer.group_rebalance", async () => {
        console.log("Kafka is rebalancing, partitions will be reassigned...");
    });
};

startConsumer();
```
4️⃣ What If a Pod Comes Back Online Too Quickly?

💡 Problem: If a pod flaps (crashes & comes back rapidly), Kafka might rebalance too frequently, causing unnecessary downtime.
✅ Solution: Use session.timeout.ms and max.poll.interval.ms to delay immediate rebalancing.

Modify consumer settings:
```typescript
const consumer = kafka.consumer({
    groupId: "db-server-group",
    sessionTimeout: 30000,  // 30 seconds before marking pod as dead
    maxPollInterval: 60000  // 60 seconds allowed between processing messages
});
```
📌 This prevents Kafka from rebalancing too quickly if a pod restarts within 30 seconds.


### DLQ Handling
```
import { Kafka } from "kafkajs";

const kafka = new Kafka({ clientId: "dlq-consumer", brokers: ["kafka:9092"] });
const consumer = kafka.consumer({ groupId: "auth-consumer-group" });
const dlqProducer = kafka.producer();

const MAX_RETRIES = 3;
const failedMessages: Map<string, number> = new Map();  // Track retry attempts

const processMessage = async (message: string) => {
    // Simulate processing failure for some messages
    if (Math.random() < 0.3) throw new Error("Processing Failed");
    console.log(`✅ Successfully processed: ${message}`);
};

const sendToDLQ = async (message: string) => {
    await dlqProducer.connect();
    await dlqProducer.send({
        topic: "dlq.auth.registered",
        messages: [{ value: message }],
    });
    console.log(`🚨 Moved to DLQ: ${message}`);
};

const startConsumer = async () => {
    await consumer.connect();
    await consumer.subscribe({ topic: "auth.registered", fromBeginning: false });

    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            const msgValue = message.value?.toString() || "{}";

            try {
                console.log(`Processing: ${msgValue}`);
                await processMessage(msgValue);
                failedMessages.delete(msgValue);  // Reset retry count if success
            } catch (error) {
                console.error(`❌ Error processing: ${msgValue}`);

                // Track retries
                const retries = failedMessages.get(msgValue) || 0;
                if (retries >= MAX_RETRIES) {
                    await sendToDLQ(msgValue);  // Move to DLQ
                    failedMessages.delete(msgValue);
                } else {
                    failedMessages.set(msgValue, retries + 1);
                    console.log(`🔁 Retrying (${retries + 1}/${MAX_RETRIES})`);
                }
            }
        },
    });
};

startConsumer();
```
```
const dlqConsumer = kafka.consumer({ groupId: "dlq-consumer-group" });

const processDLQMessage = async (message: string) => {
    console.log(`🔄 Reprocessing from DLQ: ${message}`);
};

const startDLQConsumer = async () => {
    await dlqConsumer.connect();
    await dlqConsumer.subscribe({ topic: "dlq.auth.registered", fromBeginning: false });

    await dlqConsumer.run({
        eachMessage: async ({ message }) => {
            const msgValue = message.value?.toString() || "{}";
            await processDLQMessage(msgValue);
        },
    });
};

startDLQConsumer();
```
