---
layout: post
title:  "Part 3: TypeScript and PHP Integration"
series: "gRPC for Microservice communication"
author: rowan
categories: [ grpc, php, typescript ]
#tags: []
#image: ""
description: ""
comments: false
permalink: /grpc-for-microservice-communication/3-typescript-php-integration
---

at sidekicker we use grpc to communicate between micro services

have a look at our previous two articles about what makes grpc such an efficient choice

Implementation is officially supported by Google for a range of languages.
Other languages have 

at sidekicker we have a legacy monolithic PHP app, as well as a range of new typescript microservices

grpc uses proto files, which only define the message format



the benefit here is that our engineers only need to focus on the message format (proto)

they then can simply code against the generate classes

the message encoding/decoding is all handled in the background, allowing engineers to focus on business problems

we use the protobuf tool to generate both PHP and TS code

the general flow:

- define a new message request / response for a feature
- run through proto lint -> make sure best practices are set, we reject git commits that don't pass
- we then have a script 'generate'
  - removes the generated grpc code from both php and ts code bases
  - re-generates from the latest code base
  - currently we commit the PHP generated code, but not TS
    - TS is built as part of our deployment
- we now have generated php and TS classes to code against

in our next article we'll talk about the underlying infrastructure we use to deploy grpc and our micro servces