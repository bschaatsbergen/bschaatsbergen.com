---
title: "You’ll run out of money before you run out of space"
date: 2022-01-15T10:59:15+00:00
tags: ["Object Storage", "System Design", "Distributed Systems"]
---

# 100 trillion objects

That's how many objects S3 managed on its 15 year anniversary. Quite impressive, right?
In this article I aim to shed a bit of light on object storage architecture (very high level) and why cloud providers state that you can store a unlimited amount of objects on their platform.

# What is object storage?

Object storage systems, put data into objects, discrete ‘containers’ that each have a unique identifier, called an object ID (OID). Every object is also associated with metadata.

Object storage does not use folders, directories or complex hierarchies. The object itself is useless without the metadata, the metadata is describing important details such as permissions, type of encryption, contingencies, and other information. As every piece of data together with its metadata becomes a so called self-contained 'object' we can store it anywhere and retrieve it from anywhere.

The most common object storage architectures contain 2 main components: routers and storage nodes.
You can think of a application and storage layer, where the router is part of the application layer and the storage nodes are part of the storage layer. The application layer contains logic to handle the read/write requests to the storage layer, but it also manages permissions, etc. Throughout this article the router should be seen as part of the application layer.

![Storage Router and Nodes](/storage-route-and-nodes.png)

As a object storage user you interact with the router. The object store exposes a REST based interface to communicate with. Lets take AWS S3 its `PutObject` request for example.

```bash
PUT /gizmo.jpg HTTP/1.1
Host: bschaatsbergen.s3.us-east-1.amazonaws.com
Date: Sat, 15 Jan 2022 17:50:00 GMT
Authorization: authorization string
x-amz-grant-write-acp: id=8a6925ce4adf588a4532142d3f74dd8c71fa124
x-amz-grant-full-control: emailAddress="gizmo@bschaatsbergen.com"
x-amz-grant-write: emailAddress="gizmo@bschaatsbergen.com"
Content-Type: text/plain
Content-Length: 11434
x-amz-server-side-encryption-customer-key:g0lCfA3Dv40jZz5SQJ1ZukLRFqtI5WorC
x-amz-server-side-encryption-customer-key-MD5:ZjQrne1X/iTcskbY2
x-amz-server-side-encryption-customer-algorithm:AES256
Expect: 100-continue
[11434 bytes of object data]
```

You now might wonder how the router is able to keep track of data across all these storage nodes, not a weird question. The router keeps track of every forward it does to a specific storage node. The logic that orchestrates these read/write requests can e.g. be [consistent hashing](https://bschaatsbergen.com/consistent-hashing/).

For example, we could make use of a hashing algorithm that returns a number between 1 and 100 for every object based of its name. We divide our storage nodes over the 1 - 100 range and evaluate every object that passes through the router.

Lets say we want to store a picture of my cat (Gizmo) in our object store, the process would look as following:

![Routing gizmo.jpg](/router-route-gizmo.png)

Similiar to writes, if we would want to retrieve the object `gizmo.jpg` the router would again evaluate the object by name and route us to the '1-33' storage node.

Now it's obviously not so black and white where we simply assign a number to an object, this was just an example. 'In the real world' there's much more logic involved, think of balancing the nodes and letting the router now we moved objects from one storage node to another, etc.

# Infinite scaling

I absolutely love how cloud providers say the following about their object storage capabilities:

> "The total volume of data and number of objects you can store are unlimited."

That's quite a bold thing to promise to your end users, you can only do so when you possess massive scale. I can't even imagine the millions of disks that are in use to host these 100 trillion objects in Amazon S3.

The crux in object storage architecture is designed on purpose to support and operate at such scale, separation of the router and storage nodes. To scale, you simply add more nodes in parallel to an object storage cluster for additional processing.

Fyi, 'in the real world' persisting a specific object to disk isn't just to a single storage node. It might be that the layer where the router lives slices the object and distributes individual slices of data over multiple storage nodes, which again also replicates the slices of data to other storage nodes (just for the sake of durability of the object).

Anyways, after this short introduction to object storage in the cloud I think that we can both agree that a user could never reach the limit of objects stored in a Amazon S3 bucket. One thing is for sure, you can't compete with cloud providers.

# Footnotes

- If you plan to dive into object storage I would recommend reading up on [Colossus](https://cloud.google.com/blog/products/storage-data-transfer/a-peek-behind-colossus-googles-file-system)
    - [Colossus paper](https://static.googleusercontent.com/media/research.google.com/nl//archive/gfs-sosp2003.pdf)

- Unfortunately AWS is not open about the internals of S3, but I can tell you for sure that it looks a lot like how DynamoDB is designed.
    - [DynamoDB paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

- A piece I wrote on [consistent hashing](https://bschaatsbergen.com/consistent-hashing/) might be interesting to you now?