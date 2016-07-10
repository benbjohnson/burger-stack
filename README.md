The Burger Stack: Simplifying Web Application Development
=========================================================

### Target Audience

Intermediate developers looking to simplify development.


### Introduction (5 sentences)

Gophers love to talk about how great Go is. And it is! I've used it to write
databases and distributed systems and command line tools. It's my favorite
language. But ever since I started writing Go there's been one area that
everyone advises against using Go: web development. And that's terrible. Web
applications are an enormous part of our industry!

Additionally, web development hasn't seen limited improvement over the years.
We started with the LAMP stack 20 years ago where we used slow languages like
PHP or Perl to connect to a difficult to configure MySQL server. 10 years later
we saw Rails where we used Ruby, a slow language, to connect to complex SQL
databases. Recently we've see the MEAN stack where we write JavaScript on the
backend to connect to MongoDB. It's almost like we've gone backwards in our
industry.

However, all is not lost. If we consider some recent developments in our
industry over the past few years, Go begins to stand out as a serious contender
within the web application space.


### Single Page Applications

One big flaw with Go is that it lacks a strong templating language. There is
the `template` package but it's difficult to use at scale for a large
application. However, in recent years there has been enormous adoption for
Single Page Applications (SPAs). We've seen several technologies come along such
as AngularJS, Backbone.js, & Ember.js but none of these have seen the crazy
adoption that Facebook's React has.

React does a lot of awesome UI stuff and if this was a JavaScript conference
then I'd talk about it more. However, it's important to note that React renders
client side so it only needs to consume an API. No templating is needed on the
server side.


### SQL Alternatives

Another flaw that Go has is that working with SQL databases is not much fun.
Rails has a pretty ORM built-in but you still need to know enough to optimize
things like N+1 queries. You also need to configure and maintain your database
server. Honestly, it feels like we adopted relational databases 40 years ago
as a standard and we've been doing whatever we can to avoid SQL ever since.

However, there is a much better way to avoid SQL, relational databases, and set
theory. Simply don't use them. Go excels at working with byte slices so instead
of mapping objects to relations, you can map your objects to bytes.

Now if you think that mapping to bytes is complex, don't worry, this is a
problem that computer scientists have devoted years of research to and it's
called "serialization". There are so many serialization libraries to choose
from: Protocol Buffers, FlatBuffers, Avro, MessagePack, JSON. This serialization
is essentially the schema for your data.

Once we're mapping to bytes, we don't need the high level features of most
databases. We can get rid of SQL. Since we have no SQL we can get rid of the
query planner and query cache. Let's go one step further and get rid of the
server altogether and use an embedded database. At this point we have the
simplest database possible: an embedded key/value store.

There are several key/value stores available but we want something in pure Go,
strong transactional support, and read optimized so we're going to talk about
the most amazing database ever created: BoltDB. Disclaimer: I'm the author of
BoltDB so I'm a little biased.

Bolt, Go, React: the BGR stack. That's all we need to build an extremely fast,
easy to develop, easy to deploy web application. Many of you are probably
familiar with builing APIs with Go and I'm not going to give a React talk at a
Go conference so let's look how you use Bolt for everyone's favorite web app
construct: CRUD.


### Domain Types & Serialization

The first thing you need in any web app is a way to manage users so let's build
a `UserStore`. We'll keep it simple and our user will have two fields: an ID
and a username.

We'll need to serialize this object too so let's use Protocol Buffers to do
that. I like protobufs because it's really fast and it has versioning built-in.
Here's our protobuf definition for our `User` type. Now we can add a
"generate" line in our source and it'll automatically regenerate whenever we
call `go generate`. I also like to put this generated code in my `internal`
package so it's hidden from the rest of my application and godoc.

Next, we'll implement the little known `BinaryMarshaler` and `BinaryUnmarshaler`
interfaces. To marshal, we populate our protobuf object and marshal it. To
unmarshal, we unmarshal our protobuf object and copy from it.


### Opening & initializing

Bolt databases are extremely simple. It's just a single file. You can open it
using `bolt.Open()` similar to how `os.Open()` works. Once our database is open,
we'll need a place to store our user data and Bolt has something called a
`Bucket` for this. A bucket is essentially a persisted map that maps unique
byte slice keys to byte slice values. I use buckets analogously to tables in a
relational database.

We can ensure that we have a "Users" bucket created by starting a transaction
and calling `CreateBucketIfNotExists()`.

Transactions are an important part of Bolt. There are two kinds: read-only and 
read-write. They are full ACID transactions with serializable isolation. That
means that they will have a snapshot view of the data from when the transaction
started. You can have an unlimited number of simultaneous read-only transactions
but only one read-write transaction can operate at a time.

We start a transaction using `Begin()` and the only parameter is whether the
transaction is writable. Make sure you ALWAYS defer rollback on your
transactions! Leaving a transaction open in the event of an early return or a
recovered panic will cause problems.


### Creating a user

Let's add a new user. Let's walk through the code to see how this is done.
First we start a read-write transaction. Next we'll grab our "Users" bucket.
One each bucket there exists an autoincrementing sequence similar. We can grab
a new ID for our user from there.

