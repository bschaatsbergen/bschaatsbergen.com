---
title: "Consistent hashing"
date: 2022-06-03T10:59:15+00:00
tags: ["System Design", "Distributed Systems"]
---

# Slicing and dicing data

There's many ways to distribute our data over multiple nodes. Some prefer to partition their data vertically
instead of horizontally and vice versa. Lets briefly explore the difference to give some of our readers that are new to these concepts an idea of what we are talking about.

## Horizontal partitioning

When you are partitioning a table horizontally we typically refer to the art of dividing a table into
subsets of rows that are stored in seperate nodes. As visualized below, a table is divided in 2 subsets with each a contiguous range of keys.
1 to 50 is stored on Node A and 50 to 100 is stored on Node B.

![Horizontal partitioning](/horizontal-partitioning.png)

Partitioning our data over multiple nodes is useful as we have to worry less about running out of
disk space. As some data is now only available in Node A instead of Node B we also will have to distribute our load over
the nodes we store our data on.
This reduces contention and improves performance. Horizontal partitioning is also used in the design concept
of sharding.

## Vertical partitioning

Instead of looking at rows we can also look at columns. Lets say that the data under column B and C is
accessed more frequently. And D and E contain hashed sensitive information that are only in the database
for auditing purposes (we are not really interested in this data), though they are related to each other. It
would make sense to move column D and E with its
respective keys to another database node in order to reduce the I/Ops and performance overhead when
querying the frequently accessed data. Our sensitive data can now be stored in a seperate partition with
additional security measures applied to it.

![Vertical partitioning](/vertical-partitioning.png)

To sum it up, vertical partitioning focuses on dividing a table into multiple tables that contain fewer columns which in return are distributed over multiple nodes.

# Hashing

Hashing is the process of converting input to a sequence of characters by using a mathematical formula. Hash algorithms use the binary code of the given input instead of the actual string. The hash function performs logic in order to mix and stir the binary code with some other characters and returns us a consistent result. As there's many more possible inputs than outputs we might get a result that collides with one another.

## Dealing with collision (Hash map)

We pointed out that hash functions can cause collision. We should deal with this as otherwise we would get unexpected behaviour. One of the data structures that's suitable for dealing with collision is a hash map. The hash map is an associative array with a linked list per index of the array, sounds abstract right? Ideally when a hash function returns an index we should be able to retrieve and store it under index of the associative array.

![Hashmap](/hashmap.png)

The idea of using a hash map is that everytime we face collision we insert the item in a new subsequent
node in a linked list. This way all of the colliding items for a given index are stored in their own
linked list.

Suppose we have 3 strings: "Bruno", "Bob" and "Brian" that we want to store in our hash map. The first
time our hash function returns index 3.
We look in the array and see that [3] is empty, we store the key and value "Bruno" in the
array. The second time around we hash "Bob", again our hash function returns index 3.
We look in the array and see that [3] contains a value, we store the key and value "Bob" in the first
node of the linked list that's attached to the index of the array. Again, we call our hash function with
"Brian" and the response is index 3.
We look in the array and see that it contains "Bruno", we look in the first node of the linked list and
see that it contains "Bob". We use the pointer from the first node that points to the
subsequent node and store "Brian" there.

Now anytime we want to look up "Brian", we would have to go
through each node in the linked list in order to reach "Brian". Making this data structure O(N/k) where K is the number of linked lists.

# Load balancing

Distributing our data over multiple nodes requires us to balance the requests over the nodes. We don't
want to route requests randomly as it will turn out to be unbalanced and one node might receive more
requests than another. One way to go about this is to implement a round robin algorithm and let every
request go to the subsequent node in the cluster at a time.
This would work fine for writing data but is completely illogical for querying data.

So how could we solve this? What if we assign numbers to nodes by hashing the key and applying a modulo on the hash result
with the amount of nodes in the cluster: (h(x) % m where m is the amount of nodes). This would always
return us a consistent number, the number refers to the node that we should target.

