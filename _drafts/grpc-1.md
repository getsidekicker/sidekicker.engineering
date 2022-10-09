---
layout: post
title:  "Part 1: gRPC vs REST"
series: "gRPC for Microservice communication"
author: rowan
categories: [ grpc, php, typescript ]
#tags: []
#image: ""
description: ""
comments: false
permalink: /grpc-for-microservice-communication/1-grpc-vs-rest
---

As our user base grows at Sidekicker, we are progressively migrating from a single monolithic application to a [microservice architecture](https://cloud.google.com/learn/what-is-microservices-architecture). This will allow us to horizontally scale during periods of heavy demand.

When designing a microservice architecture it becomes important to consider how data is transmitted between individual services.

Providing API access between our microservices in a fast and efficient manner is crucial for both performance and costs reasons. Rather than really on a traditional REST API, we are rolling out [gRPC](https://grpc.io/) for inter-service communication.

In this series of articles, we'll be covering:

1. How gRPC provides performance improvements over a REST API (this article)
2. [How gRPC encodes messages in a binary format](/grpc-for-microservice-communication/2-grpc-binary-encoding)
3. [How we integrate gRPC into our PHP and TypeScript codebase](/grpc-for-microservice-communication/3-typescript-php-integration)
4. [How we deploy gRPC microservices](/grpc-for-microservice-communication/4-grpc-infrastructure) 

To begin with, let's compare gRPC and REST. At Sidekicker we use PHP and TypeScript, but as we are only covering data transmission, this article is programming-language-agnostic.

First we'll look at how a normal REST API would send data, then cover some hypothetical ways we could improve this message format. Lastly, we'll see how our improvements align with what gRPC does.

To help us understand the difference, let's work through a hypothetical use case, where we want to send a happy birthday email to all our users. Imagine we want to make a microservice that fetches the relevant data from our existing monolithic app and send out the emails. We have full control of both the client and server, so we can modify the request and response as we desire.

Each day, we need to fetch the relevant data for each user with a birthday:

- First name
- Last name
- Email address
- Account ID

How would we fetch this data in REST? The simplest method would be to have query parameter on the `users` resource. We can then fetch a list of applicable users. We'll assume our REST API server returns the required fields that we need (i.e. we don't need to fetch `/users/1`, `/users/2` and so on). We can just request:

```
https://api.example.com/users?birthday=2022-05-1
```

REST APIs transmit over HTTP (which is plaintext) with the data JSON encoded. When we send the request for the current birthdays, something like this is what gets sent to the server inside the actual network packet.

```
GET /users?birthday=2022-09-10 HTTP/2
```

That is the minimum data required as per the HTTP specification. It's likely though that you will need to send additional header fields. If the server is hosting multiple websites, you will probably need a `Host` header, to specify the domain. Additionally, the server may provide different types of response formats (HTML, JSON, XML), so an `Accept` header may be required.

So in the example below we are telling the server we want to execute the request against a specific domain (`api.example.com`) and that we want the returned data to be JSON encoded.

```
GET /users?birthday=2022-09-10 HTTP/2
Host: api.example.com
Accept: application/json
```

In this example, we are assuming we have control over both server and client, so we can ensure that the server respects the `Accept` header. Of course, when you are dealing with a third-party API, it could just ignore the `Accept` header and just return HTML regardless. Alternatively, it may read the `Accept` header, consider JSON unsupported and throw a 400 error.

The key takeaway here, is that because the HTTP web server can be used for multiple purposes, our requests have to be verbose to ensure we get the data we want. Furthermore, everytime we run the request, we need to include these headers to ensure consistent responses.

This request is 86 characters of data (including line breaks), that is encoded using ISO-8859-1 and thus will be 86 bytes of data. Examining [UTF in HTTP headers](https://www.jmix.io/blog/utf-8-in-http-headers/) is left as an exercise for the reader.

Moving on, let's have a look at an example response. We'll assume there are 5 users with birthdays. The raw HTTP plaintext response body would look like the below. The server has been kind enough to minify the JSON data for us. A real-world API is likely to have considerably more headers.

```
HTTP/2 200 OK
Content-Type: application/json; charset=UTF-8
Date: Sat, 10 Sep 2022 02:51:34 GMT
Cache-Control: max-age=3600
X-Powered-By: PHP/8
Server: Apache

[{"id":5,"firstName":"😁Alice","lastName":"Green","email":"alice@hotmail.com"},{"id":45,"firstName":"Bob","lastName":"Red","email":"bob@yahoo.com"},{"id":68,"firstName":"Clare","lastName":"Blue","email":"clare@gmail.com"},{"id":235,"firstName":"Daniel","lastName":"Purple","email":"daniel@hotmail.com"},{"id":7532,"firstName":"Elizabeth ","lastName":"Yellow","email":"elizabeth@yahoo.com"}]
```

The first line is the HTTP version, status code and message. We already know the HTTP version we requested, but we can use the status code and message for our own error handling. Let's examine the HTTP headers. 

- `X-Powered-By` and `Server`, are not required for our use case. The server changing to `nginx` or PHP being bumped a version should (theoretically) not affect how we process the response.
- `Date` and `Cache-Control` headers, at first glance, could be useful. However, the task we are running occurs once a day. We don't want to cache the values and use them again later (tomorrow), nor do we care so much about the specific time the response was generated.
- `Content-Type` is useful, because it advises us that the returned data is in the format we wanted. It also lets us know that the body of the response has been UTF-8 encoded. This is important, as the `firstName` of the first user contains a unicode character. Depending on which characters are sent, the number of bytes to represent them will increase. So while `A` requires 1 byte, 😁 requires 4 bytes.

For simplicity, we'll remove the emoji and consider all data is ASCII. We'll also assume our server only sends the minimum headers. This would reduce our example response to:

```
HTTP/2 200 OK
Content-Type: application/json

[{"id":5,"firstName":"Alice","lastName":"Green","email":"alice@hotmail.com"},{"id":45,"firstName":"Bob","lastName":"Red","email":"bob@yahoo.com"},{"id":68,"firstName":"Clare","lastName":"Blue","email":"clare@gmail.com"},{"id":235,"firstName":"Daniel","lastName":"Purple","email":"daniel@hotmail.com"},{"id":7532,"firstName":"Elizabeth","lastName":"Yellow","email":"elizabeth@yahoo.com"}]
```

So our minimal REST API request and response sizes are:

```
Request  = 86  chars = 86 bytes
Response = 436 chars = 436 bytes
```

How can we improve on this? Given that we have full control of both client and server, we can change the request and response messages to whatever we want.

Let's look at the response _body_ first, which, as minified JSON, takes up `387` bytes.

```
[{"id":5,"firstName":"Alice","lastName":"Green","email":"alice@hotmail.com"},{"id":45,"firstName":"Bob","lastName":"Red","email":"bob@yahoo.com"},{"id":68,"firstName":"Clare","lastName":"Blue","email":"clare@gmail.com"},{"id":235,"firstName":"Daniel","lastName":"Purple","email":"daniel@hotmail.com"},{"id":7532,"firstName":"Elizabeth","lastName":"Yellow","email":"elizabeth@yahoo.com"}]
```

A lot of that message body is taken up by the field names. If we were prepared to write our own parser, we could come up with a more efficient data format for the server to send.

To start with, lets replace all those field names with numbers. So the response body becomes:

```
[{1:5,2:"Alice",3:"Green",4:"alice@hotmail.com"},{1:45,2:"Bob",3:"Red",4:"bob@yahoo.com"},{1:68,2:"Clare",3:"Blue",4:"clare@gmail.com"},{1:235,2:"Daniel",3:"Purple",4:"daniel@hotmail.com"},{1:7532,2:"Elizabeth",3:"Yellow",4:"elizabeth@yahoo.com"}]
```
That's not bad, we've reduced `387` bytes to `247` bytes. We just need to keep a mapping somewhere in our code about which number maps to which field:

```
id: 1
firstName: 2
lastName: 3
email: 4
```

We can go further and remove the JSON object, array and string quotes. We can inline everything and use a comma to denote the end of strings.

```
1:5,2:Alice,3:Green,4:alice@hotmail.com,1:45,2:Bob,3:Red,4:bob@yahoo.com,1:68,2:Clare,3:Blue,4:clare@gmail.com,1:235,2:Daniel,3:Purple,4:daniel@hotmail.com,1:7532,2:Elizabeth,3:Yellow,4:elizabeth@yahoo.com
```

Down to `205` bytes. Those `:` after the field numbers are not necessary anymore. We can assume the first character after a comma is a field number. So we can send:

```
15,2Alice,3Green,4alice@hotmail.com,145,2Bob,3Red,4bob@yahoo.com,168,2Clare,3Blue,4clare@gmail.com,1235,2Daniel,3Purple,4daniel@hotmail.com,17532,2Elizabeth,3Yellow,4elizabeth@yahoo.com
```

The above is a bit difficult for humans to understand (especially without access to the mapping file), but it's more data efficient. We are now at `185` bytes, less than half the original `387` bytes.

At this stage, you might think, why not go further and remove the field numbers altogether? The data is in a fixed format anyway.

So let's imagine that our users can select up to 3 favourite animals in their profile (who knows why). We want to include that in this response. Some users will have none, some will have 1, some will have multiple. So let's expand our mapping file. We'll add a `many` keyword in our mapping for animals.

```
id: 1
firstName: 2
lastName: 3
email: 4
many animals: 5
```

Let's update the response, with Alice having `dog` and Bob having `cat` and `fish`:

```
15,2Alice,3Green,4alice@hotmail.com,5dog,145,2Bob,3Red,4bob@yahoo.com,5cat,5fish,168,2Clare,3Blue,4clare@gmail.com,1235,2Daniel,3Purple,4daniel@hotmail.com,17532,2Elizabeth,3Yellow,4elizabeth@yahoo.com
```
We can now send these additional values when necessary and omit them for users that don't have favourite animals.

When parsing the data, we can use the convention that a field number `1` indicates the start of a record. So we'll keep the field numbers for now. Back to our original example.

###### What about the headers? 

If we were to move away from using HTTP and write our own protocol, could we save even more space?

If we run our server on its own process (i.e. a different port to that used by HTTP) we don't need worry about `Host` or `Accept` headers. We know that clients only want data from our server in the format that the server specifies.

So, once we connect to the server, we only need to tell it which specific request we are making and what arguments to execute it with. We could make a mapping file on the server that looks like this:

```
RequestBirthdayUsers: 1 {
  date: 1
}

ResponseBirthdayUsers: 1 {
  id: 1
  firstName: 2
  lastName: 3
  email: 4
}
```

Then our `request` could go from this:

```
GET /users?birthday=2022-09-10 HTTP/2
Host: api.example.com
Accept: application/json
```

to just this:

```
1,12022-09-10
```

Our server reads the first value and treats it as the method to call (`RequestBirthdayUsers`). Then it reads the following values as arguments to be passed. The whole request body goes from `86` bytes to just `13`.

As the server and client now share the mapping file, the `response` does not need any header values. It can simply return our encoded data only:

```
15,2Alice,3Green,4alice@hotmail.com,145,2Bob,3Red,4bob@yahoo.com,168,2Clare,3Blue,4clare@gmail.com,1235,2Daniel,3Purple,4daniel@hotmail.com,17532,2Elizabeth,3Yellow,4elizabeth@yahoo.com
```

So to compare against our original REST example:

|            | REST   | Customised   |                  |
|------------|--------|--------------|------------------|
| Request    | 86     | 13           | ~85% reduction   |
| Response   | 436    | 187          | ~60% reduction   |

Those reductions are pretty good. We are still sending plaintext data, but because the server and client have pre-shared knowledge of the message format we are able to trim a lot of fat from the message.

###### So this is what gRPC does?

As a starting point, yes. gRPC uses [Protocol Buffer](https://developers.google.com/protocol-buffers/docs/proto3) files (like our mapping data mentioned above) to pre-share the message format between client and server. It saves a lot of bandwidth by not transmitting redundant metadata on every single request. However, gRPC goes further by binary-encoding messages (which we cover in the [next article](/grpc-for-microservice-communication/2-grpc-binary-encoding) in this series).

###### What are the drawbacks?

gRPC, being a message service, is designed for communication between microservices. gRPC is not suitable for direct interaction with a JavaScript frontend. You'll still need a REST (or graphql) API for those. At Sidekicker, we use the [NestJS](https://nestjs.com/) framework to provide both gRPC and graphql connectivity. You can read more about our tech stack in [part three](/grpc-for-microservice-communication/3-typescript-php-integration) of this series.

Lastly, gRPC uses its own protocol, it can't be transmitted over a normal HTTP connection. This means you will require a dedicated gRPC service listening on its own port. This leads to all the related infrastructure changes will need to support this (firewall rules, security considerations, additional virtual machines). Check out our [last article](/grpc-for-microservice-communication/4-grpc-infrastructure) on how we deploy gRPC in our tech stack.