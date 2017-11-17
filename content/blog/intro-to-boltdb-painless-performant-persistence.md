+++
title = "Intro to BoltDB: Painless Performant Persistence"
date = 2014-07-07T08:25:00Z
updated = 2014-07-18T07:10:59Z
tags = ["persistence", "Go", "programming", "golang"]
blogimport = true 
type = "post"
[author]
	name = "Nate Finch"
	uri = "https://plus.google.com/115818189328363361527"
+++

<a href="http://github.com/boltdb/bolt" target="_blank">BoltDB</a> is a pure Go persistence solution that saves data to a memory mapped file.  I call it a persistence solution and not a database, because the word database has a lot of baggage associated with it that doesn't apply to bolt. And that lack of baggage is what makes bolt so awesome.<br /><br />Bolt is just a Go package.  There's nothing you need to install on the system, no configuration to figure out before you can start coding, nothing.  You just go get github.com/boltdb/bolt and then import "github.com/boltdb/bolt". <br /><br />All you need to fully use bolt as storage is a file name.  This is fantastic from both a developer's point of view, and a user's point of view.  I don't know about you, but I've spent months of work time over my career configuring and setting up databases and debugging configuration problems, users and permissions and all the other crap you get from more traditional databases like Postgres and Mongo.  There's none of that with bolt.  No users, no setup, just a file name.  This is also a boon for users of your application, because *they* don't have to futz with all that crap either.<br /><br />Bolt is not a relational database.  It's not even a document store, though you can sort of use it that way.  It's really just a key/value store... but don't worry if you don't really know what that means or how you'd use that for storage.  It's super simple and it's incredibly flexible.  Let's take a look.<br /><br />Storage in bolt is divided into buckets.  A bucket is simply a named collection of key/value pairs, just like Go's map.  The name of the bucket, the keys, and the values are all of type []byte.  Buckets can contain other buckets, also keyed by a []byte name. <br /><br />... that's it.  No, really, that's it.  Bolt is basically a bunch of nested maps.  And this simplicity is what makes it so easy to use.  There's no tables to set up, no schemas, no complex querying language to struggle with.  Let's look at a bolt hello world:<br /><br /><pre>package main<br /><br />import (<br />    "fmt"<br />    "log"<br /><br />    "github.com/boltdb/bolt"<br />)<br /><br />var world = []byte("world")<br /><br />func main() {<br />    db, err := bolt.Open("/home/nate/foo/bolt.db", 0644, nil)<br />    if err != nil {<br />        log.Fatal(err)<br />    }<br />    defer db.Close()<br /><br />    key := []byte("hello")<br />    value := []byte("Hello World!")<br /><br />    // store some data<br />    err = db.Update(func(tx *bolt.Tx) error {<br />        bucket, err := tx.CreateBucketIfNotExists(world)<br />        if err != nil {<br />            return err<br />        }<br /><br />        err = bucket.Put(key, value)<br />        if err != nil {<br />            return err<br />        }<br />        return nil<br />    })<br /><br />    if err != nil {<br />        log.Fatal(err)<br />    }<br /><br />    // retrieve the data<br />    err = db.View(func(tx *bolt.Tx) error {<br />        bucket := tx.Bucket(world)<br />        if bucket == nil {<br />            return fmt.Errorf("Bucket %q not found!", world)<br />        }<br /><br />        val := bucket.Get(key)<br />        fmt.Println(string(val))<br /><br />        return nil<br />    })<br /><br />    if err != nil {<br />        log.Fatal(err)<br />    }<br />}<br /><br />// output:<br />// Hello World!</pre><div></div>I know what you're thinking - that seems kinda long.  But keep in mind, I fully handled all errors in at least a semi-proper way, and we're doing all this:<br /><br />1.) creating a database <br />2.) creating some structure (the "world" bucket)<br />3.) storing data to the structure<br />4.) retrieving data from the structure.<br /><br />I think that's not too bad in 54 lines of code.<br /><br />So let's look at what that example is really doing.  First we call bolt.Open to get the database.  This will create the file if necessary, or open it if it exists.<br /><br />All reads from or writes to the bolt database must be done within a transaction. You can have as many Readers in read-only transactions at the same time as you want, but only one Writer in a writable transaction at a time (readers maintain a consistent view of the DB while writers are writing).<br /><br />To begin, we call db.Update, which takes a function to which it'll pass a bolt.Tx - bolt's transaction object.  We then create a Bucket (since all data in bolt lives in buckets), and add our key/value pair to it.  After the write transaction finishes, we start a read- only transaction with DB.View, and get the values back out.<br /><br />What's great about bolt's transaction mechanism is that it's super simple - the scope of the function is the scope of the transaction.  If the function passed to Update returns nil, all updates from the transaction are atomically stored to the database.  If the function passed to Update returns an error, the transaction is rolled back.  This makes bolt's transactions completely intuitive from a Go developer's point of view.  You just exit early out of your function by returning an error as usual, and bolt Does The Right Thing.  No need to worry about manually rolling back updates or anything, just return an error.<br /><br />The only other basic thing you may need is to iterate over key/value pairs in a Bucket, in which case, you just call bucket.Cursor(), which returns a Cursor value, which has functions like Next(), Prev() etc that return a key/value pair and work like you'd expect.<br /><br />There's a lot more to the bolt API, but most of the rest of it is more about database statistics and some stuff for more advanced usage scenarios... but the above is all you really need to know to start storing data in a bolt database.<br /><br />For a more complex application, just storing strings in the database may not be sufficient, but that's ok, Go has your back there, too.  You can easily use encoding/json or encoding/gob to serialize structs into the database, keyed by a unique name or id.  This is what makes it easy for bolt to go from a key/value store to a document store - just have one bucket per document type.  Again, the benefit of bolt is low barrier of entry.  You don't have to figure out a whole database schema or install anything to be able to just start dumping data to disk in a performant and manageable way.<br /><br />The main drawback of bolt is that there are no queries.  You can't say "give me all foo objects with a name that starts with bar".  You <i>could</i> make your own index in the database and keep it up to date manually.  This could be as easy as a slice of IDs serialized into an "indices" bucket for a particular query. Obviously, this is where you start getting into the realm of developing your own relational database, but if you don't go overboard, it can be nice that all this code is just that - code.  It's not queries in some external DSL, it's just code like you'd write for an in-memory data store.<br /><br />Bolt is not for every application.  You must understand your application's needs and if bolt's key/value style will be sufficient to fulfill those needs.  If it is, I think you'll be very happy to use such a simple data store with so little mental overhead.<br /><br />[edited to clarify reader/writer relationship]   Bonus Gob vs. Json benchmark for storing structs in Bolt: <pre><br />BenchmarkGobEncode  1000000       2191 ns/op<br />BenchmarkJsonEncode   500000       4738 ns/op<br />BenchmarkGobDecode  1000000       2019 ns/op<br />BenchmarkJsonDecode   200000      12993 ns/op<br /></pre>Code: http://play.golang.org/p/IvfDUGBpJ6 