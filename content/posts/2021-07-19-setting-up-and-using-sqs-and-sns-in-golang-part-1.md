---
template: post
title: "Setting Up And Using SQS And SNS In Golang: Part One"
slug: using-sqs-and-sns-in-golang-pt-1
socialImage: /media/sheep-crowd.jpeg
draft: false
date: 2021-07-19T01:39:59.960Z
description: >
  SQS and SNS are commonly used together to make a robust one-to-many messaging
  system. This is the first post of the two-part series for setting up and using
  SQS and SNS in Golang.
category: Software, Architecture
tags:
  - SQS
  - SNS
  - AWS
---

## Prerequisites

* An AWS account.
* An access key and a secret key for AWS.
* Golang. If you don't have it installed, follow the instructions [here](https://golang.org/doc/install).

## Why SQS and SNS?

SQS stands for Simple Queue Service - a proprietary service implemented by AWS. In the programming context, a queue is something that you can push the data to, and you can also poll messages from it one by one. For example, let's say Ken left a review for a product on an e-commerce website. The app may do the following things behind the scenes. Firstly, it creates a review for this product in the database. Secondly, the app notifies the seller via email that someone has left a review for them. Optionally, if this is the first review Ken has made in his account, the app creates a badge for him. It notifies him via web push notification about the badge. As you can see, there are four possible actions the backend will proceed from one action on the client-side.

In the above scenario, **resiliency** of the app is a concern because the app will not perform the following operations if one action fails. You may say, why don't you retry the request from the client-side? It could also be problematic because that means we need to check if Ken has already created a review for the product; if so, we need to check if we have sent the email to the seller. Our code can get messy quite quickly easily.

Queue comes to the rescue. Instead of executing all actions at once, the app can push a message such as `{ event: "finsh_product_review", body: { productId: 4, reviewed: 10 } }` to the queue, and the other service will start processing after polling the message. After the other service receives the `finish_product_review` event, the service can notify the seller about the review. A queue offers **resiliency** because it guarantees the delivery of messages. Generally, a service deletes the message from the queue after a successful operation, notifying the seller about the review. However, suppose the service fails to process the message. In that case, the service will continue to process it until one of the following situations happens, as you can see from the [documentation](https://docs.aws.amazon.com/lambda/latest/operatorguide/sqs-retries.html):

> 1. The message is processed without any error from the function, and the service deletes the message from the queue.
> 1. The Message retention period is reached, and SQS deletes the message from the queue.
> 1. There is a dead-letter queue (DLQ) configured, and SQS sends the message to this queue. It's best practice to enable a DLQ on an SQS queue to prevent any message loss.

SNS stands for Simple Notification Service. It is a pub/sub messaging service where the publisher can be any service. The subscriber can range from email, SMS, lambda and SQS etc.

Now let's take a look at why we choose to couple SQS and SNS together, according to [this](https://stackoverflow.com/questions/13681213/what-is-the-difference-between-amazon-sns-and-amazon-sqs) excellent article from StackOverflow

> You don't have to couple SNS and SQS always. You can have SNS send messages to email, SMS or HTTP endpoint apart from SQS... By coupling SNS with SQS, you can receive messages at your pace. It allows clients to be offline, tolerant to network and host failures.

The combination of SNS and SQS gives you the best of both worlds because SNS allows you to publish messages to subscribers, such as mobile push and email endpoints. It will enable the same data to be processed by multiple subscribers. In contrast, a queue is suitable when only one receiver is present. Also, using SQS with SNS allows the application to receive the messages at its own pace, and you can achieve [fanout](https://en.wikipedia.org/wiki/Fan-out_(software)) pattern using the combination.

## Alternatives

SNS and SQS are not the only technologies you can use for an event-driven architecture and achieve the things we mentioned above. Other alternatives widely adopted by the industry, such as **RabbitMQ** and **Kafka**. In an upcoming article, I will give a deep comparison between SQS, SNS, RabbitMQ and Kafka. In that article, I will dive deep and explain why SQS and SNS win in particular situations and when using RabbitMQ or Kafka is better.
## Setting up SNS from the console

As mentioned above, SNS has a concept of topics where the publisher pushes the message to a topic. Then, the respective subscribers receive the messages from SNS. Let's see how to create an SNS topic.

Go to the AWS SNS page -> Click **Topics** from the left-side panel -> Click **Create topic**

![aws-sns-page](/media/aws-sns-topics-page.png)

1. Choose the **standard** type.
2. Name the topic **ProductPriceTopicDev**.
3. Name the display name **Product Price Topic Development**.
4. Leave the rest of the options as they are and click **Create topic**.

Next, we will create a subscriber for the topic, a standard SQS queue in our case.

## Setting up SQS from the console

SQS is not the only subscriber that SNS can publish a message. For example, the valid subscriber endpoints are HTTP/HTTPS, Email/Email-JSON, and AWS Lambda etc., as you can see from the official doc [here](https://docs.aws.amazon.com/sns/latest/dg/sns-create-subscribe-endpoint-to-topic.html). As mentioned above, the benefits of using SQS as a subscriber are the following:

1. Guarantee message delivery.
2. You can use a dead-letter queue to debug failed sent messages.
3. The application can poll and process the messages from SQS at its rate. If we choose not to use SQS, then SNS may push messages directly to the applications at the rate they could not handle.
4. Follow up on the point above. Since both SNS and SQS scales definitely, the combination of these technologies will be the building blocks of a scalable, performant and reliable architecture.

Here are the steps to **create a standard SQS queue**:

Go to the AWS SQS page -> Click **Create queue** -> Name the queue **ProductPriceUdpateQueueDev** -> Leave the rest of options as they are and click **Create queue**

## Caveats

Notice how the topic **ProductPriceTopicDev** and **ProductPriceUdpateQueueDev** we created was the standard type. The topic and the queue can also be FIFO. Only a FIFO queue can subscribe to a FIFO topic.

- We will use the combination of the FIFO topic and FIFO queue in the case where the order of the arrival of the messages matters. For example, for an application where the users can claim a limited prize or purchase a limited product, the "First In First Serve" principle applies to both situations. Therefore, a FIFO topic and a FIFO queue are suitable to use here.

- Talk about my naming convention of the topic and the queue. e.g. A granular topic name and a granular queue name vs a granular topic name and multiple queues vs multiple topics and multiple queues; what are the tradeoffs of each approach.

## What's next

In the next part of the series, I will explain using Golang; how to programmatically send and receive a message in a real-world scenario, where you want to receive the message from SQS in a long-running server.

Thanks for reading and see you next time.
