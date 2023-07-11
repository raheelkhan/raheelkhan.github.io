---
title: Speaking HTTP2
date: 2018-05-04
categories:
---

## Understanding HTTP First

Many of us who work on the business application development rarely get a chance to go deeper into the networking side of computer science. This is totally acceptable as the whole idea of software development is “Abstraction”. We tend to develop things on top of technologies and methodologies that have been previously developed and trust that something is meant to be worked as it is expected. However, It is best for application developers to understand how things work behind the scene and when to use a certain technology.

Same thing is common to see in case of “Protocols” and precisely “HTTP” for the web development ninjas.

I will try to explain HTTP first in general before we go deeper into advancement happened over the time.

**Wait ! What is protocol ?**
Imagine you got a new match on Tinder. You are super excited and decided to hangout. You decided place and timing and the day is here right now. You reached the spot and headed towards the chair where she is sitting.

What will happen now ? If not always but mostly it will be something like this

You: Bring your hand forward to shake

She: After realising that you want to shake hands, bring her hand forward

You: Hi, I am Bob

She: Hi, nice to meet you ! I am Alice

and then the conversation goes on ! Whats gonna happen next depends on how smart you are 😉

The point here is we humans, over the period of time designed some rules or patterns to communicate to each other. No matter which language you speak or which culture you belong to, there will be certain patterns that you follow in order to start conversation.

Softwares are no different. They follow the same method when expected to talk to each other. [Here](https://en.wikipedia.org/wiki/Internet_protocol_suite#History) is the history of how these protocols have evolved.

## HTTP:

Hyper Text Transfer Protocol is an application layer communication protocol that uses TCP. HTTP was designed to power the world wide web and works on a client server model. HTTP does not specify how the data should be transmitted but it specifies how the data should look like. The transmission is handled by the Transport Layer i.e TCP.

In simple words, TCP defines the flow of communication and HTTP defines the format of communication.

In the below wireshark screenshot, You can see how the connection is started by client (my computer) and accepted by server (a random website) through TCP and then followed by HTTP.

![Wireshark](/img/wireshark-sc.png)

TCP is not the only layer that HTTP can work with. UDP (User Diagram Protocol) can also be an underlying protocol for HTTP but till this day TCP is the standard because of some of its reliability features such as In-Order Delivery and Retransmission of loss packets.

However, prior to HTTP2 there had been certain drawbacks using HTTP1.\*/TCP which the web industry was facing and developed various hacks to optimise the end user browsing experience.

### Three way Handshake

Every TCP Connections starts with a three way handshake before a client or a server can pass any application data.

![Three way handshake](/img/threeway-handshake-sc.png)
Imagine a web page contains few images, Javascript files or CSS stylesheets. For every external resource the browser has to open a new connection and have to go throu

### Head Of Line Blocking

Head of line blocking refers to the scenario when a client makes several request over a same TCP connection and the first-most packet in the sequence is still in process by the server or dropped during the transmission. Due to the in-order delivery and reassuring rule of TCP, the server will resend the dropped packet, and the client will only be able to resume reading the response once it receives the dropped packet.

Another way to understand Head Of Line Blocking in HTTP context is that the average webpage these days requires from 50 to 100 request to be displayed correctly. This includes the static files, images and so on. Modern day browsers sends up to 6 simultaneous request on a single server ( TCP connection limitation ). Assuming that we have a web page of picture gallery that has 100 images. With the above HOL limitation user’s browser will only send 6 requests to download images from server. If any of image is failed to download, the whole connection will block any upcoming requests unless the image is re-transmitted from the server.

### Uncompressed Repeated Headers

In HTTP/1.1 header fields are not compressed. Keeping the above assumption of average 100 request for a single webpage, every request have more or less same headers. This redundancy consumes unnecessary bandwidth and increase latency.

### Introducing SPDY (An improved protocol with the aim to reduce page load time)

In 2009, Google introduced [SPDY](https://www.chromium.org/spdy/spdy-whitepaper) (Speedy) for their internal products. SPDY was designed to work with TCP too but with some advancement. In the coming years they made it public and was adapted by other browsers such as Mozilla and the popular web servers including Apache and NGINX. However, the team building the SPDY project later on announced to stop support of SPDY and move to [HTTP2 Draft](https://tools.ietf.org/html/draft-ietf-httpbis-http2-17) by IETF on February 2015.

## Finally HTTP/2

HTTP/2 was designed to eliminate the problems I’ve mentioned above with HTTP/1.x.

Even though the developers were inventing new hacks to optimize pages running on HTTP/1.x such as gzip compresssion, concatenation, image spriting etc, but thats just hacks.

It is worth noting that HTTP/2 doesn’t change the semantics of HTTP/1.x. We still have all the METHODS, HEADERS and so on. That is why it is called protocol upgrade.

## Improvements in HTTP/2

### Multiplexing

Even though in HTTP/1.1 we have a concept called pipelining which basically means to send multiple request over a same TCP connection. But this didn’t let us go so far because of the HOL blocking. Any request in a pipeline can be a reason to block all the subsequent requests.

HTTP/2 introduced a multiplexing which use parallel streams over the same TCP connection. The good part is these streams allow HTTP/2 to receive responses in any order. Meaning we are no longer blocked by a certain request. This parallelism gives web browsers the ability to fetch multiple resources at the same time leading to fast loading of web pages.

### Header Compression (HPACK)

As described above the redundant header fields in HTTP requests increase the bandwidth and latency. HTTP/2 brings a concept of encoding headers. It maintains a table of header fields and map it to indexed values. See the specifications of HPACK .

### Server PUSH

In a normal flow of HTTP/1.x, When a browser request for a webpage it receives the HTML content from the server. It then start rendering the page for the user. While rendering the head section normally it sees the link to CSS and JS files. At this point the browser makes another request to the server asking for the assets. If we are using Connection-Alive header in the first request we get a little benefit that the browser won’t have to go with the three way handshake rather it just uses the active TCP connection and make a request. During this time the browser is doing nothing just waiting for the assets to arrive.

Server PUSH in HTTP/2 is a feature where the HTTP server will push all the assets itself without browser requesting for it. By the time browser starts rendering the page, the assets (CSS, JS, Media) already arrived at the client end and stay at the browser buffer.

This makes the rendering of full page faster as all the assets required to display the page are already at disposal.

However, this has to be used will caution as it can flood the browser memory sending too many assets while browser has not enough capacity to handle. Also HTTP/2 implementation allows the client to restrict the server PUSH.

## Implementations

At the time of writing this article 25.7% of websites are using HTTP/2. Infact this blog hosted on WordPress is being server via HTTP/2.

![HTTP2](/img/http-sc.png)

There are also implementations of HTTP/2 available in various languages

1. [simplehttp2server](https://github.com/GoogleChromeLabs/simplehttp2server)
2. [hyper](https://github.com/Lukasa/hyper)
3. [Apache](https://httpd.apache.org/docs/2.4/howto/http2.html)
4. [NGINX](http://nginx.org/en/docs/http/ngx_http_v2_module.html)
5. [IIS](https://docs.microsoft.com/en-us/iis/get-started/whats-new-in-iis-10/http2-on-iis)

While it is still relatively new and every new technology takes time to penetrate in the sphere, It is the way to future. If you are working in web development industry chances are you already used HTTP/2 or you are going to use it in near future.

I hope this article serves a basic understanding of where we are heading to. Feel free to comment if you find any ambiguity.