```go
func hash(key string) uint32 {
	hash := fnv.New32a()
	hash.Write([]byte(key))
	return hash.Sum32()
}

func getNode(key string, nodes uint32) uint32 {
	node := hash(key) % nodes
	return node
}

func main() {
	node := getNode("Bruno", 25)
	fmt.Print(node) // returns 11
}
```

In the preceding image we created 2 functions: one to hash a key and return something between 0 and the
max of an unsigned 32 bit integer. And one to apply the modulo on the result of that hash
function in order to get a consistent integer. In the main function we call `getNode()` with
"Bruno" and 25 as parameters,
the result is 11.
This indicates that node 11 would be the target node for reading and writing "Bruno" to. This is a nice
and clean way to always get the target node depending on the key we pass down to the request.

Every node will now be responsible for a range of results coming from the hash function. "Bruno" results
in 11, "John" results in 16, "Kate" results in 18, and "Lisa" results in 1.

Though there's something that's bugging me.
What if we lose a node because it crashed, with 25 nodes "Bruno" was 11. With 24 nodes "Bruno" is 7. Our
query against node 7 would return a miss and all the mapping of keys are now invalid. Every item in the cache would have to be
invalidated and retrieved from the source. If we had applied this logic to a data source instead of
a cache we would have to deal with a scheduled downtime in order to reshuffle all the data around the
cluster. As the amount of nodes in the cluster grows linearly the amount of work required becomes unsustainable.
How can we solve this problem?

# Consistent hashing

With consistent hashing, we need to be independent from the amount of nodes we're working with.
An academic paper from Karger et al. at MIT was published that goes over this solution (see footnotes).
Consistent hashing is an algorithm that operates by positioning nodes and keys on an abstract
circle. Lets say we take our above example and map it to edges on a ring, for the sake of this example our
abstract circle starts with 0 and ends with 50.

![Hashring](/hashring.png)

Now that all the keys are mapped to edges of the abstract circle we can also do the same for the nodes
(lets take 4 nodes as an example). It's a good practice to hash the IP address of the node with the same algorithm as we'd do with any other key to come up with its angle.

![Hashring with nodes](/hashring-with-nodes.png)

The algorithm to find the target node is fairly simple. Both the keys and nodes are now mapped to
positions on the abstract circle. Lets take 2 keys, "Kate" and "John". If we wanted to get
the target node for "Kate" and "John" we would use (h(x) % m) to get their positions on the hash ring. Now
that we have a key its position on the ring we could search for the target node. We do so by traversing the
abstract circle counterclockwise in an ascending direction. As we move our way around the circle we will
come across a node, "Kate" and "John" their first node to meet would be Node B. In this case we assign the
read/write requests for "Kate" and "John" to Node B. Theoretically, each node is
responsible for handling a range of the hash ring.

![Hashring with node map](/hashring-with-node-map.png)

Now that we have a basic understanding of consistent hashing lets explore why it's better and more fault
tolerant when it comes to adding or removing nodes. When adding or removing a node we earlier faced the
issue that all the keys would have to be redistributed as the mapping became incorrect. It's important to
not
directly rely on a node.
Lets split up a node in vnodes (virtual nodes) and assign them labels, e.g. 'Node A0 to Node A5'. This
reduces the load variance among the servers used. Again we
hash
each of the vnode and place it on the edge of the abstract circle.

![Hashring with labeled nodes](/hashring-with-labeled-nodes.png)

It's good to note that when assigning labels to servers we can apply additional logic if required. If
Node A would be of instance type db.r4.8xlarge (32 vCPUs) but the other nodes are of type db.r4.4xlarge
(16 vCPUs).
We could assign twice as many labels to Node A, resulting in Node A having labels ranging from A0 to A10
distributed across the abstract circle.
This would draw twice as many requests to Node A compared to the other nodes (in theory). This is
typically referred to as assigning 'weight' to nodes.

I'm still not convinced why this pattern is more fault tolerant and resilient. We only split up nodes into
vnodes and distributed the vnodes more evenly across the abstract cirle. What is it that
makes this algorithm so interesting?

