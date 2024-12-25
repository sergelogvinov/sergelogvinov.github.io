# Kubernetes on Hybrid Cloud: Service traffic topology and routing

Kubernetes is a powerful tool for managing container workloads in data centers and cloud systems. However, using Kubernetes in a hybrid cloud (both on-premises and cloud) has challenges, especially with networking and traffic routing. By default, Kubernetes services use a `round-robin` algorithm to share traffic between nodes. But this method is not always the best for hybrid clouds because nodes are spread across different locations.

In this article, we will explain why traffic routing is important in a hybrid Kubernetes environment and share best practices for improving network traffic performance.

## Why Traffic Topology and Routing Matter

In a hybrid Kubernetes environment, workloads run both in on-premises data centers and cloud providers. This setup can cause poor network traffic routing, leading to higher latency and lower performance. If your cloud costs depend on data transfer, this can also make your bills higher.

By optimizing traffic routing, you can make sure that network traffic moves efficiently between nodes. Sometimes, traffic might not even need to leave the physical server, saving time and costs. This helps to reduce latency, improve performance, and lower your monthly expenses.

## Internal Traffic Policy

Kubernetes has an internal traffic policy that lets you control how traffic moves inside the cluster. By default, Kubernetes services use the `Cluster` traffic policy. This policy sends traffic to any available endpoint within the cluster, no matter where it is located.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
    type: ClusterIP
    selector:
        app: MyApp
    ports:
        - protocol: TCP
        port: 80
        targetPort: 8080
    internalTrafficPolicy: Local
```

The `internalTrafficPolicy: Local` setting makes sure that traffic only goes to endpoints on the same node. This is helpful for workloads needing low latency and high speed. However, it can also lead to dropped requests if the pods are not running on the same node.

Production use cases:
* `Ingress controllers` (e.g., Traefik or Nginx) often use this policy to handle traffic from an external load balancer. The load balancer checks the Ingress controller pod and sends traffic to a healthy one.
* `DaemonSets` which run on every node in the cluster, can also use this policy to communicate with other pods on the same node. A good example is `CoreDNS`. Since all pods need to resolve DNS names, it makes sense to run CoreDNS on every node.

[Official documentation](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/).

## Topology Aware Routing

Kubernetes also supports topology-aware routing, which helps optimize network traffic based on the physical location of nodes. This feature lets you set rules for services to make sure traffic stays within the same region or availability zone whenever possible. This improves performance, reduces latency, and can help lower costs for cross-zone or cross-region traffic.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
    type: ClusterIP
    selector:
        app: MyApp
    ports:
        - protocol: TCP
        port: 80
        targetPort: 8080
```

In `Auto` mode, topology-aware routing will automatically send traffic to the nearest endpoint in the current zone. This works only if each zone has at least one endpoint. If one or more zones have no endpoints, traffic will follow the default `round-robin` routing instead.

So, the number of endpoints should be equal to or greater than the number of zones to ensure proper routing. I do `not recommend` using this method for hybrid clouds. In many cases, you might not have endpoints in every zone. This can cause traffic to fall back to round-robin routing, which may not be optimal for performance or cost in a hybrid setup.

[Official documentation](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)

## Traffic distribution

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
    type: ClusterIP
    selector:
        app: MyApp
    ports:
        - protocol: TCP
        port: 80
        targetPort: 8080
    trafficDistribution: PreferClose
```

The `trafficDistribution: PreferClose` setting helps to send traffic to the closest endpoint in the zone. If there is no endpoint in the current zone, traffic will be sent to other zones using round-robin routing. This method does not require an endpoint in every zone, which makes it more suitable for hybrid cloud environments.

Production use cases:
* `Ingress controllers` To serve traffic to the closest endpoint in the same zone for better performance.
* `Cache services` (e.g., Redis or Memcached) to ensure faster access by reaching the nearest endpoint.
* `Pod-to-service` communication in microservices architecture, traffic between pods and services can use nearby endpoints to reduce latency.

[Official documentation](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution)

## Conclusion

In a hybrid Kubernetes environment, traffic routing is crucial for optimizing network performance and reducing costs. By using `internal traffic policies` and `traffic distribution` settings, you can make sure that traffic moves efficiently between nodes. This helps to reduce latency, improve performance, and lower your monthly expenses.
