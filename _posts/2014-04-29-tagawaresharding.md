Title: How to Balance Collections Across Your Sharded Cluster
Date: 2014-04-29 02:04 
Type: post
Published: true
Slug: :TaggedCollectionBalancing
Tags: sharding,tags,balancing,mongodb

### Question:

Is it possible to use ["Tag aware sharding"] [1]  feature without having to use a special shard key?  The example in the tutorial makes it look like we would have to change our shard key to have a prefix value that we can define tag ranges on but we're already sharded.  We have many collections in this database and we want to limit each collection to a subset of the shards so we can isolate the busy ones from each other.

### Answer:

Yes, that is absolutely possible, and it's one of the cool capabilities of tag aware sharding.   A quick review of the feature.

##### Tag aware sharding feature
Tags associate specific ranges of shard key values with specific shards for use in managing deployment patterns.
What this means is that in your sharded cluster you can assign zero, one or more tags (or labels) to each shard.  Then you can assign ranges of shard key values in various sharded collections to these tags.  The balancer then moves the appropriate chunks to appropriate shards to keep things the way you "assigned" them. 
#####The Balancer: Diversion into migration details
The whole balancing and migrations process is worthy of its own separate write-up but for now, I will simplify most of it and point out at the high level that the balancer is a thread that runs on mongos that wakes up periodically and checks (1) if it should be running (2) if there is anything for it to do.†  For the balancer "something to do" is always about moving chunks between shards.  The priorities that it considers when deciding which chunks need to be moved are as follows:

- draining shards: if one of the shards is "draining" - i.e. you plan to decommission it - then this will always be the first priority for all migrations unless it has no data left.

- tagged shards: if any chunks are on the "wrong" tagged shard for  its range, then it has to be moved to a "correct" shard.

- balance the remaining chunks: if the shard with the most chunks has nine+ more chunks than the shard with the fewest chunks, then the balancer will move chunks to try to keep things in balance.

##### How do you tag shards and ranges?
All you have to do for tagging to work is mark some shards with "tags" and specify which ranges of shard key values will be associated with that tag.   The relevant commands are `sh.addTag("shardName","tagName")` and `sh.addTagRange("namespace", { shardKey: minValue }, { shardKey: maxValue }, "tagName")`.

The MongoDB docs have a great tutorial that you always see used as an example for tag aware sharding - your shard key has to include a prefix field that can be used to figure out which geographical region the user is in, and the range of shard key values that starts with certain regions will be associated with shards in that data center.

That's all fine and good, but I'll show you that it doesn't have to be nearly that complex.
#####How you can use tags to designate which shards a sharded collection can use.
Let's walk through an example.   I have three shards in my test cluster:

<pre class="prettyprint lang-js">
tagdb@mongos(2.6.0) > db.getSiblingDB("config").shards.find()
{ "_id" : "shard0000", "host" : "localhost:30000" }
{ "_id" : "shard0001", "host" : "localhost:30001" }
{ "_id" : "shard0002", "host" : "localhost:30002" }
</pre>

I will add two tags, each to two shards.  Let's say that shards 0000 and 0001 have a lot of RAM, and shards 0001 and 0002 have very fast flash storage and I plan to distribute my data to take advantage of the different physical resources:

<pre class="prettyprint lang-js">
tagdb@mongos(2.6.0) > sh.addShardTag("shard0000","HI_MEM")
tagdb@mongos(2.6.0) > sh.addShardTag("shard0002","FLASH")
tagdb@mongos(2.6.0) > sh.addShardTag("shard0001","FLASH")
tagdb@mongos(2.6.0) > sh.addShardTag("shard0001","HI_MEM")
</pre>

Now that I tagged my shards, I will add tag ranges to two different collections.  Note, I don't have these collections yet, and I haven't even sharded them yet, but I want to have the tags ready for them when they get created:

<pre class="prettyprint lang-js">
tagdb@mongos(2.6.0) > sh.addTagRange("tagdb.bigidx", {_id:MinKey},{_id:MaxKey},"HI_MEM");
tagdb@mongos(2.6.0) > sh.addTagRange("tagdb.bigdata", {_id:MinKey},{_id:MaxKey},"FLASH");
</pre>

I have a collection with big indexes (called bigidx) that I want to constrain only to shards tagged "HI_MEM" and I have another collection with a lot of data (called bigdata) that I want to keep on shards that have flash storage because I know the data will be read from disk a lot.  Note that I only needed to know what I will be using as my shard key, and I specified MinKey to MaxKey as my range - that means *all* of the chunks!

