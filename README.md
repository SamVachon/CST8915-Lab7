# CST8915 Lab 7 — Introduction to Kubernetes Basics

**Samuel Vachon**  
**Student number — 041101891**

## Demo video

[Watch my Lab 7 demo on YouTube](https://www.youtube.com/watch?v=HKfJIGiP6GA)

## Lab overview

This lab covers creating Pods, Deployments, and Services using kubectl commands and YAML files. The Algonquin Pet Store runs on Azure Kubernetes Service (AKS), with separate components for the storefront, Product Service, Order Service, and RabbitMQ.

## RabbitMQ configuration analysis

### Is RabbitMQ stateful or stateless?

RabbitMQ is stateful because it maintains queues, messages, and broker metadata. For the Pet Store, queued messages represent submitted orders awaiting processing. Recovering this state is necessary for reliable order handling. RabbitMQ recommends persistent storage and stable node identities when deployed on Kubernetes. [1]

### What is the problem with the provided configuration?

The RabbitMQ Deployment in the lab manifest has one replica and no persistent volume mounted for its data. Its data therefore has no configured storage that survives replacement of the Pod. If queued orders disappear, the application can lose orders that customers already submitted. Kubernetes can restore the desired number of running Pods, but recovering the application's data requires persistent storage. [1][2]

### What happens when the RabbitMQ Pod is deleted or restarted?

Deleting the RabbitMQ Pod causes its Deployment to create a replacement. The replacement starts without the previous Pod's data, so the old queued messages cannot be recovered from that storage. The queue can also disappear until a client declares it again. This is the failure scenario tested by submitting orders, checking the queue, deleting the Pod, and checking the queue on the replacement.

Kubernetes restarts a failed container with a fresh writable container layer. In this manifest, there is no data volume to retain RabbitMQ's state across that container restart. Restarting only the RabbitMQ process inside an otherwise intact container is a different event, since its existing files may still be present. [2]

### Potential solutions

1. **Persistent storage and stable identity.** Mount a PersistentVolumeClaim at RabbitMQ's data directory and use a StatefulSet with stable node identity. The RabbitMQ Cluster Operator can manage this configuration. Persistent storage allows a replacement Pod to reuse its broker data. [1]
2. **Durable messaging.** Configure durable queues and persistent messages. Queue durability and message persistence work together, and both depend on retaining the broker's data storage. [3]
3. **Reliable publishing and processing.** Use publisher confirms, acknowledge messages after successful processing, and recover connections after failures. For additional availability, consider replicated quorum queues across a properly configured RabbitMQ cluster. [4]

### Would Azure Service Bus solve this issue?

Yes. Moving the messaging workload to Azure Service Bus would address the specific dependency on the RabbitMQ Pod's local data. Service Bus stores accepted messages durably outside the AKS application Pods, and Azure manages the broker infrastructure. Replacing a Pet Store Pod would therefore leave those accepted messages in the managed queue. [5]

Migration would require updating the application's messaging code and authentication. Service Bus supports AMQP 1.0, while RabbitMQ clients using AMQP 0-9-1 require a compatible replacement integration. The application would still need correct message settlement, retry handling, and appropriate message expiration settings. [5]

## Setup notes

I reused the AKS cluster from Lab 6 and used a separate `lab7` namespace. The previous Lab 6 application was still reserving CPU in the `default` namespace, which caused new Pods to remain Pending with an `Insufficient cpu` scheduling event. Removing the previous application freed resources for Lab 7.

## References

1. RabbitMQ. [Deploying to Kubernetes (Do It Yourself)](https://www.rabbitmq.com/docs/install-kubernetes-diy).
2. Kubernetes. [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/).
3. RabbitMQ. [Queues — Queue Durability](https://www.rabbitmq.com/docs/queues#durability).
4. RabbitMQ. [Reliability Guide](https://www.rabbitmq.com/docs/reliability).
5. Microsoft Learn. [What is Azure Service Bus?](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview).

[Original Lab 7 instructions](https://github.com/ramymohamed10/26F_CST8915_Lab7)
