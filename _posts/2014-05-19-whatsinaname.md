Title: What Does FindAndModify Do
Date: 2014-05-19
Type: post
Published: true
Slug: :findAndModify
Tags: ops,update,findAndModify,schema

### Question:

I saw [your answer on SO][1] about the difference between "update" and "findAndModify", could you explain in more detail what the difference is, and why MongoDB findAndModify is named what it is?

[1]: http://stackoverflow.com/questions/10778493/whats-the-diff-between-findandmodify-and-update-in-mongodb/10778994#10778994

### Answer:

 _What's in a name?  that which we call a rose  
 By any other name would smell as sweet_;
<p style="text-align:right" markdown="1"> - said Juliet[^1] </p>  
  

[^1]: Actually [William_Shakespeare][2] said it. 

[2]: http://en.wikipedia.org/wiki/William_Shakespeare

  
As it turns out, a lot is in a name.  A poorly chosen name can confuse many users, year after year.  I believe `findAndModify` was probably not the best name for the role that it plays.

##### update
An [update][3] finds an appropriate document (by default it's just one) and then it changes its contents according to your specification.

[3]: http://docs.mongodb.org/manual/reference/method/db.collection.update/

##### findAndModify
The [findAndModify command][4] finds an appropriate document (it's always just one) and then it changes its contents according to your specification and _then it returns that exact document that it changed_ (old version or new version, depending on which you ask for)

[4]: http://docs.mongodb.org/manual/reference/command/findAndModify/#dbcmd.findAndModify

##### What's the Difference?

They both find a document and update it atomically.  What that means is that it's not possible for another thread to change part of this document between the time we find it and start updating it and when we finish updating it.   It also means that no other thread will see this document in "half-updated" state.  That's what "atomic" means - all-or-nothing.

##### Why do we even need `findAndModify` then?

What if we need to get the full document that we just updated (like marking an item in a queue "yours" and then working on it)?

What I said on [StackOverflow](http://stackoverflow.com) was:

> If you fetch an item and then update it, there may be an update by another thread between those two steps. If you update an item first and then fetch it, there may be another update in-between and you will get back a different item than what you updated.

> Doing it "atomically" means you are guaranteed that you are getting back the exact same item you are updating - i.e. no other operation can happen in between.  

That's why you'll hear people talk about `findAndModify` in the context of implementing a queue mechanism - `findAndModify` can update a single document to indicate that you are now working on it, and return that same document to you in one operation.

##### When Not to Use `findAndModify`

There are scenarios where `findAndModify` cannot help you.   If you need to update a document based on existing values of a document, you can use many [update operators][6] which are atomic and allow you to change a field value without knowing what its current value is, like `$inc` and `$addToSet` and `$min`  and `$max`, etc.  They allow you to modify a field without having to read the value of that field first.  And they work with a regular `update` as well as with `findAndModify`. 

But if you need to set field `a1` based on the current value of the field `b2` then you would have to read the document first and then when executing your update, you would have to ensure that the update is conditional on no one else having changed that document in the meantime and/or by having unique constraints to guarantee it.

There is no way to utilize `findAndModify` here, because it's limited to the exact set of operators that `update` uses, all it adds is the ability to return the exact document you modified.  Of course, `findAndModify` has to do more work than `update` so for best performance you should only use `findAndModify` when you must have the document that you just updated back in the application.   If you just want to know if an `update` succeeded, you can examine the [WriteResult][7] that update returns.



[6]: http://docs.mongodb.org/manual/reference/operator/update/#id1

[7]: http://docs.mongodb.org/manual/reference/command/update/#output

 
### Proposal

Let's rename `findAndModify` to a name that more accurately describes its function.  It updates a document and returns it, but to maintain a small connection to its current name, I nearby propose we rename it:

## modifyAndReturn

Who's with me? [^2]

[^2]: Please vote for [SERVER-13979][a] if you agree.

[a]: https://jira.mongodb.org/browse/SERVER-13979