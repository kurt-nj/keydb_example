# Key DB

## Active/Active

```yaml
active-replica yes
replica-read-only no
replicaof <node> <port>
```

Active-active replication seems to work best with TWO nodes. This is because behind the scenes
keydb is still using a primary replica setup. Each node can only have ONE primary. When this node
goes down there is no built in auto failover. What active-active adds is that a node can have the
same node as both its primary and replica. Basically giving you a hotswap.

## Replicas

```yaml
replica-read-only yes
replicaof <node> <port>
```

Outside of the writable active-active, you can also have a number of read only replicas. But how do
you handle the issue of only one primary that won't fail over?

## Proxy

If we introduce a proxy for the replicas to point at, then when a primary fails the replicas can then
be failed over to the other running primary by the proxy.

In this example we are using HAProxy due to the available [example](https://docs.keydb.dev/docs/haproxy/)
combined with this [docker-compose example](http://www.inanzzz.com/index.php/post/w14j/creating-a-single-haproxy-and-two-apache-containers-with-docker-compose).

## Testing

```
docker-compose up -d
keydb-cli set key1 val1
```
The exposed default port is going to the proxy so the value will be written to one of the primary nodes
```
keydb-cli -p 6401 get key1
keydb-cli -p 6402 get key1
```
Will verify that the key is in both primaries
```
keydb-cli -p 6501 get key1
keydb-cli -p 6502 get key1
keydb-cli -p 6503 get key1
```
Will verify that the key is in all the replicas
```
keydb-cli -p 6501 info replication
keydb-cli -p 6502 info replication
keydb-cli -p 6503 info replication
```
Will give a primary 'master_replid'. One of the replicas should have a different value (roundrobin proxy).
```
docker-compose stop primary1
```
Will kill one primary
```
keydb-cli -p 6501 info replication
keydb-cli -p 6502 info replication
keydb-cli -p 6503 info replication
```
Wait a moment and all the replicas will switch to the same primary and show 'up'
You can continue setting keys and then bring up the down primary and see that it will
sync back up.