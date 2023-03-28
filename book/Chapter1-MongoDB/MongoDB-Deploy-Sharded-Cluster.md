# Deploy a Sharded Cluster

## Overview

This tutorial involves creating a new sharded cluster that consists of a [`mongos`](https://www.mongodb.com/docs/manual/reference/program/mongos/#mongodb-binary-bin.mongos), the config server replica set, and two shard replica sets. We'll setup on 2 data centers.

## Procedure

### Hostnames and Configuration

Edit `/etc/hosts`:

```
192.168.1.19 lap-node
192.168.1.15 hung-node
```

Because I have limited amount of server, so we will setup like this:

```
Item							server			port	primary
----------------------------------------------------------------------
configserver member 1			lap-node		27030	yes
configserver member 2			lap-node		27031
configserver member 3			hung-node		27032

mongos query router				lap-node		27040

shard 1 member 1				lap-node		27050	yes
shard 1 member 2				hung-node		27051
shard 2 member 1				hung-node		27060	yes
shard 2 member 2				lap-node		27061
```

And I'm assuming that the working directory in all data center is `/home/hadoop/` of user `hadoop`.

> One one of the server, for example `lap-node`, do the following:
>
> `================= START ===================`

### Generate a Key file

```bash
$ openssl rand -base64 756 > keyfile
$ chmod 400 keyfile
$ sudo chown hadoop keyfile
```

Make sure that the keyfile is owned by the same user that you use to run the mongod/mongos processes (in this example, it is user `hadoop`), otherwise setting the permissions above will cause an error.

### Create the Config Server Replica Set

For a production deployment, deploy a config server replica set with at least three members. For testing purposes, you can create a single-member replica set.

> Repeat this procedure for each member of the config server replica set. In this example, I'm working with the config server member 1.
>
> `================= START ================`

Create a folder to save data for the config server member 1:

```bash
$ mkdir example_configserver_replica_data_27030
```

Create a configuration file, for example `example_configserver_replica_27030.conf`:

```
sharding:
  clusterRole: configsvr
replication:
  replSetName: "example_configserver_replica"
net:
  port: 27030
  bindIp: 127.0.0.1,lap-node,hung-node
storage:
  dbPath: /home/hadoop/example_configserver_replica_data_27030
  journal:
      enabled: true
security:
  keyFile: /home/hadoop/keyfile


#  engine:
#  wiredTiger:

# where to write logging data.
# systemLog:
#   destination: file
#   logAppend: true
#   path: /var/log/mongodb/mongod.log
  
# how the process runs
# processManagement:
#  timeZoneInfo: /usr/share/zoneinfo
  
#security:

#operationProfiling:

## Enterprise-Only Options:

#auditLog:

#snmp:
```

* [`sharding.clusterRole`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-sharding.clusterRole) to `configsvr`. The `clusterRole` directive tells MongoDB that this server will be a part of the sharded cluster and will take the role of a config server (as indicated by the `configsvr` value)
* [`replication.replSetName`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-replication.replSetName) to the desired name of the config server replica set. In this example, we name it `example_configserver_replica`. The config server replica set ==must not use the same name== as any of the shard replica sets.
* [`net.bindIp`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.bindIp) option to the hostname or comma-delimited list of hostnames that remote clients (including the other members of the config server replica set as well as other members of the sharded cluster) can use to connect to the instance.
* Additional settings as appropriate to your deployment, such as [`storage.dbPath`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-storage.dbPath) and [`net.port`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.port). For more information on the configuration file, see [configuration options](https://www.mongodb.com/docs/manual/reference/configuration-options/)

Start the [`mongod`](https://www.mongodb.com/docs/manual/reference/program/mongod/#mongodb-binary-bin.mongod) with the `--config` option set to the created configuration file absolute path:

```bash
$ mongod --config /home/hadoop/example_configserver_replica_27030.conf
```


> if you got this error:
>
> ```
> mongod.service - MongoDB Database Server
> Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
> Active: failed (Result: exit-code) since Sun 2022-12-25 21:07:25 +07; 4s ago
> ```
>
> then:
>
> ```bash
> sudo chown -R mongodb:mongodb /var/lib/mongodb
> sudo chown mongodb:mongodb /tmp/mongodb-27017.sock
> sudo service mongod restart
> sudo systemctl status mongod
> ```
> If you got this error:
>
> ```
> {"t":{"$date":"2022-12-25T16:32:16.436Z"},"s":"F",  "c":"CONTROL",  "id":20574,   "ctx":"-","msg":"Error during global initialization","attr":{"error":{"code":38,"codeName":"FileNotOpen","errmsg":"Can't initialize rotatable log file :: caused by :: Failed to open /var/log/mongodb/mongod.log"}}}
> ```
>
> then:
>
> ```bash
> chmod 777 /var/log/mongodb/mongod.log
> ```

> `================ STOP ================`
>
> hmm vẫn phải là mongod.conf, cái bindIP chứa những IP có thể truy cập được vào cái này
>
> ```bash
> sudo lsof -i -P -n | grep LISTEN
> ```
>

### Initiate the replica set

Connect [`mongosh`](https://www.mongodb.com/docs/mongodb-shell/#mongodb-binary-bin.mongosh) to ==only== one of the config server members, for example `configserver member 1`:

```bash
$ mongosh --port 27030
```

Run the method:

```shell
> rs.initiate(
  {
    _id: "example_configserver_replica",
    configsvr: true,
    members: [
      { _id : 0, host : "lap-node:27030" },
      { _id : 1, host : "lap-node:27031" },
      { _id : 2, host : "hung-node:27032" }
    ]
  }
)
```

> If using TCP tunneling, then setup so that:
>
> ```
> lap-node			hung-node
> -------------------------------
> 27030		<----	27030
> 27031		<----	27031
> 27032		---->	27032
> ```
>
> then when initiate config server replica set:
>
> ```shell
> > rs.initiate(
>   {
>     _id: "example_configserver_replica",
>     configsvr: true,
>     members: [
>       { _id : 0, host : "localhost:27030" },
>       { _id : 1, host : "localhost:27031" },
>       { _id : 2, host : "localhost:27032" }
>     ]
>   }
> )
> ```
>
> Either all host names in a replica set configuration ==must be localhost references, or none must be==.

Notice that the MongoDB shell prompt has now changed to `PRIMARY` or `SECONDARY`, depending on which member you used to run the previous commands. To further verify that each host has been added to the replica set:

```shell
> rs.status()
```

### Create the Shard Replica Sets

The procedure is similar to the above. In this example, I'll demonstrate the steps for the `shard 1`.

On `lap-node`:

```bash
$ mkdir example_shard1_replica_data_27050
```

Create configuration file `example_shard1_replica_27050.conf`:

```
sharding:
  clusterRole: shardsvr 
replication:
  replSetName: "example_shard1_replica"
net:
  port: 27050
  bindIp: 127.0.0.1,lap-node,hung-node
storage:
  dbPath: /home/hadoop/example_shard1_replica_data_27050
  journal:
      enabled: true
security:
  keyFile: /home/hadoop/keyfile
```

```bash
$ mongod --config /home/hadoop/example_shard1_replica_27050.conf
```

> Do similarly for `shard 1 member 2` on `hung-node`. If using TCP tunneling:
>
> ```
> lap-node			hung-node
> -------------------------------
> 27050		<----	27050
> 27051		---->	27051
> ```

```bash
$ mongosh --port 27050
```

```shell
> rs.initiate(
  {
    _id: "example_shard1_replica",
    members: [
      { _id : 0, host : "lap-node:27050" },
      { _id : 1, host : "hung-node:27051" }
    ]
  }
)
```

> if using TCP tunneling:
>
> ```shell
> > rs.initiate(
>   {
>     _id: "example_shard1_replica",
>     members: [
>       { _id : 0, host : "localhost:27050" },
>       { _id : 1, host : "localhost:27051" }
>     ]
>   }
> )
> ```

Do similar for `shard 2`.

### Start a `mongos` for the Sharded Cluster

In this section, we’ll set up the MongoDB query router. The query router obtains metadata from the config servers, caches it, and uses that metadata to send read and write queries to the correct shards. It’s also possible to use a replica set of query routers, however in this example, we'll use only one.

Create a configuration file named `example_mongos_27040.conf`:

```
net:
  port: 27040
  bindIp: 127.0.0.1

security:
  keyFile: /home/hadoop/keyfile

sharding:
  configDB: example_configserver_replica/lap-node:27030,lap-node:27031,hung-node:27032
```

* set the [`sharding.configDB`](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-sharding.configDB) to the config server replica set name and at least one member of the replica set in `<replSetName>/<host:port>` format.
* `bindIp`: replace `127.0.0.1` with `localhost,<hostname(s)|ip address(es)>`.

> If using TCP tunneling:
>
> ```
> configDB: example_configserver_replica/localhost:27030,localhost:27031,localhost:27032
> ```
>

```bash
$ mongos --config /home/hadoop/example_mongos_27040.conf
```

Connect [`mongosh`](https://www.mongodb.com/docs/mongodb-shell/#mongodb-binary-bin.mongosh) to the [`mongos`](https://www.mongodb.com/docs/manual/reference/program/mongos/#mongodb-binary-bin.mongos). Specify the `host` and `port` on which the `mongos` is running:

```bash
$ mongosh --port 27040
```

Add shards to the cluster:

```shell
> sh.addShard("example_shard1_replica/lap-node:27050")
> sh.addShard("example_shard2_replica/hung-node:27060")
```

> If TCP tunneling:
>
> ```shell
> > sh.addShard("example_shard1_replica/localhost:27050")
> > sh.addShard("example_shard2_replica/localhost:27060")
> ```
>
> 

### Create Admin user

On one of the config server replica set, create admin user:

```bash
$ mongosh --port 27030
```

```shell
> use admin
> db.createUser({user: "admin", pwd: "1", roles:[{role: "root", db: "admin"}]})
```

From now on, we need to connect `mongosh` to the `mongos` with authentication:

```bash
$ mongosh --port 27040 -u admin -p 1 --authenticationDatabase admin
```

## Shard a Collection

In order to nable sharding at the database level, which means that collections in a given database {for example the `example` databse} can be distributed among different shards:

```bash
> sh.enableSharding("example")
```

Now, enable sharding for a collection. If the collection already contains data, you must [create an index](https://www.mongodb.com/docs/manual/reference/method/db.collection.createIndex/#std-label-method-createIndex) that supports the [shard key](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-shard-key) before sharding the collection. If the collection is empty, MongoDB creates the index as part of `sh.shardCollection()`.

MongoDB provides two strategies to shard collections:

- [Hashed sharding](https://www.mongodb.com/docs/manual/core/hashed-sharding/#std-label-sharding-hashed) uses a [hashed index](https://www.mongodb.com/docs/manual/core/index-hashed/#std-label-index-hashed-index) of a single field as the [shard key](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-shard-key) to partition data across your sharded cluster.

  ```shell
  sh.shardCollection("<database>.<collection>", { <shard key field> : "hashed" })
  ```

- [Range-based sharding](https://www.mongodb.com/docs/manual/core/ranged-sharding/#std-label-sharding-ranged) can use multiple fields as the shard key and divides data into contiguous ranges determined by the shard key values.

  ```shell
  sh.shardCollection("<database>.<collection>", { <shard key field> : 1, ... } )
  ```

For example, with the `sharded`  collection:
```shell
> sh.shardCollection( "example.sharded", { "_id" : "hashed" })
```

Show sharding information of the database `example`.

```shell
> use example
> db.stats()
> db.printShardingStatus()
> db.example.getShardDistribution()
```





## Further readings

* [MongoDB Deploy a Sharded Cluster](https://www.mongodb.com/docs/manual/tutorial/deploy-shard-cluster/)
* [MongDB Remove Members from Replica Set](https://www.mongodb.com/docs/manual/tutorial/remove-replica-set-member/)
* https://www.digitalocean.com/community/tutorials/how-to-use-sharding-in-mongodb
* https://www.linode.com/docs/guides/build-database-clusters-with-mongodb/
* https://viblo.asia/p/xay-dung-sharded-cluster-mongodb-Az45b4jLZxY