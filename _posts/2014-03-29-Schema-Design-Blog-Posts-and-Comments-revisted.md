Title: Schema Design: Blog Posts and Comments revisited.
Date: 2014-03-29 08:07 
Type: post
Published: true
Slug: :EmbedOrLink
Tags: schema,modeling,embed,link,mongodb,arrays

### Question:

I have a question about whether I should store comments inside the blog post entry or in a separate collection. It'd be nice to see examples of how to access various fields in both cases, how to index and in general how to know when to embed and when to link.

### Answer:

#### The Blog Schema

There has been a lot of discussion and write-ups about how to model a simple blog that allows comments on posts - it's a fairly simple example that everyone can understand, and at the same time it offers several opportunities to choose different ways to structure the schema.  The example usually consists of four concepts: users(or authors), posts, tags on posts and comments on posts.

##### Authors

Typically everyone agrees that the authors or users are stored in a  collection of their own where you keep their information - everything from their username, password, when they last logged in, when they signed up for the service, etc.

##### Posts

There is also little argument that posts should be stored separately from authors - I don't think I've ever heard anyone advocate for embedding posts within author document - that makes no sense for many reason, not the least of them are the fact that you want to avoid unbounded growth of the author document, and querying over posts is a natural function of the use case so posts really should be first class object.

What isn't always agreed on is whether the author of the post should have just their unique primary key (or username) saved in each post or whether some of the information, like their full name, should also be denormalized into each post.


##### Tags

Tags being simple strings should be stored inside the post document.  The it advantage of document model over relational is that it allows you to embed an array with multiple values without sacrificing the ability to index the tags:

    { "_id" : <Id>,
        "author" :  { "id" :  <authorId>, "name" :  "Asya Kamsky" },
        "tags" :  [ "schema", "embed", "link" ],
        ...
    }

We can index tags with `db.posts.ensureIndex( { tags:1 } )` which will be used in queries like 
    db.posts.find( { "tags" : { "$in":  ["schema", "performance"] } } )

You probably noticed that I happen to think it's right to denormalize the author's name into the post - I'm a strong believer in optimizing for the common case, not exceptional one[^fn-f1] and I think optimizing query performance is more important than trying to minimize storage at the cost of performance.

##### Comments

Comment documents, or rather where to store them, usually generates the most discussion and disagreement. 

Let's consider both options and see what we can gain from each:

###### embed comments
    {
         _id: <Id>,
         author: { id: <authorId>, name: "Asya Kamsky" },
         tags: [ "schema", "embed", "link" ],
         comments: [
             { author : { id:<authorId>,name:"Joe Shmoe"}, 
               date:ISODate(" "), 
               text:"Blah Blah Blah" },
             { author : { id:<authorId>,name:"Jane Doe"}, 
               date:ISODate(" "), 
               text:"Blah Blah Blah" },
             { author : { id:<authorId>,name:"Asya Kamsky"}, 
               date:ISODate(" "), 
               text:"Blah Blah Blah" },
             ...
         ]
    }
    
In addition to other indexes we already plan to have on posts, we will probably need to add several indexes to support querying for comments or by comments.  For example, when someone logs in, I can see wanting to show them all the threads/posts that they commented on, which means we need to index on "comments.author.id" so that we can query for posts that this author commented on.  We also might need to include fields inside the comments array to track which comments are responses to which other comments, and the biggest downside of them all, if the discussion in comments gets really heated, we will end up with a huge array inside this post.

###### have separate comments collection 
    {  post : <postId>,
        author : { id:<authorId>,name:"Joe Shmoe"}, 
        date:ISODate(" "), 
        text:"Blah Blah Blah" 
    }
A collection of comments would have to have an index on the postId so that we can look up the comments for a particular post, probably compound index with date so that we can query for the most recent posts.  We would want to index author.id and date as well.  But the nice thing is that here we can control how many comments we want returned, and even though querying for all comments for a post might involve some random IO, we can minimize it by only querying for as many comments as we intend to display.  The fact is that most of the time the reader of the blog post won't even look at the comments, and if they do then they might read a few and never click on "show more" which we would normally have.

Is there a third option?

###### hybrid option
The nice thing about flexible schema is that in cases like these you can keep comments in separate collection but also choose to denormalize some small number of comments into the post itself, either first few or the last few or whatever fits your requirements best.

This hybrid approach may be analogous to the product collection for an e-commerce site where they store reviews of products separately from the product itself, but keep the highest voted reviews  (one positive and one negative) embedded in the product. This is a good schema because when you display the product, you want to display a few most helpful reviews, but you don't need to display all the reviews at that time.

#### Summary
The general principal to use when trying to decide between embedding and linking is this: 
- consider which objects are first class entities and which are properties of such entities
- consider what your use case requires to display fast and what allows for additional queries
- when two choices both seem to be viable, prototype both and see which works better

[^fn-f1]: Someone always brings up the possibility that the author will change their name, as if that's an everyday occurrence