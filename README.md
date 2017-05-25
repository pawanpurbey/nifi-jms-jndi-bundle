# nifi-jms-jndi-bundle

A bundle of processors that leverage JNDI to lookup JMS resources (connection factory &amp; destinations). Supports Point-to-Point and Pub/Sub messaging models.

The default JMS support in NiFi

- Creates an instance of the Connection Factory using the provided class name through reflection
- Uses [Destination references](https://docs.oracle.com/cd/E23943_01/web.1111/e13727/implement.htm#JMSPG203) by default.

> JIRA https://issues.apache.org/jira/browse/NIFI-2701

Neither of the above approaches work with certain traditional and commercial JMS implementations such as WebLogic JMS Server and SonicMQ.
Moreover, they allow central configuration and monitoring of CF and Destinations
(including value-added features like HA - clustering and service migration features, ordering etc.).
These are made available through JNDI.

> WebLogic also supports using a "Reference" instead of looking up Destinations through JNDI service.
> Look at "Use a Reference" section from the [documentation here](https://docs.oracle.com/cd/E23943_01/web.1111/e13727/implement.htm#JMSPG203) .

ConnectionFactory
- Encapsulates connection config information 
- Supports shared concurrent use 
- Can be deployed on specific independent servers, or on specific servers within a cluster, or an entire cluster

Connection
- Represents an open communication channel between an application and the messaging system
    - creates server and client side objects that manage the activity between application and JMS
- Provides authentication    
- Used to create a `Session`
- Connections supports concurrent use

Session
- Defines serial order for the messages produced and consumed
- Can create multiple producers and consumers
- Not thread-safe

Transacted sessions
- Only one transaction is active at any given time
- Any number of messages sent or received during transaction are treated as atomic unit
- Acknowledge mode is ignored. `commit` acknowledges all the messages consumed during that transaction

Destinations
- Supports concurrent use
- distributed destination https://docs.oracle.com/cd/E23943_01/web.1111/e13727/fund.htm#i1061461
    - A single *set* of destinations
    - Can be looked up in the JNDI
    - Members of the set are typically distributed across multiple servers within a cluster
    - Each member belongs to a seoarate JMS server
- apps using distributed destinations are HA
    - WebLogic JMS provides load balancing and failover
    
Message
- Headers
- Properties (JMSXUserID, JMSXDeliveryCount, JMSXGroupId, JMSXGroupSeq)
- Body

## JMS Application Design Best Practices

| Topic | Points to remember |
|-------|--------------------|
| Distributed Queue | If a QueueSender is created using the JNDI name of a distributed queue, any message sent from that QueueSender is delivered to only one physical queue destination. A decision is made every time a message is sent determining which member will get the message. This decision is based on a load-balancing algorithm - When a QueueReceiver is created using a distributed queue name, it will connect to a physical queue member and, unlike a QueueSender, remain pinned to that destination until the QueueReceiver loses connection.|
| Distributed Topic | JMS client applications can create topic message producers and consumers using a distributed topic name. Your application doesn't know how many physical destination members the distributed topic has, which eliminates any special programming considerations when using this kind of clustered JMS resource.|
| Message Design | Use Bytes where possible (Strings are expensive during serialization) - Avoid large string properties - non-persistent messages arent serialized by WL JMS server and hence no cost is incurred|
| Message Compression | Saves network bandwdith - reduces memory requirement of JMS server - reduces the size of persistent writes  - compression threshold can be set during the creation of connection factory |
| Message Properties and Headers | Use standard headers and properties where possible - avoid large values - these aren't paged out and hence consume memory - each message uses 512 bytes (for headers and properties) of JVM memory even when paged out |
| Message Ordering| Unit-of-Order - Can be configured administratively during creation of QCF and Destination - Preserves ordering during processing delays, rollbacks or session recovery|
| Sync/Async Consumers | Less network traffic by pipelining messages (aggregating multiple messages into a single network call) - MDB is async; dont block the thread - Sync consumers can take advantage of pipelining by using PREFETCH mode - Prefetch mode can be configured on QCF |
| Persistnet vs Non-Persistent | Can be configured on CF and Destination - Can be specified at producer level and also per message |
| JMSXUserId | Configure CF or Destination to auto-populate this to maintain data lineage |

[Recover from a Server Failure](https://docs.oracle.com/cd/E23943_01/web.1111/e13727/recover.htm)

Connection Factory
- A ConnectionFactory object looked up via JNDI (see Step 1 in Example 14-1 and Example 14-2) is re-usable after a server or network failure without requiring a re-lookup.
- A Destination object (queue or topic) looked up via JNDI (see Step 2 in Example 14-1 and Example 14-2) is re-usable after a server or network failure without requiring another lookup. The same principle applies to producers that send to a distributed destinations, since the client looks up the distributed destination in JNDI, and not the unavailable distributed member.


Producer
- Producers will transparently failover to another server instance if one is available
- Connection, Session and MessageProducer will try to automatically reconnect without any configuration or coding efforts

Consumer