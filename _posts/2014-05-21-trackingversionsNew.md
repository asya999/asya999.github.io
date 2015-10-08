Title: How to Track Versions with MongoDB
Date: 2014-05-21
Type: post
Published: true
Slug: :TrackVersions
Tags: schema,modeling,indexes,versioning

### Question: ###
Consider requirement that we have to be able to recreate/query any version of a document that ever existed in a particular collection.   So we start out with:

    {   docId: "A",
        version: 1,
        color: "red",
        locale: "USA"
    }

If we need to set color to "blue", instead of updating the "color" field from "red" to "blue", we have to create a new version of the document which now has its full "current" state, and preserve somehow the old version of the document.   So we insert

    {   docId: "A",
        version: 2,
        color: "blue",
        locale: "USA"
    }
    
The goal is to preserve every state for each object, but we only respond to queries with the "current" or "latest" version, we just have a requirement to be able to have an audit (which would be very infrequent so it's okay if it's slow).    Is keeping each version as we do in this example the best schema/approach to this problem?

### Answer: ###
Versioning can be tricky to get right if you don't know all of the requirements of the application and approximate expected loads for various operations.  I'll lay out a few possible approaches and point out their strength and weaknesses.

#### Problem Statement: ####
In some systems, rather than updating an existing object and overwriting its various attributes there is a business requirement to preserve the original document and to create a new version of this document, instead of updating it.  This raises the following interesting challenges:

1. You must correctly generate the new version number in a multithreaded system
2. You must return only the current version of each document when there is a query
3. You must "update" correctly by including all current attributes in addition to newly provided attributes
4. If the system fails at any point, you must either have a consistent state of the data, or it must be possible on re-start to infer the state of the data and clean it up, or otherwise bring it to consistent state.

##### Possible Schema Approaches: #####

 1. Store full document each write with monotonically increasing version number. 
      - 1a. possibily with a field in latest version identifying it as such.
 2. Store all document versions inside a single document.
 3. Store current document in your "primary" collection, and keep previous versions in a second collection.
 4. Store only "deltas" with each increasing version.  

##### Generating correct version number
No matter which schema you choose, the issue of generating the correct "higher" version number will come down to one of three possible approaches:

- maintain a separate collection which hands out the next version for each document. This is probably the worst approach as it can be prone to contention and edge cases in multi-shard, multithreaded environment.
- use optimistic locking to read the current document, increment its version and save new document contingent on some constraint keeping you from succeeding simultaneously from two different threads.  This can be handled differently for different schemas, and it's definitely a common and feasible approach.
- use a fine-grain timestamp - current time to the millisecond may be good enough, depending on your ability to synchronize all the clocks, unfortunately it only works with two of our four schema options.

I like optimistic locking option and it works well with all four of our schema options.  It involves a "compare-and-swap" technique where you read the current document,  do the appropriate calculation of new version and adding new attributes, and then try the insert or update, contingent on no one having updated the document ahead of you.  If it's an update, you include parts of the original document in your query condition, and if it's an insert you must have a unique constraint to prevent success of multiple simultaneous attempts to version the same document.   In both cases you must check the result of your write - knowing if it succeeded or failed is how the thread knows it has to try again. 

#### How does each schema: ####
- return only the current document
- generate new version number to "update" existing and add new attributes
    + this includes recovering from failure in the middle of a set of operations (if there is more than one)

***

##### Choice 1
Store full document each time there is a write with monotonically increasing version number inside.   
       
        {  "docId" : 174, "v" : 1,  "attr1": 165 }   /*version 1 */
        {  "docId" : 174, "v" : 2,  "attr1": 165, "attr2": "A-1" } 
        {  "docId" : 174, "v" : 3,  "attr1": 184, "attr2" : "A-1" }
        
For each docId value the document with the highest "v" represents the full current object state.  In this example, docId 174 v:3 represents the total current state.   There is a unique index on `{"docId":1,"v":1}`
   
###### To return only current document
If the query is for a single docId, the query would be:

    db.docs.find({"docId":174}).sort({"v":-1}).limit(-1);
    
