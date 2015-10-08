Title: Best Versions with MongoDB
Date: 2014-05-30
Type: post
Published: true
Slug: :BestVersion
Tags: schema,modeling,indexes,versioning

### Question: ###
Recall [our previous discussion](http://askasya.com/post/trackversions) about ways to  recreate older version of a document that ever existed in a particular collection.   
    
The goal was to preserve every state for each object, but to respond to queries with the "current" or "latest" version.   We had a requirement to be able to have an infrequent audit to review all or some previous versions of the document.

### Answer: ###
I had suggested at the time that there was a different way to achieve this that I liked better than the discussed methods and I'm going to describe it now.

#### Previous Discussion Summary: ####
Up to this point, we considered keeping versions of the same document within one MongoDB document, in separate documents within the same collection, or by "archiving off" older versions of the document into a separate collection.

We looked at the trade-offs and decided that the important factors were our ability to 

- return or match only the current document(s)
- generate new version number to "update" existing and add new attributes
    + including recovering from failure in the middle of a set of operations (if there is more than one)

##### Where we left off:
Here's a table that shows for each schema choice that we considered how well we can handle the reads, writes and if an update has to make more than one write, how easy it is to recover or to be in a relatively "safe" state:

|Schema | Fetch 1 | Fetch Many || Update | Recover if fail|
------------ | ------------- | ------------|-|---------------|-----------------|-------------
|1) New doc for each | Easy,Fast  | Not easy,Slow | | Medium | N/A |
|1a) New doc with "current" | Easy,Fast  | Easy,Fast | | Medium | Hard |
|2) Embedded in single doc | Easy,Fastest  | Easy,Fastest | | Medium | N/A |
|3) Sep Collection for prev. |  Easy,Fastest  | Easy,Fastest  | | Medium |  Medium Hard |
|4) Deltas only in new doc | Hard,Slow | Hard,Slow | | Medium | N/A |
|?) TBD |  Easy,Fastest  | Easy,Fastest  | | Easy,Fastest |  N/A |

"N/A" for recovery means there is no inconsistent state possible - if we only have to make one write to create/add a new version, we are safe from any inconsistency.  So "N/A" is the "easiest" value there.  

What we want is something that makes all our tasks easy, and does not have any performance issues nor consistency problems.   For creating this solution, we will pick and choose the best parts of the previously considered schema.

No doubt you noticed that fetching one or many is fastest and simplest when we keep the old versioned documents out of our "current" collection.  This makes our queries whether for one or all latest versions fast and they can use indexes whether you're querying, updating or aggregating.

How do we get fast updates that keep the current document current but save the previous version somewhere else?  We know that we don't have multi-statement transaction in MongoDB so we can't ensure that a regular  update of one document and an insert of another document are atomic.  However, there *is* something that's always updated atomically along with *every* write that happens in your collection, and that is the "Oplog".

#### The Oplog ####

The oplog (full name: 'oplog.rs' collection in 'local' database) is a special collection that's used by the replication mechanism.  Every single write operation is persisted into the oplog atomically with being applied to the data files, indexes and the journal.  You can read more about the oplog in [the docs](http://docs.mongodb.org/manual/core/replica-set-oplog/), but what I'm going to show you is what it looks like in the oplog when an insert or update happens, and how we can use that for our own purposes.

If I perform this insert into my collection:

    > db.docs.insert(
               {"_id":ObjectId("5387edd9ba5871da01786f85"), 
                "docId":174, "version":1, "attr1":165});
    WriteResult({ "nInserted" : 1 })
    
what I will see in the oplog will look like this:

    > db.getSiblingDB("local").oplog.rs.find().sort({"$natural":-1}).limit(-1).pretty();
    {
             "ts" : Timestamp(1401417307, 1),
              "h" : NumberLong("-1030581192915920539"), 
              "v" : 2, 
              "op" : "i", 
              "ns" : "blog.docs", 
              "o" : { 
                            "_id" : ObjectId("5387edd9ba5871da01786f85"), 
                            "docId" : 174, 
                            "version" : 1, 
                            "attr1" : 165 
              } 
    }