## Node B crashed

Suppose Node B crashed due to some ambiguous reason. Requests to Node
B its vnodes will now return a miss. We should first remove Node B its vnodes from the hash
ring. By doing so we can ensure that the next request is routed to the subsequent vnode on the hash
ring. Take a look at the below visualization. As vnode B1 is not available anymore we end up
routing the request to vnode C4. If this was a search query, it would return a miss the first time the key
is requested. If this was a data source (e.g. a MySQL shard) all of the search queries would continue to
miss until the shard is back up and the vnodes are again added to the abstract circle.

![Flatring crashed](/flatring-crashed.png)

There's a wide variety of solutions to ensure we do not have any query misses or downtime. This involves
another
concept called virtual replication. More on this topic will be explained in the section 'Replication
patterns'.

## Node E is added

Something similar happens if we add a node to the cluster. If we wanted to for example add Node E, we
would
have to map a new range of vnodes on the hash ring. In the below visual you can see that "Lisa" is
remapped from vnode B1 to vnode E0. This would happen for more keys as we distribute the vnodes over the
entire hash ring. We should not be dependent on any cluster change.

![Flatring added](/flatring-added.png)

The data stored under "Lisa" in vnode B1 is now not referred to anymore and becomes outdated as the next
request is routed to the new vnode E0. If this was a data source (e.g. a MySQL shard) any search query
that is now mapped to vnode E0 instead of vnode B1
will now return a miss. How do we account for this? The answer to that question will be given in the
'Replication patterns' section which can be found at the bottom of the article.

## Implementation

Now that we have a more broad understanding of hashing, load balancing and consistent hashing we can take
a look at an implementation written in Go.

To return one or more nodes that are responsible for a given positionn on the hash ring we first need to generate a so called hash key.
This hash key is used in order to determine the position on the ring.

```go
func (h *HashRing) GenKey(key string) HashKey {
	return h.hashFunc([]byte(key))
}
```

We could pass "Bruno" as a key to the `GenKey()` function and get 11 back as a hash key. 11 can
be mapped to an edge on the hash ring and we can use something like binary search in order to determine the target nodes.

```go
func (h *HashRing) GetNodePos(stringKey string) (pos int, ok bool) {
	if len(h.ring) == 0 {
		return 0, false
	}

	key := h.GenKey(stringKey)

	nodes := h.sortedKeys
	pos = sort.Search(len(nodes), func(i int) bool { return key.Less(nodes[i]) })

	if pos == len(nodes) {
		// Wrap the search, should return First node
		return 0, true
	} else {
		return pos, true
	}
}
```

`GetNodePos()` is responsible for generating the hash key and finding us the position of the
node through
binary search. Last but not least if we want to support virtual replication (which we will touch on in the
next section).
We could use something like `GetNodes()` in order to return us a slice of target nodes that we
distribute our requests over for a specific hash key.

```go
func (h *HashRing) GetNodes(stringKey string, size int) (nodes []string, ok bool) {
	pos, ok := h.GetNodePos(stringKey)
	if !ok {
		return nil, false
	}

	if size > len(h.nodes) {
		return nil, false
	}

	returnedValues := make(map[string]bool, size)
	resultSlice := make([]string, 0, size)

	for i := pos; i < pos+len(h.sortedKeys); i++ {
		key := h.sortedKeys[i%len(h.sortedKeys)]
		val := h.ring[key]
		if !returnedValues[val] {
			returnedValues[val] = true
			resultSlice = append(resultSlice, val)
		}
		if len(returnedValues) == size {
			break
		}
	}

	return resultSlice, len(resultSlice) == size
}
```

A working solution composed of the above code would look like the following.

```go
func main() {
	nodes := []string{
		"192.168.0.246:11212",
		"192.168.0.247:11212",
		"192.168.0.248:11212",
		"192.168.0.249:11212",
		"192.168.0.250:11212",
		"192.168.0.251:11212",
		"192.168.0.252:11212"}

	replicaCount := 3
	ring := hashring.New(nodes)
	targetNodes, _ := ring.GetNodes("Bruno", replicaCount)
}
```

