# Riak developer training at RICON, October 28, 2013

## Intro

Nathan Aschbacher is speaking -- he's the de facto head of training at Riak.

Riak mostly wants to be used as a key-value store.  You can use it in other ways, but it's really not too happy.

Mostly it wants to be used for <= 4MB values.

Riak is "masterless".  There's no single point of failure, so it's more fault-tolerant than other databases; also, data is by default replicated 3 times.  (If you want more than 4 replicas, you have to tweak something at compile time.)

If you have a 5-node cluster, having 2 nodes go offline is no big deal.

It's "highly available"!  ("This is a crappy term, like 'big data'.")

But perhaps most importantly, Riak scales, especially when used as a key-value store.  It's easy to just add more machines and make it go faster: that gives you both more storage capacity and lower latency.  (OTOH, that might not be your bottleneck; the network might be the bottleneck.  But that's not Riak's problem!)

"Nobody gets fired for picking EC2." -- NA

You're going to work with Riak via some sort of "client library" API.  They have five that they maintain.  These five official libraries are for Erlang, Java (and hence Scala), Ruby, Python, PHP.

Ruby client -- concurrent requests are serially blocking!  They get serialized!

Python client -- concurrent requests are OK if you use protobufs, but if you use HTTP, you get the GIL.

PHP -- concurrent requests are serially blocking!  "This is 500 degrees of bad."

There are also a ton of community-supported libraries for Go, Node.js, Haskell, C, C++, Clojure, .Net, Scala, etc.

## Riak object

A Riak object is the thing that gets replicated around the cluster.

Under the hood it's an Erlang record structure.

It's composed of a key, a value, and some metadata (last modified, a "vclock" (I guess this is what the cool kids say instead of "vector clock"), etc.).

You can also define your own metadata.

## Buckets, nodes, clusters

A node is a machine; a cluster is a collection of machines.

We store Riak objects in "buckets".  A bucket is just a logical namespace.  It doesn't imply anything about physical location.  Not all the things in a bucket live on one machine.

You don't *have* to use buckets, but they're useful to hang settings off of.

## How to interact with your data in Riak

  * Through the key-value store! It's a giant distributed hashmap.

  * Secondary indexes -- a way to look up something other than by its primary key.  This gets tagged in the metadata of the object.  As you grow the size of your cluster, though, secondary indexes get slower.  The reason for that is how the index is partitioned.  This actually came about as a way to help people make Riak interface with MapReduce!  It's a hack to get a key-value store to behave a little bit more like a traditional relational database.
	
  * Search!  It's a full-text query engine!  It looks like Solr/Lucene, but not actually Solr/Lucene under the hood.  "This is the one place where you can enforce a schema on Riak and Riak understands that there's a schema there."  Search scales linearly -- it's another application running alongside "Riak KV", the key-value store, in the Riak VM.  (Also, there's a new version coming in Riak 2.0 that is much faster and *is* actually Solr/Lucene; however, it partitions the index in the same way that secondary indexes do.  So it does get slower as the cluster grows, but nevertheless, Solr/Lucene is so much faster that you're still better off up to around 15 nodes.)
  
Of the three, key-value is the best!
  
There are two others -- link-walking and map-reduce ("but it's not Hadoop map-reduce") -- "but I'm not going to tell you about them".

## Some basic principles of distributed data architecture

"The goal was to make this material accessible to a ten-year-old.  I didn't succeed, but it worked on an eleven-year-old."

First of all, how much data is too much for Riak to handle?  Data set size is actually not a big deal.  They once tried to work with two petabytes, and the problem was disk failure, not whether Riak could run.

### Sharding

Suppose that we don't have enough room for all our data on one machine.  How do we divide up data?  A traditional approach is to shard by Type of Thing.  But we generally end up with unbalanced distribution of data, and we have to rebalance, which ends up being some kind of service outage.  And then eventually we'll have to do that again.  And again.

In Riak, we shard by what things' buckets and keys hash to.  We take our object's bucket and key, and we run that through a hash function (in Riak it's SHA1) and get an integer, which is then mapped to some node.  Consistent hashing means that every time we hash a particular key we'll get a particular integer.

### Rings

All these machines ("partitions") that we hash things into form what's known as a "ring".  When you set up your database in the first place, you decide on a ring size and a replication level.  A particular piece of data is replicated onto the place it hashes onto on the ring, and also onto however many other replicas that we've decided we want (usually two others) that directly follow that one on the ring.  (This is like what's described in the Dynamo paper.)

Every node has the same map, so when a piece of data arrives and needs to get stored somewhere, every node either knows how to store the data or how to direct it elsewhere.

One machine (a "node") is responsible for some number of "vnodes".  There's a one-to-one mapping between partitions and vnodes.

When you add a new node to the cluster, everybody can send some of their vnodes to the new node.  This is called "ownership handoff".  However, that's not the only kind of handoff; there's also the possiblility of node failure and network failure.  This is why Riak scales well horizontally -- we don't have to increase the number of partitions when we add new nodes.

Suppose that some data arrives and needs to get stored, and the "home" node that it's supposed to go to (I'm making up terminology here -- I mean the node it hashes to) is unavailable.  When that happens, someone else will create an extra "failover" vnode that it knows is transient.  When the missing node comes back online, it'll steal that vnode back.  (And replication happens even if the home node isn't there -- it'll get replicated to the next N things on the ring.)

To be specific, each node will try to send newly arriving data to the node it belongs to, and if that doesn't work, it'll go to the next node on the ring that's available.  This way, we have a deterministic way to know where in the ring to look for a given piece of data.

64 is the default ring size to start with (and the default is five nodes, which is also the minimum supported configuration), but that doesn't work well for production -- we recommend a ring size of 256 to start with.  "We try to have between 20 and 50 partitions per node."
