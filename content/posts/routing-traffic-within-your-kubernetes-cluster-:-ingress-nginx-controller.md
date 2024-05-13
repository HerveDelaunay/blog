+++
title = 'Routing Traffic Within Your Kubernetes Cluster : Ingress-Nginx Controller'
date = 2024-05-13T21:46:33+02:00
draft = false
tags = ['kubernetes', 'ingress']
+++
I recently had to setup an Ingress in one of the Kubernetes cluster of my job, so I wrote a short explanation for my colleagues that are not aware of Kubernetes Ingresses and since it provide usefull explanations on it I figured I'll also share it here :

## Classic Ingress in Kubernetes

An Ingress in Kubernetes is an object that makes microservices available outside the cluster. It routes incoming requests to the appropriate services, much like a traditional load balancer.

When discussing Ingress, it generally involves two main elements:

- **Ingress Resource**: These are the routing rules for incoming traffic. These rules determine how traffic is directed based on the request path, akin to a conventional load balancer.
- **Ingress Controller**: This controller exists within the cluster as a pod. It reads the rules from the Ingress resource and directs traffic accordingly.

By default, the Ingress controller is only accessible within the cluster. Typically, it is exposed externally using a NodePort service. However, this is not the approach we've chosen.

## On-Premises architecture considerations

In cloud architectures the Ingress controller is automatically exposed through an external load balancer provisioned automatically by the cloud provider. In an on-premises architecture, we manage the external exposure of the Ingress controller ourselves (and this is the example we will study here).
### Ingress-nginx-controller - Traffic Routing

We employ an [existing Ingress deployment](https://kubernetes.github.io/ingress-nginx/) , which exposes an nginx pod acting as reverse proxy within our cluster. Based on the Ingress resources, it routes traffic to the appropriate application pods. 

To install it :

```bash
# With helm
helm upgrade --install ingress-nginx ingress-nginx \ --repo https://kubernetes.github.io/ingress-nginx \ --namespace ingress-nginx --create-namespace

# With kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

The controller pod and the services should now be running : 

```bash
herve@master-node:~$ k get all -n ingress-nginx
NAME                                           READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-c8f499cfc-mmr7j   1/1     Running   0          11d

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.100.77.86    <pending>     80:32729/TCP,443:32366/TCP   11d
service/ingress-nginx-controller-admission   ClusterIP      10.101.134.15   <none>        443/TCP                      11d

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           11d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-c8f499cfc   1         1         1       11d

```

This deployment includes three notable elements:

- **nginx-ingress-controller pod**: Reads the Ingress resource traffic rules and redirects traffic to ClusterIp services representing the correct pods.
- **nginx-ingress-controller service**: This Load Balancer type service exposes our ingress-controller pod outside the cluster. It accepts incoming requests and redirects them to the nginx-controller pod. If the deployment has multiple nginx-controller replicas, the service performs load balancing among these pods.
- **nginx-controller-admission service**: Ensures compliance of the Ingress rules before they are communicated to ETCD. Understanding this service is not crucial for basic operations.

Thus, our Ingress routes traffic appropriately within our cluster, yet we still lack a node-level load balancing solution.
### HA-Proxy - Load Balancing

For load balancing among cluster nodes, two solutions are available to us:

- HA-proxy
- Metal-lb

We opted for HA-proxy. This service will run on a node with external exposure to the cluster, capturing client requests from outside the cluster and redistributing them across the Kubernetes nodes according to its load distribution algorithm, utilizing round-robin (equal distribution across all nodes).
### Keepalived for Resilience

To enhance resilience we employ a dual HA-Proxy setup, where both machines are represented by a single virtual IP (VIP) through Keepalived. If the primary HA-Proxy server fails, the secondary takes over seamlessly, and the IP remains consistent throughout. This switch is transparent to the firewall, which consistently routes traffic to the same IP.

After configuring the service, we should see two ip on the network interface of the main haproxy server :

```bash
herve@hapxy1:~$ ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:40:vb:05:x9 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.103.11/24 metric 100 brd 192.168.103.255 scope global dynamic ens160
       valid_lft 675798sec preferred_lft 675798sec
    inet 192.168.103.10/24 brd 192.168.103.255 scope global secondary ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feba:5d0/64 scope link
       valid_lft forever preferred_lft forever
```

And only one in the backup haproxy network interface :

```bash
nuageherve@hapxy2:~$ ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:80:33:vb:c0:tu brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.103.12/24 metric 100 brd 192.168.103.255 scope global dynamic ens160
       valid_lft 522201sec preferred_lft 522201sec
    inet6 fe80::250:56ff:feba:c0cb/64 scope link
       valid_lft forever preferred_lft forever
```
