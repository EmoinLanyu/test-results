# Test Summary

## Overall Summary

In conclusion, from the tests we learnt that zonal partial outages could cause severe performance issues to some services. A forced and controlled Global Failover could mitigate the issues quickly. To be more specific about "control", in addition to cordoning the nodes in the unhealthy zone and evicting the pods in it, some services might need to carry out extra operations before we apply network isolation.

## Performance Test Result Chart

### PostgreSQL Performance Test Result

![PostgreSQL Performance Test Result](PostgreSQL_performance_test_results.png)

### Solace Software Broker Performance Test Result

![Solace Software Broker Performance Test Result](Solace_performance_test_results.png)

*NOTE: for SSB failover scenarios, the failbacks might fail if the active instance has not released the AD-Lock. In order to guarantee a successful failover, some manual steps are needed for the graceful shutdown of the unhealthy instance. Please refer to [this document](https://docs.solace.com/Features/DR-Replication/Perf-Con-Fail-Over.htm) for more details about the manual steps.*

### Kube-APIServer Performance Test Result

![Kube-APIServer Performance Test Result](Kube_APIServer_performance_test_results.png)

*Note: for kube-apiserver failover scenarios, when network latency reached two seconds, ETCD cluster failover happened.*

## Test Summaries Per Application

### Solace Software Brokers

When there is partial outage, the performance of the Solace Software Brokers (SSB) will be severely affected. Though Solace makes sure no message is lost, it is very likely that the durable Queue is full because subscribers are not able to consume the messages. Thus making the publishers fail to send out any new message.

As observed from our tests with network latency and packet loss, only latency over 2 seconds or packet loss over 60% will trigger automatic failover of the SSB.

A forced Global Failover could quickly mitigate the performance issue. But as proved by our various tests:
- Directly adding network ACL to SSB instances could possibly make failover to fail in many occasions.
- By cordoning the node of the active instance and evicting its pod before adding network ACL, in some cases, e.g. under constant stable network latency, the unhealthy instance could be gracefully shutdown and the failover succeeds. However, when there is network packet loss or network latency with jitter, failover could fail.

Solace support has responded to us that, in order for SSB to successfully fail over, the AD-Lock on active instance must be released, which means a graceful shutdown of the active instance is needed. Solace official documentation provides a [guide](https://docs.solace.com/Features/DR-Replication/Perf-Con-Fail-Over.htm) for controlled failover in Disaster Recovery scenario. They doesn't have one for HA scenario, but the process might be similar.

In a nutshell, for Solace Software Brokers, manual steps are needed in order for a controlled failover to happen before we apply network isolation.

### CrunchyData PostgreSQL

When PostgreSQL cluster is set to default / asynchronous mode, network interruptions will only affect the performance of the cluster when they happen on the active instance. However, if PostgreSQL cluster is set to synchronous mode, network interruptions will affect the performance of the cluster no matter they happen on the active or the stand-by instance.

With one millisecond network latency, we could see obvious performance downgrade. With ten milliseconds network latency, the performance becomes around one-fourth of the original. With over two hundred milliseconds network latency, the cluster is almost not usable because almost no database operations can be successfully submitted.

As for failover scenarios, with CrunchyData PostgreSQL cluster HA setup, no automatic failover will happen. This means the performance downgrade, no matter how severe it is, will persist until the infrastructure zonal partial outage is gone. In addition, if the HA cluster is setup in synchronous mode, even if the active node is not affected, the performance of the service will be affected nonetheless.

A controlled forced failover, by cordoning unhealthy nodes, evicting unhealthy pods, and isolating unhealthy zone via network ACL, can mitigate the issue.

### Kube-APIServer with ETCD Cluster

For ETCD cluster, when partial outage happens in the zone where the ETCD leader is, the performance of the whole service will be affected. When partial outage happens in the zone where the ETCD follower is, only the user (e.g., kube-apiserver) that connects to this unhealthy follower will be affected because the write operations to the ETCD follower will be forwarded to the ETCD leader. Disk IO latency with greater than one second will cause ETCD cluster to fail over.

Usually, when forty percent of disk IO has three hundred milliseconds latency, the average latency for one request reaches two seconds.

In the case with kube-apiserver, when the apiserver happens to connect to the unhealthy ETCD instance or the unhealthy ETCD instance is the leader, its performance will be affected accordingly. The problem can be mitigated with apiserver's liveness probe. In the latter case, the problem can only be solved when failover happens for ETCD cluster.

The liveness probe of kube-apiserver is often configured with a two-second timeout, which means with aforementioned forty percent of disk IO having three hundred milliseconds latency, kube-apiserver's health check may run into continuous timeouts that leads to apiserver's restarts. The fact that kube-apiserver talks to ETCD cluster in a kept-alive round robin way makes it possible for it to connect to a healthy instance and self-heal from the outage.

As for ETCD cluster automatic failover, when the disk IO latency on leader reaches one second, it is more likely that a failover happens.

All in all, for kube-apiserver with ETCD cluster, it is possible for the HA setup to survive a partial zonal outage without human interventions. However, as shown by our tests, when the outage was not severe enough, for example, with thirty percent disk IO having two hundred milliseconds latency, the performance was still greatly influenced (each API request could have several seconds of latency). In this case, Global Failover may still be useful.