If I perform this update:

    > db.docs.update( 
                   { "docId" : 174 }, 
                   { "$inc":{"version":1}, "$set":{ "attr2": "A-1" }  } 
        );
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
     
what I get in the oplog is this:

    {
	        "ts" : Timestamp(1401417535, 1),
	        "h" : NumberLong("2381950322402503088"),
	         "v" : 2,
	         "op" : "u",
	         "ns" : "blog.docs",
	         "o2" : {
	          	     "_id" : ObjectId("5387edd9ba5871da01786f85")
	         },
	         "o" : {
		            "$set" : {
			                "version" : 2,
			                "attr2" : "A-1"
		            }
	         }
    }
    
It turns out that with versioned documents, I wouldn't actually ever do an insert, but rather I would just always do an update, with an upsert option, that way I don't need to test if a document with this `docId` already exists.
    
    > db.docs.update( 
         { "docId" : 175 }, 
         { "$inc":{"version":1}, "$set":{ "attr1": 999 }  }, 
         { "upsert" : true } 
    );
    WriteResult({
    	"nMatched" : 0,
    	"nUpserted" : 1,
    	"nModified" : 0,
    	"_id" : ObjectId("5387eff7a08472e30040b4bc")
    })
    
Let's see what the oplog entry for this upsert looks like:

    { 
    	"ts" : Timestamp(1401417719, 1), 
    	"h" : NumberLong("2031090002854356513"), 
    	"v" : 2, 
    	"op" : "i", 
    	"ns" : "blog.docs", 
    	"o" : {
    		 "_id" : ObjectId("5387eff7a08472e30040b4bc"), 
    		 "docId" : 175, 
    		 "version" : 1, 
    		 "attr1" : 999 
    	} 
    }
    
Looks like the oplog entry reflects the actual operation that was performed, **not** the operation that I specified.  I asked for an update - when it's an update, the oplog will show it as an update, when it's turned into an upsert, the oplog will show it as an insert.  When it was an update, I had asked it to "increment" but what it put in the oplog was what the actual value saved was.[^1]

I'm sure most of you see where I'm going with this.  Rather than fumbling with creating and updating documents in the "previous versions" collection when we perform an update to a document, we can do it asynchronously, the way MongoDB secondaries do it.