I will now shard the collections and take a look at how things are shaping up:

<pre class="prettyprint lang-js">
tagdb@mongos(2.6.0) > sh.enableSharding("tagdb")
{ "ok" : 1 }
tagdb@mongos(2.6.0) > sh.shardCollection("tagdb.bigdata", {_id:"hashed"})
{ "collectionsharded" : "tagdb.bigdata", "ok" : 1 }
tagdb@mongos(2.6.0) > sh.shardCollection("tagdb.bigidx", {_id:"hashed"})
{ "collectionsharded" : "tagdb.bigidx", "ok" : 1 }
tagdb@mongos(2.6.0) > sh.status()
--- Sharding Status ---
  sharding version: {
	"_id" : 1,
	"version" : 4,
	"minCompatibleVersion" : 4,
	"currentVersion" : 5,
	"clusterId" : ObjectId("535be5d7d5274545e9d01426")
  }
  shards:
	{  "_id" : "shard0000",  "host" : "localhost:30000",  "tags" : [ "HI_MEM" ] }
	{  "_id" : "shard0001",  "host" : "localhost:30001",  "tags" : [ "FLASH", "HI_MEM" ] }
	{  "_id" : "shard0002",  "host" : "localhost:30002",  "tags" : [ "FLASH" ] }
  databases:
	{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
	{  "_id" : "tagdb",  "partitioned" : true,  "primary" : "shard0001" }
		tagdb.bigdata
			shard key: { "_id" : "hashed" }
			chunks:
				shard0001	3
				shard0002	3
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : -6148914691236517204 } on : shard0001
			{ "_id" : -6148914691236517204 } -->> { "_id" : -3074457345618258602 } on : shard0002
			{ "_id" : -3074457345618258602 } -->> { "_id" : 0 } on : shard0001
			{ "_id" : 0 } -->> { "_id" : 3074457345618258602 } on : shard0001
			{ "_id" : 3074457345618258602 } -->> { "_id" : 6148914691236517204 } on : shard0002
			{ "_id" : 6148914691236517204 } -->> { "_id" : { "$maxKey" : 1 } } on : shard0002
			 tag: FLASH  { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } }
		tagdb.bigidx
			shard key: { "_id" : "hashed" }
			chunks:
				shard0000	3
				shard0001	3
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : -6148914691236517204 } on : shard0000
			{ "_id" : -6148914691236517204 } -->> { "_id" : -3074457345618258602 } on : shard0000
			{ "_id" : -3074457345618258602 } -->> { "_id" : 0 } on : shard0001
			{ "_id" : 0 } -->> { "_id" : 3074457345618258602 } on : shard0001
			{ "_id" : 3074457345618258602 } -->> { "_id" : 6148914691236517204 } on : shard0000
			{ "_id" : 6148914691236517204 } -->> { "_id" : { "$maxKey" : 1 } } on : shard0001
			 tag: HI_MEM  { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } }
</pre>
##### How you can use tags to make collection migrate from one shard to another
What if you have a number of unsharded collections in your sharded database and you don't want for all of them to hang out on the primary shard for this DB?   Well, you might need unique tags for each shard, but then you can do this to move collection one to `shard0001`:
 
<pre class="prettyprint lang-js">
tagdb@mongos(2.6.0) > sh.addShardTag("shard0002","shard2")
tagdb@mongos(2.6.0) > sh.addTagRange("tagdb.one", {_id:MinKey},{_id:MaxKey},"shard2")
tagdb@mongos(2.6.0) > sh.shardCollection("tagdb.one",{_id:1})
{ "collectionsharded" : "tagdb.one", "ok" : 1 }
tagdb@mongos(2.6.0) > sh.status()
   ...
 		tagdb.one
			shard key: { "_id" : 1 }
			chunks:
				shard0002	1
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0002
			 tag: shard2  { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } }
</pre>

If we peek inside the config database, we should see our tags in the `config.tags` collection, our shard ranges attached to chunks in `config.chunks` and we can find evidence of the chunk moves due to tag policy in the `config.changelog` collection, as well as the `mongos` and `mongod` log files.

To summarize: tag aware sharding can be easily used to distribute a single collection a particular way across all shards,  to isolate whole collections on a subset of shards, and even to move an entire collection from one shard to another.

---

† This is definitely a gross simplification of all the steps the balancer goes through - look for a more detailed write-up demystifying the inner workings of migrations some time soon.

[1]: http://docs.mongodb.org/manual/core/tag-aware-sharding/