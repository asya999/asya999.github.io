Title: How to Merge Shapes with Aggregation Framework
Date: 2014-05-24
Type: post
Published: true
Slug: :MergeShapes
Tags: schema,modeling,aggregation,versions

### Question:

Consider two separate shapes of data like this in a single collection:

    {   type: "A",
        level: 0,
        color: "red",
        locale: "USA"
    }
    {   type: "A",
        level: 1,
        color: "blue"
    }

The goal is to present a merged shape to the application with the level n data overridden by level n+1 if level n+1 data exists for type A, starting with n = 0.  In other words, the app wants to see this shape:

    {   type: "A",
        level: 1, 
        color: "blue",
        locale: "USA"
    }

If no level 1 data exists, the app would see the default (level 0) shape.   Think of it as a layered merge.

### Answer:
In the [previous "AskAsya" on tracking versions][1] we looked at different ways of tracking all versions of changing objects, and this happens to be a complex variant of that problem that we considered as "schema 4" - it's a possible approach to versioning, but it presents an interesting challenge returning the "full" current object back to the client.   

[1]: http://askasya.com/post/trackversions

#### Merging Different Shapes

This problem would be easily solved with aggregation framework query, except for the problem that we need to know the names of all the keys/fields, and we might not  know all of the possible fields that could exist in our documents. Without this information, the only way we have of merging documents is using MapReduce, which is both more complex _and_ slower.   I will show both solutions and I'll leave it up to you to determine which will be more performant in your scenario (or whether you want to switch to a different versioning schema).

##### Aggregation Framework
This will be the fastest way if you either have all possible attribute names that your documents could have, or get them via a scan of the entire collection (note that the latter immediately becomes stale, as new documents with new attributes could show up as soon as you start querying, but that's inherently an issue that always exists in any system that doesn't provide repeatable read isolation).

Get the possible attribute names (I'm assuming `type` and `level` are your 'id' and 'version'):

    var att = { };
    var attrs = [ ];
    db.coll.find( {}, {_id:0, type:0, level:0} ).forEach( function(d) {
        for ( i in d)
             if ( !att.hasOwnProperty(i) ) {
                 att[i]=1;
                 attrs.push(i);
             }
    } );                   

You now have an array `attrs` which holds all the strings representing different attributes in your collection.

We now programmatically generate stage for `$project` that turns each attribute into a subdocument with its level first and attribute itself second.  

    proj1={$project:{type:1, level:1}};
    attrs.forEach(function(attr) { 
        _a="_"+attr; 
        a="$"+attr;   
        proj1["$project"][_a]={}; 
        proj1["$project"][_a]["l"]={"$cond":{}};
        proj1["$project"][_a]["l"]["$cond"]={if:{"$gt":[a,null]},then:"$level",else:-1};
        proj1["$project"][_a][attr]=a;
    } );

Since levels are increasing, this set us to be able to `$group` using the `$max` operator to keep the highest level for each attribute.
 
    group={$group:{_id:"$type",lvl:{$max:"$level"}}};
    attrs.forEach(function(attr) { 
        a="$_"+attr;
        group["$group"][attr]={"$max":a};
    } )

The last `$project` just transforms the fields of our document back into the same key names they had before.

    proj2={$project:{_id:0,type:"$_id", level:"$lvl"}}
    attrs.forEach(function(attr) {
        a="$"+attr;  
        proj2["$project"][attr]=a+"."+attr;
    } )

We are now all set to run the aggregation with your programmatically generated stages:

    db.coll.aggregate( proj1, group, proj2 );
       
To recap,`proj1` is the stage where we converted every attribute into a subdocument which included "level" (first) and attribute value (second).  If a given attribute didn't exist in a document, it went in with level:-1 and value:null.  

`group` is where we grouped by `type` which is our `docId` and kept the highest (max) "subdocument" for each possible attribute.  This works because MongoDB allows you to compare any types (including BSON) and level:-1 is always going to "lose" to a higher level.  

`proj2` is when we turned all the fields into readable format, or at least format resembling our initial document.

This now returned to us the merged documents.

If we had original documents like these:

    > db.coll.find({},{_id:0}).sort({type:1,level:1})
    { "type" : "A", "level" : 0, "color" : "red", "locale" : "USA" }
    { "type" : "A", "level" : 1, "color" : "blue" }
    { "type" : "A", "level" : 2, "priority" : 5 }
    { "type" : "A", "level" : 3, "locale" : "EMEA" }
    { "type" : "B", "level" : 0, "priority" : 1 }
    { "type" : "B", "level" : 1, "color" : "purple", "locale" : "Canada" }
    { "type" : "B", "level" : 2, "color" : "green" }
    { "type" : "B", "level" : 3, "priority" : 2, "locale" : "USA" }
    { "type" : "B", "level" : 4, "color" : "NONE" }

We got back results that looked like this:

    > db.coll.aggregate( proj1, group, proj2 );
    { "color" : "NONE", "locale" : "USA", "priority" : 2, "type" : "B", "level" : 4 }
    { "color" : "blue", "locale" : "EMEA", "priority" : 5, "type" : "A", "level" : 3 }

Note that this is not performant for filtering on attributes since we can't apply the filter until we have "merged" all the documents, and that means that indexes can't be used effectively.  While this aggregation may be a good exercise, unless you are saving this output into a new collection that you then index by attributes for querying, it won't be a good schema if you need very fast responses.

Here is MapReduce for the same functionality:

    map = function () {
        var doc=this;
        delete(doc._id);
        var level=this.level;
        delete(doc.level);
        var t=this.type;
        delete(doc.type);
        for (i in doc) {
             val={level:level};
             val[i]={ l:level, v:doc[i]};
             emit(t, val);
        }
    }

    reduce = function (key,values) {
      result={level:-1};
      values.forEach(function(val) {
        if (result.level<val.level) result.level=val.level;
        var attr=null;
        for (a in val) if (a!="level") { attr=a; break; }
        if (!result.hasOwnProperty(attr) || result[attr].l<=val[attr].l) {
              result[attr]=val[attr];

        }
      })
      return result;
    }

