## Connection pooling

### Basics

The driver communicates with Cassandra over TCP, using the Cassandra
binary protocol. This protocol is asynchronous, which allows each TCP
connection to handle multiple simultaneous requests:

* when a query gets executed, a *stream id* gets assigned to it. It is a
  unique identifier for the current connection;
* the driver writes a request containing the stream id and the query on
  the connection, and then proceeds without waiting for the response (if
  you're using the asynchronous API, this is when the driver will send you
  back a `ResultSetFuture`). Once the request has been written to the
  connection, we say that it is *in flight*;
* at some point, Cassandra will send back a response on the connection.
  This response also contains the stream id, which allows the driver to
  trigger a callback that will complete the corresponding query (this is
  the point where your `ResultSetFuture` will get completed).

Version 2.0.x of the driver uses version 1 of the binary protocol when
connecting to Cassandra 1.2, and version 2 when connecting to Cassandra
2.0 or higher. In both cases, that gives you **128 stream ids per
connection**.

Because the number of stream ids per connection is limited, we use a
pool with multiple connections to each host. **For each `Session`
object, there is one connection pool per connected host**. The number of
connections per pool depends on the configuration, this will be
described in the next section.

```ditaa
+-------+1   n+-------+1   n+----+1   n+----------+1   128+-------+
|Cluster+-----+Session+-----+Pool+-----+Connection+-------+Request+
+-------+     +-------+     +----+     +----------+       +-------+
```

### Configuring the connection pool

Connections pools are configured with a `PoolingOptions` object, which
is global to a `Cluster` instance. You can pass that object when
building the cluster:

```java
PoolingOptions poolingOptions = new PoolingOptions();
// customize options...

Cluster cluster = Cluster.builder()
    .withContactPoints("127.0.0.1")
    .withPoolingOptions(poolingOptions)
    .build();
```

Most options can also be changed at runtime. If you don't have a
reference to the `PoolingOptions` instance, here's how you can get it:

```java
PoolingOptions poolingOptions = cluster.getConfiguration().getPoolingOptions();
// customize options...
```

#### Pool size

Connection pools have a variable size, which gets adjusted automatically
depending on the current load. There will always be at least a *core*
number of connections, and at most a *max* number. These values can be
configured independently by host *distance* (the distance is determined
by your `LoadBalancingPolicy`, and will generally indicate whether a
host is in the same datacenter or not).

```java
poolingOptions
    .setCoreConnectionsPerHost(HostDistance.LOCAL,  4)
    .setMaxConnectionsPerHost( HostDistance.LOCAL, 10)
    .setCoreConnectionsPerHost(HostDistance.REMOTE, 2)
    .setMaxConnectionsPerHost( HostDistance.REMOTE, 4);
```

Note: the setters always enforce core <= max, which can get annoying
when you try to set both values at once.
[JAVA-662](https://datastax-oss.atlassian.net/browse/JAVA-662) will
address this issue.

`PoolingOptions.setMaxSimultaneousRequestsPerConnectionThreshold` has a
slightly misleading name: it does not limit the number of requests per
connection, but only determines the threshold that triggers the creation
of a new connection when the pool is not at its maximum capacity. In
general, you shouldn't have to change its default value.


#### Heartbeat

If connections stay idle for too long, they might be dropped by
intermediate network devices (routers, firewalls...). Normally, TCP
keepalive should take care of this; but tweaking low-level keepalive
settings might be impractical in some environments.

The driver provides application-side keepalive in the form of a
connection heartbeat: when a connection has been idle for a given amount
of time, the driver will simulate activity by writing a dummy request to
it.

This feature is enabled by default. The default heartbeat interval is 30
seconds, it can be customized with the following method:

```java
poolingOptions.setHeartbeatIntervalSeconds(60);
```

If it gets changed at runtime, only connections created after that will
use the new interval. Most users will want to do this at startup.

The heartbeat interval should be set higher than
[SocketOptions.readTimeoutMillis](http://www.datastax.com/drivers/java/2.1/com/datastax/driver/core/SocketOptions.html#getReadTimeoutMillis()):
the read timeout is the maximum time that the driver waits for a regular
query to complete, therefore the connection should not be considered
idle before it has elapsed.

To disable heartbeat, set the interval to 0.

Implementation note: the dummy request sent by heartbeat is an
[OPTIONS](https://github.com/apache/cassandra/blob/trunk/doc/native_protocol_v3.spec#L278)
message.


#### Acquisition timeout

When the driver tries to send a request to a host, it will first try to
acquire a connection from this host's pool. If the pool is busy (i.e.
all connections are already handling their maximum number of in flight
requests), the client thread will block for a while, until a connection
becomes available (note that this will block even if you're using the
asynchronous API, like `Session.executeAsync`).

The time that the driver blocks is controlled by
`PoolingOptions.setPoolTimeoutMillis`. If there is still no connection
available after this timeout, the driver will try the next host.

For some applications, blocking is not acceptable, and it is preferable
to fail fast if the request cannot be fulfilled. If that's your case,
set the pool timeout to 0. If all hosts are busy, you will get a
`NoHostAvailableException` (if you look at the exception's details, you
will see a `java.util.concurrent.TimeoutException` for each host).


### Monitoring and tuning the pool

The easiest way to monitor pool usage is with `Session.getState`. Here's
a simple example that will print the number of open connections, active
requests, and maximum capacity for each host, every 5 seconds:

```java
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(1);
scheduled.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        Session.State state = session.getState();
        for (Host host : state.getConnectedHosts()) {
            int connections = state.getOpenConnections(host);
            int inFlightQueries = state.getInFlightQueries(host);
            System.out.printf("%s connections=%d current load=%d max load=%d%n",
                host, connections, inFlightQueries, connections * 128);
        }
    }
}, 5, 5, TimeUnit.SECONDS);
```

In real life, you'll probably want something more sophisticated, like
exposing a JMX MBean or sending the data to your favorite monitoring
tool.

If you find that the current load stays close or equal to the maximum
load at all time, it's a sign that your connection pools are saturated
and you should raise the max connections per host. On the other hand, if
the load is often less than core * 128, your pools are underused and you
could get away with less core connections.
