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

For example, ...

#3
Clients should talk to Riak via an haproxy on localhost which is aware of all nodes in the ring. Don't attempt to build a client that is aware of all nodes and intelligently routes requests. Building such a client is complex and degenerates to building all of the failure detection intrinsic to haproxy.