You may think it's not easy, but it turns out that there are lots of [helpers](http://docs.mongodb.org/manual/reference/method/cursor.addOption/) for dealing with [capped collections](http://docs.mongodb.org/manual/core/capped-collections/) (which is what the oplog is).  One of the most useful things you can do is "tail the oplog".  This is the same mechanism that secondaries use to find out when new writes happen on the primary: they tail the oplog the same way you can do `tail -f logfile.txt` command - this will show you the last part of the file, but rather than giving you back the prompt when it's done, it will just sit there and wait.  When more things are written to the file, `tail -f` will echo them to the screen.   This is exactly how it works with [tailable cursors](http://docs.mongodb.org/manual/reference/method/cursor.addOption/#example) on capped collections.  If you specify the [right special options](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/?pageVersion=106#op-query), you can get data back, but when there is no more data, instead of timing out and having to re-query, you will just sit there and wait till more data shows up.

Here is a little demo.  The code and explanations are after the video, so feel free to browse ahead before watching, or you can watch first and read the explanations after.

##### Tailing the oplog to maintain a copy of a collection elsewhere
For our first example, we'll do something simple - we will watch the oplog for changes to a specific collection, and then we will apply those changes to our own copy of the collection - we will call our collection something else.  Our example stores the copy in the same database, but of course, it could be anywhere else, including in a completely different replica set or standalone server.

<pre>
<script src="https://google-code-prettify.googlecode.com/svn/loader/run_prettify.js"></script>
<iframe width="560" height="315" src="//www.youtube.com/embed/U-MVlb0cRHU" frameborder="0" allowfullscreen></iframe>
</pre>

Code for set-up of variables with comments:

<pre class="prettyprint lang-js">
/* shell likes to ”page” things for us every 20 documents,  *
 * so we will increase that value for this demo */
DBQuery.shellBatchSize=200;
var namespace="blog.docs"; /* the collection to watch */ 
var coll="docs_archive";  /* copy collection name */

db.getCollection(coll).drop();  /* only the very first time :) */
db.lastApplied.drop();
/* find the last timestamp we applied: this is where we restart */
var prevTS=(db.getCollection("lastApplied").count({"_id":coll})==0) ?
     db.getSiblingDB("local").oplog.rs.findOne({"ns":namespace}).ts :
     db.getCollection("lastApplied").findOne({"_id":coll}).ts;
/* initialize or update lastApplied for this collection */
db.getCollection("lastApplied").update(
                { "_id" : coll }, 
                { "$set : { "ts" : prevTS } },
                { "upsert" : true }
);
</pre>

Code for setting up the cursor using [appropriate options](http://docs.mongodb.org/manual/reference/method/cursor.addOption/#flags) allows us to find the right spot in the oplog quickly, and makes our cursor tail the data, asking server to forgo the usual cursor timeout based on inactivity:

<pre class="prettyprint lang-js">
/* set up the cursor with appropriate filter and options */
var cursor=db.getSiblingDB("local").getCollection("oplog.rs"
       ).find({"ns":namespace,"ts":{"$gte":prevTS}}).addOption(DBQuery.Option.oplogReplay
       ).addOption(DBQuery.Option.awaitData).addOption(DBQuery.Option.tailable
       ).addOption(DBQuery.Option.noTimeout);
</pre>

Code running on the right-hand-side (blue screen) which loops and inserts or updates the watched collection every second:

<pre class="prettyprint lang-js">
for ( docId = 270; docId < 290; docId++ ) {
        print("waiting one second...");
        sleep(1000);
        printjson(db.docs.update(
                      { "_id": docId },
                      { "$inc" : {"version":1}, "$set":{"attr7":"xxx"+docId } },
                      { "upsert" : true }));
}
</pre>

Code that fetches documents from the tailable cursor and applies appropriate changes to our "copy" collection:

<pre class="prettyprint lang-js">
while (cursor.hasNext()) {
       
       var doc=cursor.next();
       var operation = (doc.op=="u") ? "update" : "insert";
       print("TS: " + doc.ts + " for " + operation + " at " + new Date());
       
       if ( doc.op == "i") {  /* originally an upsert */
           result = db.getCollection(coll).save(doc.o);
           if (result.nUpserted==1) print("Inserted doc _id " + doc.o._id);
           else {
              if (result.nMatched==1 ) {
                 if ( result.nModified==0) print("Doc " + doc.o._id + " exists.");
                 else  print("Doc " + doc.o._id + " may have been newer");
              } else throw "Insert error " + tojson(doc)  + " " + tojson(result);
           }
       } else if ( doc.op == "u" ) { /* originally an update */
           result = db.getCollection(coll).update(doc.o2, doc.o);
           if (result.nModified ==1) print("Updated doc _id " + doc.o2._id);
           else if (result.nMatched==1 && result.nModified==0) print(
                         "Already updated doc _id " + doc.o2._id);
           else  throw "No update for " + tojson(doc) + " " + tojson(result);
       } else if (doc.op != "c") throw "Unexpected op! " + tojson(doc);
       
       res=db.getCollection("lastApplied").update(
                     { "_id" : coll },
                     { "$set" : { "ts" : doc.ts } }
       );

       if (res.nModified==1) print("Updated lastApplied to " + doc.ts);
       else print("Repeated last applied op");

       prevTS=doc.ts; /* save in case we need to refetch from the oplog */
}
</pre>
Of course this code does minimal error checking and it's not set up to automatically restart if it loses connection to the primary, or the primary changes in the replica set.  This is because here we are reading from a local oplog when in real life you may be fetching data from another server or cluster entirely.  Even so, about 15 lines of code there are for error checking and information printing, so the actual "work" we do is quite simple.  

##### Creating a full archive from tailing the oplog
Now that we know how to replay original documents to maintain an indentical "copy" collection, let's see what we have to do differently when we want to insert a new version of the document without losing any of the old versions.

For simplicity, I put the docId in the example collection into the `_id` field, so I will need to structure the full archive collection schema differently, since it cannot have multiple documents with the same `_id`.[^2]  For simplicity, I will let MongoDB generate the `_id` and I will use the combination of docId and version with a unique index on them to prevent duplicate versions.  I could achieve something similar by using the combination of original `_id` (which is the docId) and `version` fields as a compound `_id` field but then I would need to do more complicated transformations on the oplog entry.  I always choose the method that is  simpler.

[^2]: Note that if I used a separate `docId` field, I would have to do extra work, since replication identifies the document by `_id` the `docId` field would not be in the oplog, so I would need to store the original collection's `_id` fields in my documents, otherwise I wouldn't know which docId was being updated.

Now when we get an insert operation in the oplog, we should be able to insert the document the same way we were doing it before, except we want to move `_id` value into `docId` field.  If the save fails to insert a new document because of a duplicate constraint violation, then we already have that docId and version - we would expect that when we are replaying the same entry in the oplog more than once.   

If we get an update, it can be one of two kinds - it can be one that sets or unsets specific fields, or it can be the kind that overwrites the entire document with a new document (with the same `_id` of course).  The latter case can be handled by the same code we have for the insert, with an appropriate transformation of the document.   If it's the `$set` and `$unset` kind, then we have to fetch the previous version of this document and apply the changes to it before inserting it as a document representing a new version.[^3]

[^1]: Because the oplog must write change operations in ["idemponent"](http://docs.mongodb.org/manual/reference/glossary/#term-idempotent) form, all update operators are transformed into their equivalent `$set` operations.

[^3]: See footnote **1.**

Here is our code, with comments:

The setup:
<pre class="prettyprint lang-js">
var coll="docs_full_archive";  // different collection
db.getCollection(coll).drop();     // first time only!
db.getCollection(coll).ensureIndex( 
         { "docId":1, "version": 1 },
         { "unique" : true, "background" : true } );
if (db.getCollection("lastApplied").count({"_id":coll})==0) {
     prevTS=db.getSiblingDB("local").oplog.rs.findOne({"ns":namespace}).ts;
     db.getCollection("lastApplied").update( { "_id" : coll }, 
        {"$set" : { "ts" : prevTS } }, { "upsert" : true } );
} else {
     prevTS=db.getCollection("lastApplied").findOne({"_id":coll}).ts;
}
</pre>
The cursor (same as before):
<pre class="prettyprint lang-js">
var cursor=db.getSiblingDB("local").getCollection("oplog.rs"
       ).find({"ns":namespace,"ts":{"$gte":prevTS}}).addOption(DBQuery.Option.oplogReplay
       ).addOption(DBQuery.Option.awaitData).addOption(DBQuery.Option.tailable
       ).addOption(DBQuery.Option.noTimeout);
</pre>
The worker loop (slightly adjusted):
<pre class="prettyprint lang-java">
while (cursor.hasNext()) {
       var doc=cursor.next();
       var operation = (doc.op=="u") ? "update" : "insert";
       var id = (doc.op=="u") ? doc.o2._id : doc.o._id;
       var newDoc={ };
       print("TS: " + doc.ts + " for " + operation + " at " + new Date());
       if ( doc.op == "i" || 
               (doc.op == "u" && doc.o.hasOwnProperty("_id")) ) {
           for (i in doc.o) {
                   if (i=='_id') newDoc.docId=doc["o"][i];
                   else newDoc[i]=doc["o"][i];
           }
       } else if ( doc.op == "u" ) {
           /* create new doc out of old document and the sets and unsets */
           var prevVersion = { "docId" : doc.o2._id, 
                        "version" : doc.o["$set"]["version"]-1 };
           var prevDoc = db.getCollection(coll).findOne("prevVersion", {"_id":0});
           if (prevDoc == null) {
                        throw "Couldn't find previous version in archive! " + 
                                     tojson(prevVersion) + tojson(doc);
           }
           newDoc = prevDoc;
           if (doc.o.hasOwnProperty("$set")) {
               for (i in doc.o["$set"]) {
                   newDoc[i]=doc.o["$set"][i];
               }
           } else if (doc.o.hasOwnProperty("$unset")) { 
               for (i in doc.o["$unset"]) {
                   delete(newDoc[i]);
               }
           } else throw "Can only handle update with '_id', '$set' or '$unset' ";
       } else if (doc.op != "c") throw "Unexpected op! " + tojson(doc);

       var  result = db.getCollection(coll).insert(newDoc);
       if (result.nInserted==1) {
             print("Inserted doc " + 
                      newDoc.docId + " version " + newDoc.version);
       } else {
             if (result.getWriteError().code==11000 ) {
                    print("Doc " + newDoc.docId + " version " + 
                           newDoc.version + " already exists.");
             } else throw "Error inserting " + tojson(doc)  + 
                           " as " + tojson(newDoc)+ "Result " + tojson(result);
       }

       var res=db.getCollection("lastApplied").update(
                   { "_id" : coll },
                   { "$set" : {ts:doc.ts} },
                   { "upsert" : true }
       );
       var prevTS=doc.ts;
       print("Set lastApplied to " + doc.ts);
}
</pre>

It turns out that the loop will be slightly simpler because no matter what comes in, we will always do an insert into the full archive collection.

##### Test it out!
Let's run this code and then compare for a single docId the operations in the oplog, and what we end up with in the archive collection:

The oplog entries:
<pre class="prettyprint lang-java">
db.getSiblingDB("local").getCollection("oplog.rs").find( {
            "ns" : namespace,
            "$or" : [ { "o._id" : 279 }, { "o2._id" : 279 } ] 
         },
         { "o" : 1 } );
{ "o" : { "_id" : 279, "version" : 1, "attr7" : "xxx279" } }
{ "o" : { "$set" : { "version" : 2 } } }
{ "o" : { "$set" : { "version" : 3, "attrCounter" : 1, "attr9" : 1, "attrArray" : [ "xxx" ] } } }
{ "o" : { "_id" : 279, "version" : 4, "attr7" : "xxx279", "attrCounter" : 1, "attr9" : 1, "attrArray" : [ "xxx" ], "attrNew" : "abc" } }
{ "o" : { "_id" : 279, "version" : 5, "attr7" : "xxx279", "attrCounter" : 2, "attr9" : 1, "attrArray" : [ "xxx" ], "attrNewReplacement" : "abc" } }
{ "o" : { "$set" : { "version" : 6, "attrCounter" : 3, "attrArray" : [ ] }, "$unset" : { "attr9" : true } } }
{ "o" : { "_id" : 279, "version" : 7 } }
{ "o" : { "$set" : { "version" : 8, "attrCounter" : 1, "a" : 1 } } }
{ "o" : { "$set" : { "version" : 9 }, "$unset" : { "a" : true, "attrCounter" : true } } }
</pre>
The archive collection contents (slightly formatted for readability):
<pre class="prettyprint lang-js">
db.docs_full_archive.find( {"docId":279}, {"_id":0} )
{ "docId" : 279, "version" : 1, "attr7" : "xxx279" }
{ "docId" : 279, "version" : 2, "attr7" : "xxx279" }
{ "docId" : 279, "version" : 3, "attr7" : "xxx279", 
   "attrCounter" : 1, "attr9" : 1, "attrArray" : [ "xxx" ] }
{ "docId" : 279, "version" : 4, "attr7" : "xxx279", 
   "attrCounter" : 1, "attr9" : 1, "attrArray" : [ "xxx" ], "attrNew" : "abc" }
{ "docId" : 279, "version" : 5, "attr7" : "xxx279", 
   "attrCounter" : 2, "attr9" : 1, "attrArray" : [ "xxx" ], "attrNewReplacement" : "abc" }
{ "docId" : 279, "version" : 6, "attr7" : "xxx279", 
   "attrCounter" : 3, "attr9" : 1, "attrArray" : [ ], "attrNewReplacement" : "abc" }
{ "docId" : 279, "version" : 7 }
{ "docId" : 279, "version" : 8, "attrCounter" : 1, "a" : 1 }
{ "docId" : 279, "version" : 9, "attrCounter" : 1, "a" : 1 }
</pre>

There you have it, my preferred way to isolate an infrequently used collection and keep it updated based on every write action that happens in the main DB.  I hope you can see how this can be extended for many different pub/sub needs as you can adapt your code to watch for different types of events on different collections.

Hope you found this educational and keep those questions coming!