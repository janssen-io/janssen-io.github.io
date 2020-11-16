---
title: "Dynamics 365: Web Resource communication"
layout: post
excerpt_separator: <!--more-->
category: Software Design
tags:
    - Javascript
    - Dynamics 365
    - Architecture
---

In Dynamics 365 you can customize the forms used for record creation and updating. One of those customizations is adding custom web resources: html files that add functionality such as uploading files to Azure Blob storage or display PDFs inside the form.

These web resources are loaded into their own iframe. One way of communication between these web resources is to find out which iframe it's in and then call a method on its window. But that's a lot of work and it doesn't seem very reliable.

Another way is to add in a component on the top window that can support this communication, for this we can use the publisher-subscribe pattern.

## Publisher-subscriber
This top level component can easily be found by any web resource. After this component is registered, any other resource can ask to be subscribed to specific messages and also send those specific messages.

For example, we have a component that allows uploading files to Azure Blob storage and we want to disable it when the status of the record is inactive. 

First, in our Upload to Blob resource, we subscribe to the message `disableUpload`. Then from our form resource we check the status, if it's inactive we send the message `disableUpload` to our pubsub component. The pubsub component will forward it to all subscribers, in this case just our Upload to Blob resource.

This interaction then looks like this:

![pubsub sequence diagram](/assets/2020-11-08/pubsub.png)

## Firing blanks
Unfortunately, when we implemented this the first time it didn't work. The form fired the message but there was no receiver yet!
Back to the drawing board and luckily the solution was easy, because we could reuse our mechanism to inform the form that the resource was ready! We define another message, `uploadToBlobReady` and attached the form as a listener. Then when the form receives this message, it can reply by sending the `disableUpload` message.

Wiring this all up, it looks like this:

![improved pubsub sequence diagram](/assets/2020-11-08/improved-pubsub.png)

## Alternative solutions
Alternatively, we could just register some functions on the top window that each resource could call. But this could lead to clashes: what if two resources want to define a function with the same name? Also, what do we do if a funtion is not defined yet, but we expected it to be? 

With the publisher-subscriber pattern we can subscribe to a message __and__ send the message at the same time. In case the receiver is not yet listening, we will receive a message when it is listening and resend the message. If the receiver was listening, we probably missed the 'ready' message, but we can send our own message.

This pubsub component can also add logic to route the messages differently, retry sending messages if there is no listener yet or validate parameters. 

I think these possibilities make this pattern a great solution for our problem. What do you think? Leave a comment down below!

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
