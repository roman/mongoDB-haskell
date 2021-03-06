MongoDB Haskell Mini Tutorial
-----------------------------

  __Updated:__ Oct 2010

This is a mini tutorial to get you up and going with the basics
of the Haskell mongoDB drivers. You will need the mongoDB driver
installed as well as mongo itself. Prompts used in this tutorial are:

    $ = command line prompt
    > = ghci repl prompt


Installing Haskell Bindings
---------------------------

From Hackage using cabal:

    $ cabal install mongoDB

From Source:

    $ git clone git://github.com/TonyGen/mongoDB-haskell.git mongoDB
    $ cd mongoDB
    $ runhaskell Setup.hs configure
    $ runhaskell Setup.hs build
    $ runhaskell Setup.hs install

Getting Ready
-------------

Start a MongoDB instance for us to play with in a separate terminal window:

    $ mongod --dbpath <directory where Mongo will store the data>

Start up a haskell repl:

    $ ghci

Import the MongoDB driver library, and set
OverloadedStrings so literal strings are converted to UTF-8 automatically.

    > import Database.MongoDB
    > :set -XOverloadedStrings

Making A Connection
-------------------
Create a connection pool for your mongo server, using the standard port (27017):

    > pool <- newConnPool 1 $ host "127.0.0.1"

or for a non-standard port

    > pool <- newConnPool 1 $ Host "127.0.0.1" (PortNumber 30000)

*newConnPool* takes the connection pool size and the host to connect to. It returns
a *ConnPool*, which is a potential pool of TCP connections. They are not created until first
access, so it is not possible to get a connection error here.

Note, plain IO code in this driver never raises an exception unless it invokes third party IO
code that does. Driver code that may throw an exception says so in its Monad type,
for example, *ErrorT IOError IO a*.

Access monad
-------------------

A query/update executes in an *Access* monad, which has access to a
*Pipe*, *WriteMode*, and read-mode (*MasterSlaveOk*), and may throw a *Failure*.
A Pipe is a single TCP connection.

To run an Access action (monad), supply WriteMode, MasterOrSlaveOk, Connection,
and action to *access*. For example, to get a list of all the database on the server:

    > access safe Master conn allDatabases

*access* return either Left Failure or Right result. Failure means there was a connection failure
or a read or write exception like cursor expired or duplicate key insert.

Since we are working in ghci, which requires us to start from the
IO monad every time, we'll define a convenient *run* function that takes an
action and executes it against our "test" database on the server we
just connected to, with typical write and read mode:

    > let run action = access safe Master pool $ use (Database "test") action

*use* adds a *Database* to the action context, so query/update operations know which
database to operate on.

Databases and Collections
-----------------------------

MongoDB can store multiple databases -- separate namespaces
under which collections reside.

As before, you can obtain the list of databases available on a connection:

    > run allDatabases

The "test" database in context is ignored in this case because *allDatabases*
is not a query on a specific database but on the server as a whole.

Databases and collections do not need to be created, just start using
them and MongoDB will automatically create them for you. In the below examples
we'll be using the database "test" (captured in *run* above) and the colllection "posts".

You can obtain a list of all collections in the "test" database:

    > run allCollections

Documents
---------

Data in MongoDB is represented (and stored) using JSON-style
documents, called BSON documents. A *Document" is simply a list of *Field*s,
where each field is a named value. A *Value" is a basic type like Bool, Int, Float, String, Time;
a special BSON value like Binary, Javascript, ObjectId; a (embedded)
Document; or a list of values. Here's an example document which could
represent a blog post:

    > import Data.Time
    > now <- getCurrentTime
    > :{
      let post = ["author" =: "Mike",
                  "text" =: "My first blog post!",
                  "tags" =: ["mongoDB", "Haskell"],
                  "date" =: now]
      :}

Inserting a Document
-------------------

To insert a document into a collection we can use the *insert* function:

    > run $ insert "posts" post

When a document is inserted a special field, *_id*, is automatically
added if the document doesn't already contain that field. The value
of *_id* must be unique across the collection. *insert* returns the
value of *_id* for the inserted document. For more information, see
the [documentation on _id](http://www.mongodb.org/display/DOCS/Object+IDs).

After inserting the first document, the posts collection has actually
been created on the server. We can verify this by listing all of the
collections in our database:

    > run allCollections

Note, the system.indexes collection is a special internal collection
that was created automatically.

Getting a single document with findOne
-------------------------------------

The most basic type of query that can be performed in MongoDB is
*findOne*. This method returns a single document matching a query (or
*Nothing* if there are no matches). It is useful when you know there is
only one matching document, or are only interested in the first
match. Here we use *findOne* to get the first document from the posts
collection:

    > run $ findOne (select [] "posts")

The result is a document matching the one that we inserted previously.
Note, the returned document contains the *_id* field, which was automatically
added on insert.

*findOne* also supports querying on specific elements that the
resulting document must match. To limit our results to a document with
author "Mike" we do:

    > run $ findOne (select ["author" =: "Mike"] "posts")

If we try with a different author, like "Eliot", we'll get no result:

    > run $ findOne (select ["author" =: "Eliot"] "posts")

Bulk Inserts
------------

In order to make querying a little more interesting, let's insert a
few more documents. In addition to inserting a single document, we can
also perform bulk insert operations, by using the *insertMany* function
which accepts a list of documents to be inserted. It send only a single
command to the server:

    > now <- getCurrentTime
    > :{
      let post1 = ["author" =: "Mike",
                   "text" =: "Another post!",
                   "tags" =: ["bulk", "insert"],
                   "date" =: now]
      :}
    > :{
      let post2 = ["author" =: "Eliot",
                   "title" =: "MongoDB is fun",
                   "text" =: "and pretty easy too!",
                   "date" =: now]
      :}
    > run $ insertMany "posts" [post1, post2]

* Note that post2 has a different shape than the other posts - there
is no "tags" field and we've added a new field, "title". This is what we
mean when we say that MongoDB is schema-free.

Querying for More Than One Document
------------------------------------

To get more than a single document as the result of a query we use the
*find* method. *find* returns a *Cursor*, which allows us to
iterate over all matching documents. There are several ways in which
we can iterate: we can call *next* to get documents one at a time
or we can get all the results by applying the cursor to *rest*:

    > Right cursor <- run $ find (select ["author" =: "Mike"] "posts")
    > run $ rest cursor

Of course you can use bind (*>>=*) to combine these into one line:

    > run $ find (select ["author" =: "Mike"] "posts") >>= rest

Note, *next* automatically closes the cursor when the last
document has been read out of it. Similarly, *rest* automatically
closes the cursor after returning all the results.

Counting
--------

We can count how many documents are in an entire collection:

    > run $ count (select [] "posts")

Or count how many documents match a query:

    > run $ count (select ["author" =: "Mike"] "posts")

Advanced Queries
-------------

To do

Indexing
--------

To do
