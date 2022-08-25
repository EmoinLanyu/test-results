# Extended Test Summary for Etcd and Kube-APIServer Tests

## Table of Content

- [Extended Test Summary for Etcd and Kube-APIServer Tests](#extended-test-summary-for-etcd-and-kube-apiserver-tests)
  - [Table of Content](#table-of-content)
  - [Test Setup](#test-setup)
  - [Test Results](#test-results)
    - [Our Test Results](#our-test-results)
      - [With Network Latency](#with-network-latency)
        - [Our Enhancement to Etcd's Readiness Probe](#our-enhancement-to-etcds-readiness-probe)
      - [With Disk IO Latency](#with-disk-io-latency)
  - [Test Summary](#test-summary)
    - [Etcd Failover](#etcd-failover)
    - [Kube-APIServer Failover](#kube-apiserver-failover)
    - [Conclusions](#conclusions)

## Test Setup

We used [Etcd Druid](https://github.com/gardener/etcd-druid) to bootstrap an Etcd cluster with 3 instances. A kube-apiserver was also deployed and connected to the Etcd cluster.

We configured readiness probe and liveness probe for the kube-apiserver. Readiness probe and liveness probe for the Etcd instances were by default configured by Etcd Druid.

## Test Results

### Our Test Results

#### With Network Latency

In our tests with above setup, we observed performance downgrade when network latency was injected. When the latency reached and went over one second, Etcd failovers were more likely to happen. When the latency was between one second and two seconds, we could see the Etcd cluster failed over. And when the kube-apiserver was restarted, the performance issue was resolved.

However, when the kube-apiserver was not restarted, the performance issue could still persist depending on how bad the outage was. In addition, with default Etcd cluster configuration, when latency reached two seconds, though we could see that Etcd cluster had successfully failed over, the kube-apiserver was still unavailable even if it had restarted.

We looked closer at the problem and found that:

- Etcd server's health check need to check the current leader of its cluster. And the health check has by default one second timeout. So when network latency happens and reaches one second, Etcd instance's health check is likely to fail.
- Kubelet is constantly running readiness probes against the unhealthy Etcd Pod.
  - When latency was below two seconds and over one second, the probe command `curl http://<etcd-name>-local:2379/health` always returned `{ "health" : "false" }` with non-zero exit code. The check also took around one second to complete.
  - However, when latancy reached two seconds, the command still returned `{ "health" : "false" }` but with exit code zero. Moreover, the check then was finished instantly.
  
There could be a circut breaker for Etcd server's health check, so that when network latency is too much, the result is returned immediately. This mechanism itself is not a problem, however, since the Etcd cluster bootstrapped by Etcd Druid is using the command execution probe, it only checks the exit code of that specific command. In this case, the exit code was always zero when latency was high.

This has caused the readiness probe to always succeed when latency is high, thus causing the unhealthy instance's addresses to stay in the Etcd Endpoint. Because of this, kube-proxy still maintains the unhealthy destination in the iptable rules, so when applications use ClusterIP to connect to the Etcd cluster, their connections could still be established with the unhealthy instance and their performance are still affected.

##### Our Enhancement to Etcd's Readiness Probe

We manually modified the readiness probe of the Etcd container. Instead of directly checking the exit code of `curl http://<etcd-name>-local:2379/health`, we did something like:

```bash
bash -c 'check=$(curl -s http://<etcd-name>-local:2379/health) | grep "true" && if [[ "$check" == "" ]]; then return 1; else return 0; fi'
```

This successfully solved the issue, and the readiness probe failed when latency reached two seconds. Therefore the unhealthy Etcd instance's addresses were successfully removed from the Endpoint, and restarted kube-apiserver were no longer suffering from the performance issue.

#### With Disk IO Latency

In our tests with above setup, we observed performance downgrade when disk IO latency was injected. When the latency reached and went over one second, Etcd failovers were more likely to happen. With readiness probe in place, unhealthy endpoints will be removed from the Endpoint when several health checks fail in a row.

We configured our disk IO chaos testing to have a certain percentage of IO operations to be affected by the latency. So during the tests, it was possible that some health check succeeded. With `successThreshold` of the readiness probe set to one, we observed that the unhealthy endpoints were periodically removed from and added back to the Endpoint. In this case, the kube-apiserver that is using the Etcd cluster was still affected by the outage.

## Test Summary

### Etcd Failover

For Etcd cluster, when partial outage happens in the zone where the Etcd leader is, the performance of the whole service will be affected (the write operations to the Etcd follower will be forwarded to the Etcd leader). When partial outage happens in the zone where the Etcd follower is, only the service (e.g., kube-apiserver) that connects to this unhealthy follower will be affected. With default configurations, disk IO latency as well as network latency greater than one second will cause Etcd cluster to fail over.

In general there are two ways to connect to an Etcd cluster. The impact on the performance of the applications that use Etcd is different for each method.

- Connect via ClusterIP
  
  In default mode, kube-proxy maintains the iptable rules on the node. All the Etcd instance destinations are maintained in the iptable chain. So when an application tries to establish a new TCP connection to the Etcd cluster, one of the instance will be chosen (randomly or in a round-robin way), including the unhealthy one. For those that connect to the unhealthy instance, the performance downgrade will persist until they switch to connect to another instance.

  However, if the readiness probe of the Etcd pod is well designed, kubelet can detect when an Etcd instance is unhealthy, and marks the Pod as "not ready", thus removing the unhealthy instance endpoints from the Etcd Endpoint.

- Connect via Pod fully qualified domain name endpoints

  Application that connects to Etcd cluster using Etcd client via Pod's FQDN endpoints will establish a TCP connection for each endpoint. And depending on which load balancer it is using, the impact would be slightly different. For example, if there are three Etcd instances, one of whom is unhealthy, and the application uses round-robin load balancer, one third of the requests will be sent to the unhealthy instance.

  Depending on the version of Etcd client used, the behavior might be different (ref. [here](https://etcd.io/docs/v3.5/learning/design-client/)).

  In case of the kube-apiserver (as of K8s v1.25.0), it is using Etcd clients (v3.5.4) to connect to Etcd clusters with round-robin load balancer.

### Kube-APIServer Failover

The connection established between kube-apiserver and Etcd is kept-alive. By default, every thirty seconds a heartbeat will be sent. Timeout for the heartbeat is by default ten seconds. If the heartbeat server does not reply in ten seconds, the TCP connection will be dropped and a new one will be established.

Usually, we will configure liveness probe and readiness probe for the kube-apiserver. By default, if the kube-apiserver could not get a response from the Etcd cluster in two seconds, a health check would fail. After several consequential unsuccessful health check, the kube-apiserver Pod will be restarted.

Kube-APIServer establishes one TCP connection to Etcd cluster per API resource (which means there might be over a hundred connections). When an Etcd instance becomes unhealthy, a lot of the connections could be affected. Moreover, kube-apiserver's health check service uses different connection from its API resources, which means the health check might pass when the issue affects the performance of the kube-apiserver as a whole.

1. Let's assume that the connection for the health check service is established with the unhealthy Etcd instance. And that:

   - Etcd cluster has properly configured[1] readiness probe.

     If the Etcd cluster the kube-apiserver connects to (via ClusterIP) is implemented with a well designed readiness probe. Then the unhealthy instance will be automatically removed from the Endpoint. So when the kube-apiserver's health check failed (health check timeout is two seconds by default), the container will be restarted. After restarting the container, TCP connections are re-established. Since no connection will be created between the kube-apiserver and the unhealthy Etcd instance anymore, the performance of the kube-apiserver is no longer affected.

   - Etcd cluster does not have properly configured readiness probe.

     If the readiness probe is not properly configured, the unhealthy instance may still end up in the Endpoint's addresses (or periodically removed from and added back to the Endpoint). The restarted Pod could still connect to the unhealthy Etcd instance.

2. Let's assume that the connection for the health check service is established with a healthy Etcd instance. And that:
  
   - Etcd cluster has properly configured readiness probe.

     If the Etcd cluster the kube-apiserver connects to (via ClusterIP) is implemented with a well designed readiness probe. Then the unhealthy instance will be automatically removed from the Endpoint. Kube-Proxy then removes the related rules from the iptables. Since the kube-apiserver's health check succeeds, the container itself will not be restarted, so the current TCP connections to the unhealthy instance are not closed forcefully. Instead, these TCP connections need to time out first prior to be re-established. And as mentioned above, the default TCP connection heartbeat timeout is ten seconds. Moreover, the performance of the kube-apiserver can only recover after the connections are re-established with healthy instances.

   - Etcd cluster does not have properly configured readiness probe.

     If the readiness probe is not properly configured, the unhealthy instance may still end up in the Endpoint's addresses (or periodically removed from and added back to the Endpoint). The kept-alive TCP connections to the unhealthy instance may continue to exist and cause performance issues to the kube-apiserver. Even if some of them time out, they may still be re-established with the unhealthy instance.

*[1] Properly configured here indicates that a readiness probe is able to quickly identify when an instance is available. And that its `successThreshold`, `failureThreshold`, etc., are properly configured so that the instance is not occasionally marked as ready because the probe succeeds from time to time during an outage (like what happened during out tests with disk IO latency).*

### Conclusions

A well designed cloud native application is able to gracefully handle some outages and fail over when necessary.

In addition, by correctly using the liveness probe and readiness probe that K8s provides, application developers can automatically trigger failovers and remove the unhealthy instances from the service.

That being said, liveness probe and readiness probe still have their limitations.

- Readiness probe: assuming there is a multi-instance service with an unhealthy instance, when the outage occurs and readiness probe occasionally succeeds for the unhealthy instance, the problem could be tricky to mitigate because a proper `successThreshold` is needed to keep the unhealthy instance down (marked as "not ready"). In addition, readiness probe (as well as liveness probe) does not take unhealthy endpoints off from headless services.

And if we see from another perspective, application developers should also review their products that whether they are capable of an automatic failover and whether they have leveraged these features K8s provided.