We store the IP addresses from the nodes in a array and pass it down to the `New()`
function to generate the hash ring.
Once we have the hash ring we can pass a key and replication count to the `GetNodes()` function
in order to return us a array containing the IP addresses of the target nodes.

# Replication patterns

There's always the risk of losing a node.
We don't want any downtime when trying to access our data, but how do we elimate this
risk? We'll discuss virtual replication and balancing it.
There's multiple patterns that allow us to replicate data across other nodes in order to sustain
availability. If we can compute a target node we can also compute a list of nodes, (1
target node + N replica nodes). If a node is removed from the cluster we can fallback on a
replica node that holds the data for a specific hash key.

Replication in a nutshell is automatic and asynchronous copying of data across other
nodes. The benefit of storing the same data in several locations is that we can fallback and refer to
other nodes for our data.

## Lets keep a copy close by

This pattern is very straightforward. Remember how we traverse the hash ring in a counterclockwise
ascending direction? Well, anytime a write comes in we store the data in the the first target vnode we
meet on the hash ring.
We could simply say lets continue traversing the hash ring in the same direction until we meet the
subsequent vnode and store a copy there of what we just stored on the previous vnode.

![Keep a copy](/keep-a-copy-repl.png)

In the preceding image we can see that the first call is routed to vnode A2 and the replication call is
routed to vnode D4. Now what if Node A2 would go down? As we learned earlier, the
keys would be reassigned to the closest vnode (D4) we meet on the hash ring by rotating in a
counterclockwise ascending direction. Guess what, vnode D4 holds our data as per the replication calls we
told it to make! Without any downtime we were able to retrieve our data.

## More vnodes!

Another more fault tolerant pattern for replication is to add replication nodes to our hash ring.
As with any other node we split the replication node up in a range of vnodes and distribute the vnodes
across the hash ring.
When a write request comes in we compute a list of N distinct vnodes their position and write the data to
all
of the vnodes. For querying our data we can use the same hash function and use a 'try get' type of
implementation. If the data is
not available on the vnode or if the vnode is down we can do a 'try get' on the subsequent vnode from the
computed list of IP addresses.

![More vnodes](/replication-vnodes.png)

As you can see we have a key "Bruno" which is distributed across a few vnodes. The vnode we always try to
write to is the first vnode we meet when traversing the hash ring in a counterclockwise ascending
direction. The other vnodes we write data to are the IP addresses coming from the specific hash function
we
used in order to get list of IP addresses including our replication nodes. We
are not simply
using h(x) % m anymore as we need a hash function that returns us more distinct points of the hash ring.
(if you want to look at an
example look at the above Go implementation).

# Conclusion

Consistent hashing is a good algorithm when you plan on distributing your data across a large sized
cluster.
To implement it you need a deep understanding of both the algorithm and various replication
patterns in order to balance the load appropriately. Implementing consistent hashing in a sample project
is a great way of learning about distributed systems, I highly recommend doing so.
I was really impressed by how this algorithm is actually very simple but yet still hard to implement
properly as there's so many edge cases to cover. Bottom line, consistent hashing rocks and even though you
might never get to work with it it's still a great algorithm to study and bring up during a talk.

# Footnotes

- I often use the word 'node'. This refers to a 'server' or 'host' but can also be interpret as a 'shard' when we speak about horizontally partitioning and assigning a partition to a 'node'.

- As an extra remark, vnodes are 'virtual nodes'. Which is a node but split up in a range of labels. These labels are distributed across the hash ring.

- Sometimes I call the hash ring an abstract circle and vice versa, they both refer to the circle on the visuals.

- The ['Consistent Hashing and Random Trees'](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf) paper from David Karger played a very important role in my reserach, I recommend reading this.

- The ['Dynamo: Amazon’s Highly Available Key-value Store'](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) explains how DynamoDb is implemented. I really enjoyed the read and learned a lot.

- I've used pieces of code from the [hashring](https://github.com/serialx/hashring) project throughout this article.
