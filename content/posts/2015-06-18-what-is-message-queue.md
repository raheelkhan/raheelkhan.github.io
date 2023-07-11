---
title: What is message queue
date: 2015-06-18
categories: 
---

If you have been programming in any server side language, I believe you have came up with the scenario where you make stuffs like sending bulk emails, processing some images, video conversions and other heavy lifting tasks.

If you are not aware of what message queue is and how you can use it in your day to day programming. This tutorial will give you a summarize introduction of what message queue is and how you can utilize it.

> Message Queuing (MSMQ) technology enables applications running at different times to communicate across heterogeneous networks and systems that may be temporarily offline. Applications send messages to queues and read messages from queues. – Source : MSN

What this mean is that a message queue is a mechanism where process communicates with each other. These processes can be within a same application or they can be on different application and even on a different server that are connected in network.

Typically, message queue consists of 3 entities.

1. Producer
2. Server (Message Box)
3. Consumer

### Producer

A producer is someone who has the message that needs to be send to the Message Box.

### Server (Message Box)

A storage area where messages are stored in queue typically in FIFO (First In First Out) method. This message box continuously pushes messages to the Consumer.

### Consumer

A consumer is a process that process the message.

## Example Workflow

For the purpose of simplicity lets take a real world example where you have to take out email address from a CSV file and you have to send them an email. If there tens of email address you definitely do not need a queue but what if there are millions of records? Are you going to make your user wait for long time until the process finished? You will not.

Generally you will make a producer script that fetches CSV data and pass it to the Message Box. At this stage you let your user go and relax and show him a message **Your request received**. On the other hand you will have consumer script that listens to the Message box and process the data that may arrived.

**What if connection lost or any exception occurs ?**
This is where a message queue shines. Unlike cronjobs, A message queue is reliable in a sense that there is almost no chance of loosing the data in case of any unwanted thing happens in the server.

This becomes possible because of the acknowledgement system. A message queue holds a message in box as long as it does not receive acknowledgement from consumer like **Hey, I have processed this message. You can dispose this message now**.

## Resources

While PHP offers supports for message queue by its Semaphore Functions. You are not likely going to implement in real world message queue because of its lacking in multi servers supports.

There are other third party resources that helps you build a message box quickly and leave you only to worry about the processing of message.

1. [Gearman](http://gearman.org/getting-started/)
2. [RabbitMQ](https://www.rabbitmq.com/)