Since we've already implemented `BinaryMarshaler`, we can easily convert our
user to bytes. Then we'll map our ID to our encoded bytes using `Put()`.
Finally, we'll commit the transaction and return an error if it failed for any
reason. Again, these are true ACID transactions so any operations that occurred
will be rolled back if anything fails.

You'll notice in the `Put()` line I use a helper function called `itob()`. This
is short for integer to bytes. I use this a lot and it saves me 3 lines over
and over again. It also ensures that you're always using big endian encoding.
Endianness is simply how you encode an integer to bytes and we always use
big endian in Bolt because it makes our integer keys sortable.


### Reading a user

Once you have users in your system, you need to get them out. Let's walk through
it. Again, we start a transaction. This time it is read-only. It's important to
use read-only transactions for read-only operations so that it can scale.
Read-write transactions can only operate one at a time. Next we'll pull out our
bytes using the `Get()` method on our bucket. If there is no value then the
key doesn't exist and we can return a `nil` user. But if we get a value then
decode it and return it. The defer rollback will be called and close out the
transaction automatically.

If you want to read all our users you can use a cursor on your bucket. A cursor
is simply a way to iterate over a bucket. We'll start the transaction the same
as before, then grab a cursor, and then we can traverse the whole bucket in a
simple `for` loop. This calls `First()` to move the cursor to the beginning and
then calls `Next()` until there are no more keys. We'll unmarshal and append on
our slice for each user and return.


### Updating a user

We've covered the "C" and the "R" of CRUD, next is update. Update is a bit like
read and create put together. This time we'll use a different method in Bolt for
running transactions called `Update()`. This will run a function in the context
of a transaction. If the function returns an error, everything will rollback.
Otherwise it will commit. This can be really helpful if our operation only
returns an error.

We'll also group some of the operations in if/else statements so it's easier to
see related operations. First we grab the bucket and the value, just like in our
read operation. If it doesn't exist we'll return an error. Otherwise we'll
unmarshal the object. Once we have our object we can update it's state. Now with
the new state we'll marshal and overwrite our previous value. Finally, returning
`nil` at the end will cause our transaction to commit.


### Deleting a user.

Finally we come to the "D" of CRUD. This operation is incredibly simple. We
can use the `Update()` helper and simply delete the key from our bucket.

And that's it! Full set of CRUD operations in a handful of lines of code. And
the best part is that it's all Go.


### BoltDB in Practice

There's a lot more to applications than just writing code. There's testing,
deploying, and maintenance. Testing is quite simple. It's an embedded database
and a single file so you can simply create a temp file for your test and delete
it when the test is done. That also means that you can run your tests in
parallel.

When you deploy BoltDB, it's advised that you use SSDs. Bolt doesn't use
compactions so there's a lot of random writes. Another gotcha is that Bolt
returns byte slices that directly point to an mmap so these slices are only
valid until you close a transaction. One huge benefit to this mmap, though, is
that the data is in the OS page cache so it'll stay there even if you restart
your application. You can use graceful restarts to have zero downtime.

One big question about the maintenance side is how do you backup your data or
handle catastrophic hardware failures? Bolt transactions implement the
`io.WriterTo` interface so you can actually stream a full, consistent copy of
your database to any writer. My personal favorite is adding an HTTP endpoint
so that I can use `curl` as my backup tool.

This does leave a window for data loss between snapshots. There is async
streaming replication support coming in the future so that you can have a
failover server and you can minimize that data loss window to only a few
seconds. However, that is still in early stages. In the meantime you'll need
to evaluate what window is acceptable. An SSD can copy out your database at 
3 - 5 seconds per GB so it may be acceptable to take a snapshot hourly or even
every 10 minutes depending on the size of your database. Sharding your database
can also help keep it small.

Performance is always a question that comes up with any database. Most people
want the "fastest" database but it's difficult to evaluate speed without your
specific workload. Also, most people don't need all the performance they think
they do. Bolt is read optimized which is a good fit for most web applications.
I generally give people these ballpark numbers: 2K random writes/sec, or up to
450,000 writes/sec if you can sort and batch your writes. For reads, it really
depends on if your data is hot in memory. Many applications can fit their
working set in memory -- especially with 


### Conclusion (5 sentences)

I believe it's time to move past the slow, complex stacks that permeate through
our industry and adopt something simpler. I've shown how the BGR stack provides
a simple, fast environment for not only developing applications but also
deploying and scaling them. I'm excited to see the future of Go not just in the
distributed systems and low-level services but also in the huge space of web
development. Please, give the BGR stack a try on your next project!







### Key Takeaways (bullet points)

- LAMP & Rails provided great ORM and HTML templating tools
- MEAN tried to modernize with MongoDB & JavaScript but who wants to write JS server side?
- React has seen huge adoption so HTML templating is no longer needed
- BoltDB avoids the headaches of SQL & remote servers so no ORM needed
  - Vertical scaling works for a long time
- How to get started with BoltDB, sample CRUD operations
- BoltDB in production