This will find the documents with "docId" 174 and return the one that has the highest "v" only.  This will efficiently use our index on docId and v to only scan a single document.  The `-1` for `limit` just tells the server to close the cursor when the document is returned as we are done with it (it's what `findOne` functionally does under the covers).

But what if you want to query for all documents that match a particular condition, but what you expect is that only the latest version of each document would be returned?   Now you have to use the aggregation framework to "merge" your document set to only keep those with the highest version and apply your filter then:

Careful that you don't do it the **wrong** way:

    db.docs.aggregate( [  
        {"$match":{<your-match-condition>}}, /* WRONG */
        {"$sort":{"docId":-1,"v":-1}},
        {"$group":{"_id":"$docId","doc":{"$first":"$$ROOT"}}}
    ] )

Note that applying your filter *before* you filter out all but the latest version of each document may not return the current version of some of the documents!   So **don't** do this!

Instead we have to do this:

    sort={"$sort": { "documentId" : 1, "version" : -1 } };
    group={"$group" : { "_id" : "$documentId",
                        "doc": { "$first" : "$$ROOT" }
                       } };
    match={"$match":{"doc.attrN":<value>}}; /* RIGHT */
    db.collection.aggregate( sort, group, match )
    
This is not efficient, sadly, as the indexes on our collection can only be effectively used **before** the first group operation.  
    
###### To generate new version and update
Optimistic locking: each thread reads in the most current document (in this case `docId:174, v:3`) makes attribute changes, increments `v` by one and then tries the insert of this document:

    db.docs.insert({"docId":174,"v":4, "attr1":184,"attr2":"A-1","attr3":"blue"})
    
If the insert succeeds, it's done, but if it gets a unique constraint violation, it means another thread has already inserted a new version of this document, and this thread needs to try again (making sure to read the new `"v":4"` or whatever the latest version of the document is and trying its change till it succeeds.

Failure does _not_ create an inconsistent state, since there is only a single write.
   
##### Choice 1a
In a variant of 1. we might add a field in the "current" version of each documentId
 
         {  "docId" : 174, "v" : 1,  "attr1": 165 }
         {  "docId" : 174, "v" : 2,  "attr1": 165, "attr2": "A-1" }
         {  "docId" : 174, "v" : 3,  "attr1": 184, "attr2" : "A-1", current: true }
         
Fetching the current document should now be easy, just query for { docId:174, current:true }, and when querying for multiple documents, add {"current":true} to the query predicate, solving that problem.
      
Updating becomes difficult now, since there is no method to insert one document and update another document "as one".  So we would want to first insert a new version, based on currently highest version, and then  update the previously current document to $unset the "current" field.
      
Now if our process fails between those two write operations, we will have two documents for a particular docId that have "current" set to true and that means all of our queries will have to guard against that possibility - that  seems unnecessarily complex, so let's hold off on this method for now.

##### Choice 2
Store all document versions inside a single document.
        
        {  "_id" : 174, "current" : { "v" :3, "attr1": 184, "attr2" : "A-1" },
            "prev" : [ 
                  {  "v" : 1,  "attr1": 165 },
                  {  "v" : 2,  "attr1": 165, "attr2": "A-1" }
            ]
        }
           
For each docId, the current state is represented by its "current" subdocument.   Since "docId" is unique it can be stored in the "_id" field.

The merits of multiple possible solutions mostly depend on how many versions of each object you expect to have and how long you have to keep them.  If the lifetime of a document usually has a number of versions that are in the single digits or low double digits, you can do well embedding the versions inside of the single document that represents each object.  
###### To return only current document
Since there is only one document, we just search by docId or other attributes of "current" and we use projection to exclude the previous array: `db.collection.find({"docId":174}, {"prev":0})` except in cases we want to see it.
###### To generate new version and update
Even though you only have one document per docId, you really can't create a new version with a single update, since it depends on knowing what the current subdocument is, but what you can do is utilize the "compare-and-swap" technique where you read the current document, move "current" field to the end of the previous array, set the new "current" field appropriately, and then do the update **contingent on no one having updated the document ahead of you**:

    var doc = db.collection.findOne( { "_id" : 174 });
    /* save the current version */
    var currVersion = doc.current.v;
    /* push the current subdocument to the end of prev array */
    doc.prev.push(doc.current);
    /* construct the new "current" subdocument */
    doc.current = { "v" : currVersion+1, "attr1" : <value>, "attr2" : <value> }
    var result = db.collection.update( { "_id" : 174, "current.v" : currVersion },  { "$set" : doc } )
    if (result.nModified != 1) {
        print("Someone must have gotten there first, re-fetch the new document, try again");
    }

###### atomicity and maintainability 
There are many pros in this approach:
- atomic updates of document (single operation both, sets new current and updates previous)
- querying only needs to happen on "current" attributes, since you only need to access previous infrequently
- it's simple to return just current attributes or to exclude previous from being returned to the application
- the `_id` index can be used for the unique docId preventing duplicates being accidentally inserted
- creating the next version number is simple and thread-safe

The cons are all performance based:
- when each document grows beyond its previously allocated size, it has to be moved which makes some updates more time consuming
- this won't work at all for documents that have thousands of versions over their lifetime (the documents would get too big, unwieldy, and could potentially exceed 16MB limit)
- there'll likely be more fragmentation than with the more "naive" approach of inserting new versions as new documents
- if the system has a very high level of concurrency when multiple threads are trying to each make different update to a specific document, a single thread could keep getting beat and it might take multiple re-tries to persist its update

The last "con" shouldn't really be a concern if you are genuinely talking about a system where each document is only going to have a handful of versions (or updates) in its lifecycle, and this schema isn't appropriate for systems where a single document will live through hundreds (or thousands) of updates.
             
##### Choice 3
Store current document in your "primary" collection, and keep older versions in another collection.
        
        CurrentCollection:     {  "docId" : 174,  "v" :3, "attr1": 184, "attr2" : "A-1" }
        CollectionOfPrevious:
                  {  "docId" : 174, "v" : 1,  "attr1": 165 }
                  {  "docId" : 174, "v" : 2,  "attr1": 165, "attr2": "A-1" }
           
    For each documentId, there is only one document which represented its current state.
###### To return only current document
This one is the simplest of them all.  Since the old versions are in another collection, you just query normally when you need to find a single or multiple documents - they will all be the current version.
###### To generate new version and update
This one may be the hardest.  Technically, you only need to do a few things: read the current document, construct out of it the new current document, save the new current document on top of previous one and if successful, then insert the old current document into the "previous" collection.   Of course we would use the same compare-and-swap update to make sure that no one changed the document between our read and our write, and only insert into previous collection if we successfully update current.

The problem is that you may fail before that last write and now you'll be missing a version of this document from the "previous" collection.  

What if we  switch the order of writes to save into the previous collection first?   We read the current document, we write it to previous collection, we now change it to be "new" current and save it into "current" collection.  This has several advantages:
- if someone else is trying to update this document, they will also be saving into "previous" collection, so having a unique index on docId, version will tell us if we lost the race and now have to try again.  
- if the thread dies in the middle (after insert into previous and before updating current) it's not the end of the world, as your current collection was not affected, but you do need a way to "clean up" your "previous" collection, first because you need to remove the version of the document that never existed in "current" and second because it will block all other "updates" on this document by using an invalid "docId", "version".

Luckily, clean-up may be simple, as any time we detect that there exists a docId, version in "current" that also exists in "previous" it means either there is an update "in progress" or it means that an update "died" and we should clean up.  Of course the devil is in the details - and it could cause delays in the system since you have to wait long enough to be sure that this "in progress" update actually died.  Or you can have another field that you update after successful writes in both places (now making it easier to recover, but needing to do three writes before you're done with a single document update!)  Let's call this Medium Hard still. 
      
##### Choice 4
Store only "deltas" with increasing versions

        {  "documentId" : 174, "v" : 1,  "attr1": 165 }
        {  "documentId" : 174, "v" : 2,  "attr2": "A-1" }
        {  "documentId" : 174, "v" : 3,  "attr1": 184 }

For each docId, the current state must be derived by "merging" all the documents with matching docId, keeping the "latest" or "highest" version's value of each attribute if it occurs in more than one version.
  
This one is quite complex, and the answer is getting quite long, so let's call it hard and defer the details of how we accomplish this merge till [next time](http://askasya.com/post/mergeshapes).

#### Let's Summarize
Here's a table that shows for each schema choice how well we can handle the reads, writes and if an update has to make more than one write, how easy it is to recover or to be in a relatively "safe" state:

|Schema | Fetch 1 | Fetch Many || Update | Recover if fail|
------------ | ------------- | ------------|-|---------------|-----------------|-------------
|1) New doc for each | Easy,Fast  | Not easy,Slow | | Medium | N/A |
|1a) New doc with "current" | Easy,Fast  | Easy,Fast | | Medium | Hard |
|2) Embedded in single doc | Easy,Fastest  | Easy,Fastest | | Medium | N/A |
|3) Sep Collection for prev. |  Easy,Fastest  | Easy,Fastest  | | Medium |  Medium Hard |
|4) Deltas only in new doc | TBD/Hard | TBD/Hard | | Medium | N/A |
|?) TBD |  Easy,Fastest  | Easy,Fastest  | | Easy,Fastest |  N/A |

"N/A" for recovery means there is no inconsistent state possible - if we only have to make one write to create/add a new version, we are safe from any inconsistency.  So "N/A" is the "easiest" value there.  

I deferred describing what it would take to query the "Store deltas only" option till the next "Ask Asya" but let me foreshadow and tell you that it's not particularly easy - it involves a long and tricky aggregation.

But you can see I filled in a yet undescribed way that magically somehow makes all our tasks easy, and yet seems to not have any performance issues nor consistency problems.  If you've stuck with me this far, I promise that I will describe the magical "winner" for version keeping in the next "Ask Asya" after the one that shows how to aggregate deltas of document into one.

Since I just enabled comments and discussion on these pages, if you see a possible schema approach I didn't mention, feel free to suggest it.  Free "MongoDB" t-shirt for you if you guess the "TBD" schema I have in mind.