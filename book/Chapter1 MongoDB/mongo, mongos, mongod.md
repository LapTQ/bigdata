# `mongo` vs `mongosh` vs `mongos` vs `mongod` 

The MongoDB is divided into a two components *server* and *client*.

* *server* is the main database component which stores and manages data.
* *client* comes in various flavours and connect to the server to perform various queries and db operations.

Here, `mongod` is the server component. You start it, it runs, that’s it.

> By definition we also call it the *primary daemon process* for the MongoDB database.

Whereas `mongosh` is a command line client. You start it, you connect to a server, you enter commands, you exit out of it. Since the beginning, `mongo` is the MongoDB interactive shell, but as MongoDB grows, they needed to design a new MongoDB Shell, called `mongosh`. You have to run `mongod` first, otherwise you have no database to interact with.

As for `mongos`, see [MongoDB Cluster](./"MongoDB Clusters.md")

## Further readings

* [
Introducing the new MongoDB Shell](https://www.mongodb.com/blog/post/introducing-the-new-shell)