# Test Summary

## Overall Summary

In conclusion, from the tests we learnt that zonal partial outages could cause severe performance issues to some services. A well designed cloud native service that leverages Kubernetes features could gracefully handle the outages and fail over when necessary. However, for many services, especially the legacy applications, it is not easy to quickly become "cloud native". For these services, a forced and controlled Global Failover could mitigate the issues quickly. To be more specific about "control", in addition to draining the nodes in the unhealthy zone, some services might need to carry out extra operations before we apply network isolation. This is where a webhook / callback, prepared by corresponding service team, kicks in that we could call before starting our workflow during a Global Failover.

## Performance Test Result Chart

### PostgreSQL Performance Test Result

![PostgreSQL Performance Test Result](PostgreSQL_performance_test_results.png)

### Solace Software Broker Performance Test Result

![Solace Software Broker Performance Test Result](Solace_performance_test_results.png)

*NOTE: for SSB failover scenarios, the failbacks might fail if the active instance has not released the AD-Lock. In order to guarantee a successful failover, some manual steps are needed for the graceful shutdown of the unhealthy instance. Please refer to [this document](https://docs.solace.com/Features/DR-Replication/Perf-Con-Fail-Over.htm) for more details about the manual steps.*

### Kube-APIServer Performance Test Result

![Kube-APIServer Performance Test Result](Kube_APIServer_performance_test_results.png)

*NOTE: for kube-apiserver failover scenarios, when network latency reached two seconds, Etcd cluster failover happened. Test result here comes from a kube-apiserver connected to a Etcd cluster bootstrapped by [Etcd Druid](https://github.com/gardener/etcd-druid). The readiness probe of the Etcd cluster StatefulSet was manually modified otherwise the failover will fail when latency reaches three seconds.*

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

### Kube-APIServer with Etcd Cluster

***NOTE: The detailed result analysis and summary is kept [here](extended_summary_for_etcd_and_kube_apiserver_tests.md) to keep this overall summary clean. Hovever, it is recommended that readers check it out for better understanding of below statements.***

Well designed cloud native applications such as Etcd and kube-apiserver are able to gracefully handle outages and fail over when necessary. In addition, by correctly using the liveness probe and readiness probe that K8s provides, we can automatically trigger failovers of applications and remove the unhealthy instances from the service.
