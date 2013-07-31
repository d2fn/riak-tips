_in progress_

# Riak Tips

A collection of Riak tips follows. This is based on my own experience using 1.2.1 with leveldb and bitcask backends. Yours, obviously, may differ. Apologies in advance for the subjective nature of this content. Every workload is unique and the combinatorial explosion of variables needed to control a comprehensive set of benchmarks mean you are better off building your own.

At sufficiently low scale, almost any database or usage pattern will suffice. This isn't a credit to the database, but rather a sign that you aren't solving a particularly difficult problem as far as scale is concerned. These tips are made for those who are truly building and operating "large scale" systems and considering Riak.

Pull requests and generally constructive disparaging comments are welcome.

#0
Riak is not a place to store relational data. Don't use Riak with the expectation of replacing a MySQL database that makes heavy use of joins, enforces referential integrity, and forces data into a type system. Riak is not a search engine so don't expect to use it for free text indexing with long-term success. Don't employ Riak with the expectation of "querying" or "scanning" a range of data. While it affords some secondary index support (2i) by way of the leveldb backend, relying on these features in a heavily loaded production cluster has only brought myself and my colleagues disaster. Projects like [Yokozuna](https://github.com/basho/yokozuna) for text search are in progress and may turn out to be awesome. I just don't have experience with them.

That being said, Riak is the best key-value store I have ever used. It is wonderful for storing huge amounts of data so long as you can remember or compute the keys you want to read once they have been written. Once you learn to tilt your head the right way and squawk the right squawks, it is a pleasure both to develop against and use in production. It is possible to perform cluster upgrades and lose nodes (hard stop) with little to no application downtime.

#1
Use SSDs. RAID several together per machine in a 0 or 10 configuration if you can. Then proceed to #2.

#2
A riak cluster tends to reach capacity long before its nodes exhaust CPU or IO. Rather, coordination overhead between nodes is typically a pinch point. Bear in mind that, in almost every case, reading or writing a key requires some coordination among replicas. At a certain workload the amount of time spent awaiting responses from other nodes causes mean, and especially 99th percentile, latencies to rise unacceptably. In not so extreme cases, the default gen_fsm timeout of 60s is even exhausted meaning the cluster is basically offline. No reads. No writes. No handoff. 

But hope is not lost. There is usually a way to adjust your workload in such a way that lets Riak breathe much better and make use of the powerful hardware you've given it. The problem above arises from exhausting throughput of the cluster in terms of keys read/written. Simply reduce the number of keys by storing similar data together in blocks. Riak works wonderfully as a block store in this way.

For example, if you are storing time-series data at a high resolution, try storing all information for, say, a 10 minute window under the same key. The math for aligning timestamps to these boundaries is simple enough:

```
irb(main):001:0> `date +%s`.strip + " -> " + (`date +%s`.to_i / (10*60) * (10*60)).to_s
=> "1373491342 -> 1373491200"
```

But now you have a coordination problem. If writes for the same key originate from multiple nodes, you risk data loss. Ensuring a single writer per key is one way to overcome that. There are more sophisticated ways to handle this but there is no one correct answer. Future CRDT work in Riak may allow this coordination to happen within Riak, but for now it must happen elsewhere.

#3
Clients should talk to Riak via an haproxy on localhost which is aware of all nodes in the ring. Don't attempt to build a client that is aware of all nodes and intelligently routes requests. Building such a client is complex and degenerates to building all of the failure detection intrinsic to haproxy.

#3.1
Simulate both failure and slowdown of a Riak node and measure its impact on your client. It is important to understand how a misbehaving cluster ripples through your infrastructure--and it will.

#4
Don't expose Riak to the internet. Not only does Riak not include native support for fine-grained access controls, it is straightforward to take down a cluster with an expensive map-reduce job. And because map-reduce jobs accept javascript, all of the issues of [mobile code](http://en.wikipedia.org/wiki/Mobile_code) in a distributed system applies--even if access to the cluster is well-controlled.

Read [this post](http://aphyr.com/posts/224-do-not-expose-riak-to-the-internet) from @aphyr and @jrecursive for more details.

Solve security concerns outside of Riak, but make sure you store sufficient information in keys and values to do so.

#5 todo rolling upgrades

#6
Use the bitcask backend unless you absolutely can't do without indexed scans for keys falling in a range. If this is something you think you need, it's worth consdering redesigning your application so that you can compute keys directly instead. I mentioned in #2 that a cluster will usually reach capacity long before hardware resources are exhausted. Bitcask can mitigate this because it allows much greater disk throughput. There are, I believe, some other subtle interactions between the eleveldb NIF and the erlang scheduler that can briefly cause nodes to stop doing meaningful work. This may be part of the reason leveldb-backed clusters tend will tend to fair worse with equivalent workloads than a bitcask backed cluster.


