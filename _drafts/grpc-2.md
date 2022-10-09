---
layout: post
title:  "Part 2: gRPC encoding"
series: "gRPC for Microservice communication"
author: rowan
categories: [ grpc, php, typescript ]
#tags: []
#image: ""
description: ""
comments: false
permalink: /grpc-for-microservice-communication/2-grpc-binary-encoding
---

In the [first article](/grpc-for-microservice-communication/1-grpc-vs-rest) in this series, we saw how removing redundant metadata in a message format can significantly reduce the payload size. In that article we created our own custom message protocol, that still used text encoding. In this article, we'll discuss how we use binary encoding instead, in a manner similar to [gRPC](https://grpc.io/). 

Understanding how binary encoding works is actually not necessary for day to day use. At Sidekicker, our [gRPC integration with Typescript and PHP](/grpc-for-microservice-communication/3-typescript-php-integration) allows engineers to code, without having to worry about the underlying network communication.  

To begin with, let's do a super quick introduction (or refresher) on bits, bytes and binary. If you already have a good understanding of his, you may want to jump over to the [official Protocol Buffers documentation on encoding](https://developers.google.com/protocol-buffers/docs/encoding) and read those.

In `binary` (`base2`) numbers are represented using only `0` or `1`. Thus:

| Decimal | Binary |
|---------|--------|
| 0       | 0      |
| 1       | 1      |
| 2       | 10     |
| 3       | 11     |
| 4       | 100    |
| 5       | 101    |
| 6       | 110    |
| 7       | 111    |

A `bit` is the smallest data size. It can have two values either `0` or `1`. To store a boolean value, for example, you only need 1 bit. If you have `2` bits you can represent one of 4 values:

```
00 = 0
01 = 1
10 = 2
11 = 3
```

A `byte` is made up of eight bits. So a single byte can be any value between `0` and `255` (i.e. `2^8 = 256` minus 1, because we start at zero): 

```
0000 0000 = 0
0000 0001 = 1
0000 0010 = 2
...
1111 1101 = 253
1111 1110 = 254
1111 1111 = 255
```

If you use `2` bytes, you can therefore represent `256^2 = 65,356`.

So if we use `750` as an example, this means you need two bytes. Byte `A` should be `2` or `0000 0010` to represent `2 x 256 = 512`. Byte `B` is required to represent `750 - 512 = 238` or `1110 1110`.

You would normally write this as `0000 0010 1110 1110`, with the 

```
            First byte      Second byte
Binary      0000 0010       1110 1110
Decimal     2               238
            2 x 256
            512           + 238         =   750
```


If you are using [ASCII](https://en.wikipedia.org/wiki/ASCII) to encode a string, then you need 1 byte per character. For example, to encode the word `hello` you need 5 bytes. The first byte is would be set to `104` to represent `h`, the second to `101` to represent `e` and so on:

| Char | Decimal | Binary    |
|------|---------|-----------|
| h    | 104     | 0110 1000 |
| e    | 101     | 0110 0101 |
| l    | 108     | 0110 1100 |
| l    | 108     | 0110 1100 |
| o    | 111     | 0100 1001 |


So let's refer back to our custom protocol in the previous article. Remember, we came up with this string to request the list of our birthday users:

```
1,12022-09-10

where

1           is the request are sending (RequestBirthdayUsers)
,           delimiter for the end of the field
1           we are about to send the date field
2022-09-10  the date value we want to search for
```

Now, that date value is encoded as a 10 character string, which requires `10` bytes. We can instead convert it to a [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time), which gives us `1662768000`<sup>1</sup>. Unix timestamps are 32-bit, which means we only need `32 ÷ 8 = 4 bytes`.

For the moment, let's assume that the first character of our message format is always the service that is being called, so we can drop the `,` character. This means our request is now:

| Type    | Data       | As byte                                 |
|---------|------------|-----------------------------------------|
| Service | 1          | 0000 0001                               |
| Field   | 1          | 0000 0001                               |
| Value   | 1662768000 | 0110 0011 0001 1011 1101 0011 1000 0000 |

Which we could send as stream of `8` bytes.

```
Data    1           1           166276800
Bytes   0000 0001   0000 0001   0110 0011 0001 1011 1101 0011 1000 0000
```

This looks ok, but we want to be able to send different messages to our server, some with multiple or even optional fields and values. Let's consider the _service type_ and _field type_ bytes. By using a single byte for each data value:

- We are limiting ourselves to a maximum of 256 services or fields<sup>2</sup>.
- If we are only using 1 service or field, we're wasting 7 bits of that single byte<sup>3</sup>.

Another issue occurs when we need to support _values_ that have variable length (e.g. a string). Let's imagine a different service (`2`) for setting a product SKU (`1`) and name (`2`). We want to set the SKU to `C82` and name to `Hat`.


| Type    | Data | As byte                       | ASCII codes |
|---------|------|-------------------------------|-------------|
| Service | 2    | 0000 0001                     |             |
| Field   | 1    | 0000 0001                     |             |
| Value   | C82  | 0100 0011 0011 1000 0011 0010 | 67 56 50    |
| Field   | 2    | 0000 0001                     |             |
| Value   | Hat  | 0100 1000 0110 0001 0111 0100 | 72 97 116   |


As our server reads the data from left to right, how does it know if:

- SKU is `C82` and name is `Hat`; or
- SKU is `C8` and name is `2Hat`

So we need a way to delimiter when each piece of data (service, field, value) ends. In the original text encoded format used a comma character. We could do the same again, inject a value to delimit. We'd just need to use a value that won't ever clash with actual data (say hello [Null character](https://en.wikipedia.org/wiki/Null_character)). That's a bit wasteful though.

###### Introducing Varints

.....







<sup>1</sup> This is actually more precise than we need, as it measures in seconds.

<sup>2</sup> You might argue that a microservice with over 256 message types isn't really _micro_.

<sup>3</sup> In this contrived example, of a single daily _birthday email_ request, 7 bits is not a lot. However, in another environment with millions of messages, this can build up.  








