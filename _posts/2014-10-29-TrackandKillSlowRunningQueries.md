Title: How to Find and Kill Slow Running Queries
Date: 2014-10-29
Type: post
Published: true
Slug: :FindKillSlowQueries
Tags: performance,operations,mongodb

### Question:

Is there a way that I can prevent slow queries in MongoDB?  I'd like to be able to set something on the server that will kill all queries running longer than a certain amount of time.

### Answer:

There are some options available on the **client** side, for example [$maxTimeMS][1] starting in 2.6 release.  This gives you a way to inject an option to your queries *before* they get to the server that tells the server to kill this query if it takes longer than a certain amount of time.  However, this does not help with any query which already got to the server without having this option added to it.

On the **server** side, there is no global option, because it would impact all databases and all operations, even ones that the system needs to be long running for internal operation (for example tailing the oplog for replication).  In addition, it may be okay for some of your queries to be longer running by design but not others.

The correct way to solve this would be to monitor currently running queries via a script and kill the ones that are both long running *and* user/client initiated - you can then build in exceptions for queries that are long running by design, or have different thresholds for different queries/collections/etc.

The way to implement this script is by using [db.currentOp() command][2] (in the shell) to see all currently running operations.  The field "secs_running" indicates how long the operation has been running.  Other fields visible to you will be the namespace ("ns") whether the operation is "active", whether it's waiting on a lock and for how long.   The [docs contain some good examples][3].   

Be careful not to kill any long running operations that are not initiated by your application/client - it may be a necessary system operation, like chunk migration in a sharded cluster as just one example, replication threads would be another.


[1]: http://docs.mongodb.org/manual/reference/operator/meta/maxTimeMS/
[2]: http://docs.mongodb.org/manual/reference/method/db.currentOp/
[3]: http://docs.mongodb.org/manual/reference/method/db.currentOp/#examples