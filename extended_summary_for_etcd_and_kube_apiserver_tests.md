# Extended Test Summary for Etcd and Kube-APIServer Tests

## Table of Content

- [Extended Test Summary for Etcd and Kube-APIServer Tests](#extended-test-summary-for-etcd-and-kube-apiserver-tests)
  - [Table of Content](#table-of-content)
  - [Test Setup](#test-setup)
  - [Test Results](#test-results)
    - [Our Test Results](#our-test-results)
    - [Our Enhancement to Etcd's Readiness Probe](#our-enhancement-to-etcds-readiness-probe)
  - [Test Summary](#test-summary)
    - [Etcd Failover](#etcd-failover)
    - [Kube-APIServer Failover](#kube-apiserver-failover)
    - [Conclusions](#conclusions)

## Test Setup

We used [Etcd Druid](https://github.com/gardener/etcd-druid) to bootstrap an Etcd cluster with 3 instances. A kube-apiserver was also deployed and connected to the Etcd cluster.

We configured readiness probe and liveness probe for the kube-apiserver. Readiness probe and liveness probe for the Etcd instances were by default configured by Etcd Druid.

## Test Results

### Our Test Results

In our tests with above setup, we observed performance downgrade when disk IO latency or network latency was injected. When the latency reached and went over one second, Etcd failovers were more likely to happen. When the latency was between one second and two seconds, we could see the performance of kube-apiserver went back to normal after Etcd cluster failed over.

However, with default Etcd cluster configuration, when latency reached two seconds, though we could see that Etcd cluster had successfully failed over, the kube-apiserver was still unavailable.

We looked closer at the problem and found that:

- Etcd server's health check need to check the current leader of its cluster. And the health check has by default one second timeout. So when network latency happens and reaches one second, the health check is likely to fail.
- Kubelet is constantly running readiness probes against the unhealthy Etcd Pod.

  - When latency was below two seconds and over one second, the probe command `curl http://<etcd-name>-local:2379/health` always returned `{ "health" : "false" }` with non-zero exit code. The check also took around one second to complete.
  - However, when latancy reached two seconds, the command still returned `{ "health" : "false" }` but with exit code zero. Moreover, the check then was finished instantly.
  
There could be a circut breaker for Etcd server's health check, so that when latency was too much the result was returned immediately. This itself is not a problem, however, since the Etcd cluster bootstrapped by Etcd Druid is using the command execution probe, it only checks the exit code of that specific command. In this case, the exit code was always zero when latency was high.

This has caused the readiness probe to always succeed when latency was high, thus causing the unhealthy instance's addresses to stay in the Etcd Endpoint. Because of this, kube-proxy still maintains the unhealthy destination in the iptables, so when applications use ClusterIP to connect to the Etcd cluster, their connections could still be established with the unhealthy instance and their performance are affected.

### Our Enhancement to Etcd's Readiness Probe

We manually modified the readiness probe of the Etcd container. Instead of directly checking the exit code of `curl http://<etcd-name>-local:2379/health`, we did something like:

```bash
bash -c 'check=$(curl -s http://<etcd-name>-local:2379/health) | grep "true" && if [[ "$check" == "" ]]; then return 1; else return 0; fi'
```

This successfully solved the issue and the readiness probe stayed failed when latency reached two seconds. Therefore the unhealthy Etcd instance's addresses were successfully removed from the Endpoint, and restarted kube-apiserver were no longer suffering from the performance issue.

## Test Summary

### Etcd Failover

For Etcd cluster, when partial outage happens in the zone where the Etcd leader is, the performance of the whole service will be affected. When partial outage happens in the zone where the Etcd follower is, only the user (e.g., kube-apiserver) that connects to this unhealthy follower will be affected (the write operations to the Etcd follower will be forwarded to the Etcd leader). With default configurations, disk IO latency as well as network latency greater than one second will cause Etcd cluster to fail over.

In general there are two ways to connect to an Etcd cluster. The impact on the performance of the applications that use Etcd is different for each method.

- Connect via ClusterIP
  
  In default mode, kube-proxy maintains the iptable on the node. All the Etcd instance destinations are maintained in the iptable chain. So when an application tries to establish a new TCP connection to the Etcd cluster, one of the instance will be chosen (randomly or in a round-robin way), including the unhealthy one. For those that connect to the unhealthy instance, the performance downgrade will persist until they switch to connect to another instance.

  However, if the readiness probe of the Etcd pod is well designed, kubelet can detect when a Etcd instance is unhealthy, and marks the Pod as `NotReady`, thus removing the unhealthy instance addresses from the Etcd Endpoint.

- Connect via Pod fully qualified domain name endpoints

  Application that connects to Etcd cluster via Pod's FQDN endpoints will establish a TCP connection for each endpoint. And depending on which load balancer it is using, the impact would be slightly different. For example, if there are three Etcd instances one of whom is unhealthy, and the application uses round-robin load balancer, one third of the requests will be sent to the unhealthy instance

### Kube-APIServer Failover

The connection established between kube-apiserver and Etcd is kept-alive. By default, every thirty seconds a heartbeat will be sent. Timeout for the heartbeat is by default ten seconds. If the heartbeat server does not reply in ten seconds, the TCP connection will be dropped and a new one will be established.

Usually, we will configure liveness probe and readiness probe for the kube-apiserver. By default, if the kube-apiserver could not get a response from the Etcd cluster in two seconds, a health check would fail. After several consequential unsuccessful health check, the kube-apiserver Pod will be restarted.

Kube-APIServer establishes one TCP connection to Etcd cluster per API resource (which means there might be over a hundred connections). When an Etcd instance becomes unhealthy, a lot of the connections could be affected. Moreover, kube-apiserver's health check service uses different connection from its API resources, which means the health check might pass when the issue affects the performance of the kube-apiserver as a whole.

Let's assume that the connection for the health check service is established with the unhealthy Etcd instance. And that:

- Etcd cluster has well designed readiness probe.

  If the Etcd cluster the kube-apiserver connects to (via ClusterIP) is implemented with a well designed readiness probe. Then the unhealthy instance will be automatically removed from the Endpoint. So when connections are re-established, no connection will be created between the kube-apiserver and the unhealthy Etcd instance anymore. Therefore, the performance of the kube-apiserver is no longer affected.

- Etcd cluster does not have good readiness check.

  If the readiness probe is not well written, the unhealthy instance may still end up in the Endpoint's addresses. The restarted Pod could still connect to the unhealthy Etcd instance. 

### Conclusions

A well designed cloud native application is able to gracefully handle outages and fail over when necessary. In addition, by correctly using the liveness probe and readiness probe that K8s provides, application developers can automatically trigger failovers and remove the unhealthy instances from the service.

And if we see from another aspect, application developers should also review their products that whether they are capable of an automatic failover and whether they have leveraged these features K8s provided.
