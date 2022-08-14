---
draft: false
title: "Websockets + Python + Redis"
date: "2022-08-14T19:56:44+10:00"
tags: ["websocket", "python", "wsgi", "asgi", "asynchronous", "redis"]
showFullContent: false
hideComments: false
---

## WebSocket 101

A WebSocket is a persistent bidirectional communication channel between a client and server, operating over HTTP through a single TCP/IP socket connection. Both WebSocket and HTTP are located at layer 7 and depend on TCP at layer 4. The WebSocket is designed to work over HTTP ports to support proxies and intermediaries.

WebSocket enables a lower overhead alternative to HTTP polling, facilitating real-time data transfer from and to the server. It is a framed protocol, meaning that a chunk of data (a message) is divided into a number of discrete chunks, with the size of the chunk encoded in the frame.

To initiate a WebSocket connection, the client must first send a standard HTTP request with headings of `Connection: Upgrade` and `Upgrade: websocket`, and the server must respond with a `HTTP 101 Switching Protocols` to acknowledge that the handshake is complete. Now data can flow over this connection using a basic framed message protocol.

## Web Server Gateway Interface (WSGI)

Applications built in Python (e.g. Flask or Django) that are meant to serve as a website, are actually only application servers, and are not web servers. An application server is responsible for generating the content dynamically, and a web server is responsible for serving static pages using HTML and CSS. Application servers aren’t guaranteed to be optimised for performance and haven’t been built with security in mind.

WSGI is an agreed upon standard interface that various Python applications can implement. The interface has two sides, the server/gateway side which is responsible for invoking the application callable for each HTTP request, and the application/framework side which is responsible for creating the application object. A WSGI server will invoke a callable object on the WSGI application as defined by the PEP 3333 standard (Python Web Server Gateway Interface v1.0.1).

It's flexible in that application developers can easily modify components and change which implementation of WSGI they want to use without changing the actual application code itself. WSGI is meant for scale, it's designed to handle requests from the web server and deciding how to communicate those requests to the application framework. Serving thousands of requests is the domain of the WSGI servers, not the frameworks.

Read more here: [PEP 3333](https://peps.python.org/pep-3333/)

## Asynchronous Server Gateway Interface (ASGI)

This is the successor of WSGI. WSGI applications are a single, synchronous callable that takes a request and returns a response; this doesn’t allow for long-lived connections that you get with long-poll HTTP or WebSocket connections.

ASGI is a single, asynchronous callable. It takes a `scope` (containing information about the specific connection), a `send` (asynchronous callable, letting the application send event messages to the client, and `receive` (asynchronous callable letting the application receive event messages from the client). This allows for multiple incoming events and outgoing events for each application, while also allowing background coroutines so the application can do other things (e.g. listening for an external trigger such as a Redis queue).

Every event that is sent or received is a Python `dict`, with a predefined format. These events have a defined `type` key which is used to infer the event’s structure. For HTTP, this is as simple as `http.request` and `http.disconnect`. For something like a WebSocket, it’s receiving `websocket.connect`, sending a `websocket.send`, receiving a `websocket.receive`, and finally receiving a `websocket.disconnect`. Users are also free to create their own message types and send them between application instances for high-level events (e.g. `chat.message`).

Read more here: [ASGI Specification](https://asgi.readthedocs.io/en/latest/specs/index.html)

## Redis

Redis is an in-memory data store (in-memory = RAM = blazingly fast), commonly used as a database, cache, streaming engine, and message broker.

Redis relates to ASGI and WebSockets by enabling multiple consumers to talk to each other. This is used as a backend to the already running ASGI server. Without such a backend, clients or consumers can only communicate with themselves.

Redis has an implementation of the common publish/subscribe message paradigm where the publishers send messages to channels (instead of specific subscribers), without the knowledge of if there are any subscribers on the channel. Subscribers can then subscribe to one or more channels without the knowledge of the publishers.

As an example to integrate all of the above, let's say there's a ASGI web server, handling WebSocket connections that also connects to a Redis backend. Given two clients connect to this web server, on establishing the connection, they both subscribe to a channel called "chatmessage". Now whenever one of the clients sends a message to the server, the server can publish that message to "chatmessage" and the other client will receive it.
